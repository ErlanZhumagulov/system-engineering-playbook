# Интерфейсы (ICD-lite) — LMS+P

---

Документ описывает контракты взаимодействия между компонентами LMS+P и с внешними системами. Для каждого контракта указаны: endpoint / событие, поля, коды ошибок, версионирование, идемпотентность и retry-поведение.

## Интерактивная API-документация (Swagger UI)

<iframe src="swagger/swagger.html" width="100%" height="1500px" style="border:0;" allowfullscreen="allowfullscreen"></iframe>

---

---

## Контракт №1: Запуск экзаменационной сессии

| Параметр | Значение |
|---|---|
| **Endpoint** | `POST /api/v1/exams/{exam_id}/sessions` |
| **Версия** | v1 |
| **Протокол** | HTTPS, синхронный |
| **Аутентификация** | Bearer JWT (из SSO) |
| **Идемпотентность** | Не требуется (409 при дублирующем вызове) |
| **Retry** | Клиент может повторить при 503; 409 означает «уже создано» |

**Заголовки запроса:**

| Заголовок | Тип | Обязательность | Описание |
|---|---|---|---|
| `Authorization` | string | ✅ Да | `Bearer <JWT>` |
| `Content-Type` | string | ✅ Да | `application/json` |

**Тело запроса:**

| Поле | Тип | Обязательность | Описание |
|---|---|---|---|
| `proctoring_mode` | enum | ✅ Да | `none` / `async` / `realtime` |
| `device_info` | object | ❌ Нет | `{"browser": "Chrome/120", "os": "Windows 11"}` |

**Пример запроса:**
```json
{
  "proctoring_mode": "realtime",
  "device_info": {"browser": "Chrome/120", "os": "macOS 14"}
}
```

**Ответы:**

| HTTP код | Ситуация | Тело ответа |
|---|---|---|
| 201 Created | Сессия создана | `{"session_id": "uuid", "webrtc_offer_url": "wss://...", "start_time": "ISO8601", "duration_minutes": 90}` |
| 400 Bad Request | Неверный режим прокторинга | `{"error": "invalid_proctoring_mode", "code": 400, "details": "allowed: none, async, realtime"}` |
| 401 Unauthorized | Токен истёк или отсутствует | `{"error": "unauthorized", "code": 401}` |
| 403 Forbidden | Студент не зачислен на курс | `{"error": "not_enrolled", "code": 403}` |
| 409 Conflict | Активная сессия уже существует | `{"error": "session_already_active", "session_id": "uuid", "code": 409}` |
| 423 Locked | Экзамен вне временного окна | `{"error": "exam_not_available", "available_from": "ISO8601", "code": 423}` |
| 503 Service Unavailable | Медиасервер недоступен | `{"error": "media_server_unavailable", "suggestion": "async_mode", "code": 503}` |

**Версионирование:** При breaking changes — `/api/v2/exams/{exam_id}/sessions`. Старая версия поддерживается 6 месяцев с предупреждением в заголовке ответа `Deprecation: true`.

---

## Контракт №2: Сохранение ответа студента

| Параметр | Значение |
|---|---|
| **Endpoint** | `POST /api/v1/sessions/{session_id}/answers` |
| **Версия** | v1 |
| **Протокол** | HTTPS, синхронный |
| **Аутентификация** | Bearer JWT |
| **Идемпотентность** | ✅ Заголовок `Idempotency-Key` обязателен. Повторный запрос с тем же ключом возвращает 409 с исходным `answer_id` |
| **Retry** | Клиент повторяет при 5xx или timeout, используя тот же `Idempotency-Key` |

**Заголовки запроса:**

| Заголовок | Тип | Обязательность | Описание |
|---|---|---|---|
| `Authorization` | string | ✅ Да | `Bearer <JWT>` |
| `Idempotency-Key` | string(36) | ✅ Да | UUID, генерируется клиентом. Хранится в Redis 24ч |
| `Content-Type` | string | ✅ Да | `application/json` |

**Тело запроса:**

| Поле | Тип | Обязательность | Описание |
|---|---|---|---|
| `question_id` | string(36) | ✅ Да | UUID вопроса |
| `answer` | object | ✅ Да | Зависит от типа вопроса |
| `answer.selected_ids` | array[string] | Для выбора | UUID выбранных вариантов |
| `answer.text` | string | Для текста | Текстовый ответ (макс. 10 000 символов) |
| `answer.pairs` | array[object] | Для сопоставления | `[{"left_id": "uuid", "right_id": "uuid"}]` |
| `client_timestamp` | int64 | ✅ Да | Unix timestamp на устройстве студента (мс) |

**Пример запроса (одиночный выбор):**
```json
{
  "question_id": "550e8400-e29b-41d4-a716-446655440000",
  "answer": {"selected_ids": ["b1a2c3d4-..."]},
  "client_timestamp": 1716451200000
}
```

**Ответы:**

| HTTP код | Ситуация | Тело ответа |
|---|---|---|
| 200 OK | Ответ сохранён | `{"answer_id": "uuid", "saved_at": "ISO8601", "question_id": "uuid"}` |
| 400 Bad Request | Неверный формат ответа | `{"error": "invalid_answer_format", "code": 400, "details": "text answer exceeds 10000 chars"}` |
| 401 Unauthorized | Токен истёк | `{"error": "unauthorized", "code": 401}` |
| 403 Forbidden | Сессия не принадлежит студенту | `{"error": "forbidden", "code": 403}` |
| 409 Conflict | Idempotency-Key уже использован | `{"answer_id": "uuid", "status": "existing", "code": 409}` |
| 410 Gone | Сессия завершена (время вышло) | `{"error": "session_expired", "completed_at": "ISO8601", "code": 410}` |
| 422 Unprocessable | question_id не принадлежит данной сессии | `{"error": "question_not_in_session", "code": 422}` |

---

## Контракт №3: Событие тревоги от Proctoring Service (асинхронное)

| Параметр | Значение |
|---|---|
| **Event** | `proctoring.alert.detected` |
| **Протокол** | RabbitMQ, fanout exchange `proctoring_events` |
| **Версия** | v1 |
| **Потребители** | Notification Service (push проктору), Proctoring DB Writer (сохранение в PostgreSQL) |
| **Идемпотентность** | Потребители используют `event_id` для дедупликации |
| **Retry / DLQ** | При ошибке обработки — до 3 повторных попыток, затем Dead Letter Queue; алерт администратору |

**Поля события:**

| Поле | Тип | Обязательность | Описание |
|---|---|---|---|
| `event_id` | string(36) | ✅ Да | UUID события (для дедупликации) |
| `event_version` | string | ✅ Да | `"v1"` |
| `session_id` | string(36) | ✅ Да | UUID экзаменационной сессии |
| `student_id` | string(36) | ✅ Да | UUID студента |
| `exam_id` | string(36) | ✅ Да | UUID экзамена |
| `event_type` | enum | ✅ Да | `face_absent` / `multiple_faces` / `tab_switch` / `foreign_audio` / `screen_change` |
| `timestamp` | int64 | ✅ Да | Unix timestamp события (мс) |
| `confidence_score` | float | ✅ Да | 0.0 – 1.0 |
| `frame_url` | string | ❌ Нет | Pre-signed URL скриншота (для face-событий), TTL 1 ч |
| `duration_ms` | int | ❌ Нет | Длительность события в мс (`tab_switch`, `face_absent`) |

**Пример события:**
```json
{
  "event_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "event_version": "v1",
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "student_id": "3f2504e0-4f89-11d3-9a0c-0305e82c3301",
  "exam_id": "6ba7b810-9dad-11d1-80b4-00c04fd430c8",
  "event_type": "tab_switch",
  "timestamp": 1716451320000,
  "confidence_score": 0.95,
  "duration_ms": 4200
}
```

**Поведение при ошибках:**

| Ситуация | Поведение |
|---|---|
| Ошибка сохранения в PostgreSQL | Сообщение → DLQ, алерт администратору |
| Notification Service недоступен | Логирует ошибку, не повторяет (push некритичен для хода экзамена) |
| Неизвестный `event_type` | Сообщение отклоняется, пишется в error-лог с `event_id` |

---

## Контракт №4: Передача оценок в SIS

| Параметр | Значение |
|---|---|
| **Endpoint** | `POST /api/v1/grades` (SIS API) |
| **Версия** | v1 (адаптируется к версии API конкретного SIS через адаптер) |
| **Протокол** | HTTPS, синхронный |
| **Аутентификация** | API-ключ (заголовок `X-LMS-API-Key`) |
| **Идемпотентность** | ✅ SIS должен принимать повторный POST с теми же `student_id`+`course_id` без создания дубля (409 = уже записано) |
| **Retry** | SIS Sync Service: 1, 5, 15, 60 мин; после 5 попыток — алерт администратору |

**Тело запроса (LMS+P → SIS):**

| Поле | Тип | Обязательность | Описание |
|---|---|---|---|
| `student_id` | string(36) | ✅ Да | UUID студента в SIS |
| `course_id` | string(36) | ✅ Да | UUID курса в SIS |
| `grade` | decimal(5,2) | ✅ Да | Итоговая оценка |
| `grade_scale` | string | ✅ Да | `100` / `5` / `pass_fail` |
| `grade_date` | date | ✅ Да | Дата выставления (YYYY-MM-DD) |
| `status` | enum | ✅ Да | `passed` / `failed` / `absent` |
| `lms_submission_id` | string(36) | ✅ Да | UUID передачи из LMS+P (для идемпотентности) |

**Ответы SIS:**

| HTTP код | Ситуация | Поведение LMS+P |
|---|---|---|
| 200 OK | Оценка принята | Статус `transferred`, запись в аудит-лог, уведомление студенту |
| 404 Not Found | `student_id` или `course_id` не найден | Статус `error`, уведомление администратору с деталями |
| 409 Conflict | Оценка уже существует | Статус `skipped_duplicate`, запись в лог |
| 422 Unprocessable | Нарушение бизнес-правил SIS | Статус `error`, детали ошибки — администратору |
| 500 / 503 | SIS недоступен | Retry: 1/5/15/60 мин; после 5 попыток — алерт |

---

## Контракт №5: Вердикт проктора (асинхронное событие)

| Параметр | Значение |
|---|---|
| **Event** | `proctoring.verdict.submitted` |
| **Протокол** | RabbitMQ, direct exchange `proctoring_verdicts` |
| **Версия** | v1 |
| **Потребители** | Assessment Service (обновление статуса результата), Notification Service (уведомление преподавателю) |
| **Идемпотентность** | ✅ Assessment Service использует `session_id` для предотвращения двойного применения вердикта |
| **Retry** | При ошибке Assessment Service — до 3 повторных попыток с задержкой 30 сек, затем DLQ |

**Поля события:**

| Поле | Тип | Обязательность | Описание |
|---|---|---|---|
| `event_id` | string(36) | ✅ Да | UUID события |
| `event_version` | string | ✅ Да | `"v1"` |
| `session_id` | string(36) | ✅ Да | UUID экзаменационной сессии |
| `proctor_id` | string(36) | ✅ Да | UUID проктора |
| `overall_verdict` | enum | ✅ Да | `approved` / `rejected` / `escalated` |
| `violation_events` | array[object] | ❌ Нет | Список подтверждённых нарушений: `[{"event_id": "uuid", "event_type": "...", "timestamp": int}]` |
| `comment` | string | ❌ Нет | Комментарий проктора (макс. 2000 символов) |
| `verdict_timestamp` | int64 | ✅ Да | Unix timestamp вердикта (мс) |

**Поведение потребителей:**

| Потребитель | Действие при `approved` | Действие при `rejected` | Действие при `escalated` |
|---|---|---|---|
| Assessment Service | Итоговый результат доступен студенту | Результат аннулирован, статус `invalidated` | Результат заморожен, ждёт решения администратора |
| Notification Service | Email преподавателю: «Вердикт вынесен: допущен» | Email преподавателю: «Нарушение подтверждено» | Email администратору: «Требует эскалации» |

---

## Контракт №6: Получение pre-signed URL для видеозаписи

| Параметр | Значение |
|---|---|
| **Endpoint** | `GET /api/v1/sessions/{session_id}/recording` |
| **Версия** | v1 |
| **Протокол** | HTTPS, синхронный |
| **Аутентификация** | Bearer JWT (роль: Проктор или Администратор) |
| **Идемпотентность** | Каждый вызов генерирует новый pre-signed URL с новым TTL |
| **Retry** | Клиент может повторить при 503 |

**Ответы:**

| HTTP код | Ситуация | Тело ответа |
|---|---|---|
| 200 OK | URL сгенерирован | `{"url": "https://s3.../...", "expires_at": "ISO8601", "ttl_seconds": 14400}` |
| 403 Forbidden | Недостаточно прав (роль Студент) | `{"error": "forbidden", "code": 403}` |
| 404 Not Found | Запись не найдена в S3 | `{"error": "recording_not_found", "code": 404}` |
| 503 Service Unavailable | S3 недоступен | `{"error": "storage_unavailable", "code": 503}` |

**Аудит:** Каждый вызов фиксируется в `access_audit_log`: `user_id`, `role`, `session_id`, `action=view`, `access_timestamp`.

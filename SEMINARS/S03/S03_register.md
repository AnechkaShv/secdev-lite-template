# S03 - Шаблон реестра NFR (Given-When-Then)


## Поля реестра (data dictionary)

* **ID** - короткий идентификатор, например `NFR-001`.
* **User Story / Feature** - к какой истории/фиче относится требование (ссылка допустима).
* **Category** - выберите из банка (напр.: `Performance`, `Security-AuthZ/RBAC`, `RateLimiting`, `Privacy/PII`, `Observability/Logging`, …).
* **Requirement (NFR)** - *измеримое* требование (числа/пороги/границы действия).
* **Rationale / Risk** - зачем это нужно, какой риск/ценность покрываем.
* **Acceptance (G-W-T)** - проверяемая формулировка: *Given … When … Then …*.
* **Evidence (test/log/scan/policy)** - чем подтвердим выполнение (тест, лог-шаблон, сканер, политика).
* **Trace (issue/link)** - ссылка на задачу, обсуждение, артефакт.
* **Owner** - ответственный.
* **Status** - `Draft` | `Proposed` | `Approved` | `Implemented` | `Verified`.
* **Priority** - `P1 - High` | `P2 - Medium` | `P3 - Low`.
* **Severity** - `S1 - Critical` | `S2 - Major` | `S3 - Minor`.
* **Tags** - произвольные метки (через запятую).

---

## User Stories


### US-004 - Редактирование профиля

* Роль-Цель-Ценность: Как пользователь, я хочу редактировать профиль, чтобы поддерживать актуальные данные.
* Кратко: Изменение имени, контактов; маскирование PII в логах.
* API/Endpoints: GET/PUT /api/profile
* NFR hooks: Privacy/PII, Data-Integrity, Security-AuthZ/RBAC, Observability/Logging

### US-011 - Роли и права (RBAC, tenant isolation)

* Роль-Цель-Ценность: Как разработчик, я хочу выпускать и отзывать персональные API-токены, чтобы интегрироваться с сервисом.
* Кратко: Список токенов, маскирование, отзыв.
* API/Endpoints: GET/POST/DELETE /api/tokens
* NFR hooks: Security-Secrets, Auditability, RateLimiting, API-Contract/Errors

### US-002 - Вход в систему (Login)

* Роль-Цель-Ценность: Как пользователь, я хочу входить в систему по логину/паролю, чтобы управлять своими данными.
* Кратко: Вход, выдача JWT/сессии, защита от перебора.
* API/Endpoints: POST /api/auth/login
* NFR hooks: Security-AuthN, RateLimiting, Timeouts/Retry, Observability/Logging


## Таблица реестра

| ID      | User Story / Feature      | Category                 | Requirement (NFR)                                                   | Rationale / Risk                     | Acceptance (G-W-T)                                                                                                    | Evidence (test/log/scan/policy)               | Trace (issue/link) | Owner  | Status   | Priority    | Severity   | Tags              |
| ------- | ------------------------- | ------------------------ | ------------------------------------------------------------------- | ------------------------------------ | --------------------------------------------------------------------------------------------------------------------- | --------------------------------------------- | ------------------ | ------ | -------- | ----------- | ---------- | ----------------- |
| NFR-001 | As a user I upload avatar | Security-InputValidation | Reject payloads >1 MiB; MIME из allowlist; **extra** поля запрещены | Защита от DoS/грязных входных данных | **Given** тело 2 MiB и неизвестные поля<br>**When** POST `/api/files/avatar`<br>**Then** 413 с RFC7807 + запрет extra | test: `e2e-upload-limit`; policy: schema/size | #123               | team-a | Proposed | P2 - Medium | S2 - Major | limits,validation |
| NFR-002 | As a client I call /api/* | Performance              | P95 ≤ 300 ms при 50 RPS в течение 5 мин                             | UX/SLO                               | **Given** сервис здоров<br>**When** 50 RPS на `/api/*` 5 минут<br>**Then** P95 ≤ 300 ms; error rate ≤ 1%              | test: `load-50rps`; log: latency quantiles    | #124               | team-a | Draft    | P1 - High   | S2 - Major | perf,slo          |

> Продолжайте ниже добавлять строки до достижения **8-10 NFR** (или больше, если нужно):

| ID | User Story / Feature | Category | Requirement (NFR) | Rationale / Risk | Acceptance (G-W-T) | Evidence (test/log/scan/policy) | Trace (issue/link) | Owner | Status | Priority    | Severity   | Tags |
| -- | -------------------- | -------- | ----------------- | ---------------- | ------------------ | ------------------------------- | ------------------ | ----- | ------ | ----------- | ---------- | ---- |
|  NFR001  |        US-004. As a user I edit profile              |     Rivacy/PII     |          PII не попадает в логи; сырые PII хранятся не дольше 5 дней         |         приватность и комплаенс      |      **Given** DTO с персональными данными<br>**When** происходит логирование<br>**Then** поля PII маскированы/удалены; план удаления/ретенции существует и применяется в срок 5 дней              |               "phone": "+7916*******"                        |         #125           |       | Draft  | P2 - Medium | S2 - Major |      |
|  NFR002  |         US-011. As an administrator I want to set roles             |     Security-AuthZ/RBAC     |         Пользователь <tenant=A> не может читать/менять ресурсы <tenant=B>; роль <role> ограничивает действия          |         изоляция данных, наименьшие привилегии         |          **Given** пользователь с ролью user из tenant A<br>**When** он запрашивает ресурс tenant B<br>**Then** 403 по политике без утечки данных + аудит попытки доступа         |              test: rbac-tenant-isolation; policy: RBAC matrix; log: access_denied events                   |         #126           |       | Draft  | P2 - Medium | S2 - Major |      |
|  NFR003  |          US-002. As a user I want to log in            |     Security-AuthN     |          Все write-эндпойнты требуют валидный токен; истёкший/некорректный → 401         |        базовая линия безопасности.          |       **Given** истекший JWT<br>**When** POST /api/users/profile<br>**Then** 401 с телом в RFC 7807             |         test: auth-token-validation<br>example 401 response:<br>json<br>{<br> "title": "Token validation failed",<br> "status": 401,<br> "detail": "JWT token expired at 1672531200",<br> "instance": "/api/users/profile"<br>}<br>WAF config: rate limiting rules                        |           #127         |       | Draft  | P2 - Medium | S2 - Major |      |
| NFR004 (Хромова Е. И.)  | US-004. As a user I edit profile | Data-Integrity | Все изменения профиля валидируются и записываются атомарно; формат email/phone нормализован (RFC 5322, E.164) | защита от некорректных данных и потери целостности | **Given** запрос с email в нестандартном формате<br> **When** PUT `/api/profile` выполняется <br> **Then** запись нормализуется и сохраняется без ошибок; SQL-инъекции исключены | test: `profile-data-integrity`; policy: ORM parameterization | #128 | team-a      | Draft  | P2 - Medium | S2 - Major | data,integrity,validation |
| NFR005 (Хромова Е. И.) | US-011. As an administrator I want to set roles | Auditability | Каждое изменение роли фиксируется в audit-журнале с actor, target, action, timestamp; | отслеживаемость админ-действий и расследование инцидентов | **Given** администратор изменяет роль пользователя <br> **When** POST `/api/org/users/{id}/role` выполняется <br> **Then** создается audit-запись с actor, target, временем и результатом | test: `audit-role-change`; log sample: `{"actor":"admin1","action":"role_update", "target":"user42","result":"success"}` | #129  | team-a | Draft  | P2 - Medium | S2 - Major | audit,security,rbac |
| NFR006 (Хромова Е. И.) | US-002. As a user I want to log in | RateLimiting | На `/api/auth/login` действует лимит 5 попыток / минута на IP; превышение → 429 + Retry-After | предотвращение перебора паролей (brute force) | **Given** 6 запросов логина за 60 с с одного IP <br> **When** выполняется 6 запрос <br> **Then** ответ 429 с заголовком `Retry-After: 60` | test: `login-rate-limit`; WAF policy: `limit-login-5rpm` | #130 | team-a | Draft  | P1 - High | S1 - Critical | security,ratelimit,authn |
| NFR007 (Хромова Е. И.) | US-002. As a user I want to log in | Security-SessionManagement | Сессии авторизации истекают через 30 мин неактивности; токен refresh действителен ≤24 ч; при logout все токены инвалидируются | предотвращение несанкционированного доступа при утере токена или неактивности | **Given** пользователь бездействует 31 мин <br> **When** пользователь делает запрос с access-токеном **Then** ответ 401; требуется обновление токена; при logout refresh-токен недействителен | test: `session-expiry-check`; log: `token-revoke-events`; policy: `session-timeout-30m` | #131 | team-a      | Draft  | P1 - High | S1 - Critical | security,session,authn |
| NFR008 (Шварева А.А.)   |          US-004. As a user I edit profile            |     Observability/Logging     |          Все запросы логируются в JSON и содержат correlation_id на каждом этапе         |        **Given** запрос с заголовком X-Correlation-ID=<id> <br> **When** он проходит через сервис **Then** во всех логах появляется correlation_id=<id> и ключевые поля запроса/ответа         |        трассировка и разбор инцидентов            |                 паттерн логов; скрин/запрос поиска по id.                |         test: correlation-id-tracing<br>log examples:<br>json<br>`{"timestamp":"2024-01-15T10:30:00Z","level":"INFO","correlation_id":"prof-123-abc","service":"api-gateway","method":"PUT","path":"/api/profile","status":200,"latency_ms":45}<br>{"timestamp":"2024-01-15T10:30:00Z","level":"INFO","correlation_id":"prof-123-abc","service":"profile-service","operation":"updateProfile","user_id":"usr-456","duration_ms":32}<br>{"timestamp":"2024-01-15T10:30:01Z","level":"ERROR","correlation_id":"prof-123-abc","service":"profile-service","error":"Validation failed","stack_trace":"..."}`<br>alerting: error-rate > 1% triggers within 60s           |       | Draft  | P2 - Medium | S2 - Major |      |

---

## Памятка по заполнению

* **Измеримость.** В `Requirement` фиксируйте числа и границы (мс, RPS, минуты, MiB, коды 4xx/5xx, CVSS).
* **Проверяемость.** В `Acceptance (G-W-T)` используйте объективные условия и наблюдаемые факты (код ответа, квантиль, наличие заголовка, запись в лог).
* **Связность.** Сверяйте, чтобы NFR не конфликтовали (timeouts vs retry, rate limits vs SLO, privacy vs logging).
* **План проверки.** В `Evidence` укажите, чем это будет подтверждаться позже (test/log/scan/policy). В рамках семинара **реальные артефакты не требуются**.
* **Трассировка.** В `Trace` добавляйте ссылки на Issues/документы, чтобы потом не искать контекст.

---

## После семинара

* Перенесите/доработайте **8-10 утверждённых NFR** (по сути, те же строки) в раздел **NFR** вашего `GRADING/TM.md`.
* На S04-S05 свяжете эти NFR с угрозами (STRIDE) и ADR - по ID.

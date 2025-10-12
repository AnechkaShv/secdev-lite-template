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
|  NFR001(Шварева А.А.)  |        US-004. As a user I edit profile              |     Rivacy/PII     |          PII не попадает в логи; сырые PII хранятся не дольше 5 дней         |         приватность и комплаенс      |      **Given** DTO с персональными данными<br>**When** происходит логирование<br>**Then** поля PII маскированы/удалены; план удаления/ретенции существует и применяется в срок 5 дней              |               "phone": "+7916*******"                        |         #125           |       | Draft  | P2 - Medium | S2 - Major |      |
|  NFR002(Шварева А.А.)  |         US-011. As an administrator I want to set roles             |     Security-AuthZ/RBAC     |         Пользователь <tenant=A> не может читать/менять ресурсы <tenant=B>; роль <role> ограничивает действия          |         изоляция данных, наименьшие привилегии         |          **Given** пользователь с ролью user из tenant A<br>**When** он запрашивает ресурс tenant B<br>**Then** 403 по политике без утечки данных + аудит попытки доступа         |              test: rbac-tenant-isolation; policy: RBAC matrix; log: access_denied events                   |         #126           |       | Draft  | P2 - Medium | S2 - Major |      |
|  NFR003(Шварева А.А.)  |          US-002. As a user I want to log in            |     Security-AuthN     |          Все write-эндпойнты требуют валидный токен; истёкший/некорректный → 401         |        базовая линия безопасности.          |       **Given** истекший JWT<br>**When** POST /api/users/profile<br>**Then** 401 с телом в RFC 7807             |         test: auth-token-validation<br>example 401 response:<br>json<br>{<br> "title": "Token validation failed",<br> "status": 401,<br> "detail": "JWT token expired at 1672531200",<br> "instance": "/api/users/profile"<br>}<br>WAF config: rate limiting rules                        |           #127         |       | Draft  | P2 - Medium | S2 - Major |      |
| NFR004 (Хромова Е. И.)  | US-001. As a guest I register account | Security-InputValidation | Тело `POST /api/auth/register` ≤1 MiB; поля `email`, `password` обязательны; MIME: `application/json`; неизвестные поля -> 400 | Предотвращение DoS и некорректных входных данных | **Given** тело запроса с неизвестными полями или >1 MiB<br> **When** `POST /api/auth/register` <br> **Then** 400/413 с телом в RFC 7807, ошибка `invalid_request` | test: `input-validation-register`; policy: schema `register_request.json` | #128 | team-a      | Draft  | P2 - Medium | S2 - Major | validation,security |
| NFR005 (Хромова Е. И.) | US-003. As a user I reset password | Security-Secrets | Секреты сброса пароля (token) хранятся ≤15 мин; случайность ≥128 бит; не сохраняются в логах | защита от утечек и повторного использования токенов | **Given** истекший или повторно используемый токен <br> **When** POST `/api/auth/reset` <br> **Then** ответ 401; токен не проходит валидацию и не встречается в логах | test: `reset-token-expiry`; scan: `secrets-leak-check` | #129  | team-a | Draft  | P1 - High | S1 - Critical | secrets,authn |
| NFR006 (Хромова Е. И.) | US-015. As an auditor I view actions history | Auditability | Все админ-операции фиксируются в audit-журнале с actor, target, action, timestamp, result; записи неизменяемы | соответствие политикам, разбор инцидентов | **Given** администратор изменяет роль пользователя <br> **When** операция завершается <br> **Then** создается audit-запись с actor, target, временем и результатом; запись доступна для чтения, но не для редактирования | test: `audit-log-entry`; log sample: `{actor:"admin1",action:"role_change", target:"user42",result:"success"}` | #130 | team-a | Draft  | P2 - Medium | S2 - Major | audit,logging |
| NFR007 (Хромова Е. И.) | US-014. As a user I export my data | Privacy/PII | При экспорте `/api/export` поля PII маскированы; выгрузка ограничена 1000 записями; срок хранения готового файла ≤24 ч | защита персональных данных и утечек через экспорт | **Given** запрос на экспорт данных с PII <br> **When** пользователь скачивает результат `/api/export?format=csv` **Then** PII-поля (email, phone) маскированы; файл удаляется через 24 ч; при превышении лимита 1000 записей возвращается 413 | test: `export-privacy-limit`; policy: data-retention 24h; log: `export_cleanup` | #131 | team-a      | Draft  | P2 - Medium | S2 - Major | privacy,export,limits |
| NFR008(Шварева А.А.) |           US-004. As a user I edit profile            |     Data Integrity / Canonicalization (Data-Integrity)     |        Все SQL-запросы параметризованы; строки/даты/номера приводятся к канонической форме.           |        защита от инъекций, целостность.          |          **Given** входные данные с нестандартными форматами <br> **When** выполняется запись в БД **Then** используется параметризация; значения нормализованы (NFC/E.164/UTC/Decimal)          |           test: sql-injection-protection<br>code example:<br>javascript<br>// Параметризованный запрос<br>await db.query(<br> 'UPDATE profiles SET name = $1 WHERE id = $2',<br> [inputName, userId]<br>);<br>ORM config: validation rules enabled                      |                    |       | Draft  | P2 - Medium | S2 - Major |      |

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

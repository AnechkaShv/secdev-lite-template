# S04 - STRIDE per element (матрица)

Этот файл - рабочая матрица **без оценок**: фиксируем релевантные угрозы для каждого элемента и потока из вашей DFD (mermaid), связываем их с NFR из S03 и накидываем идею mitigation (для будущих ADR на S05).

> Рабочее расположение у студента: `SEMINARS/S04/S04_stride_matrix.md`.
> Оценивание L×I (1-5) и выбор Top-5 делаются **отдельно** в `S04_risk_scoring.md`.
> Семинарские артефакты в `/EVIDENCE/` **не кладём**.

---

## Как работать с матрицей

1. Возьмите элементы вашей DFD: **узлы** (API, Service, DB, External) и **рёбра/потоки** (JWT, PII, files, payments).
2. Для **каждого** элемента пройдите буквы **STRIDE** и зафиксируйте **только релевантные** угрозы (0-2 на букву).
3. Для каждой угрозы:

   * кратко опишите **Description** (1-2 предложения, по делу);
   * укажите **NFR link (ID)** из реестра S03 (или `need NFR`, если такого ещё нет);
   * добавьте **Mitigation idea (ADR later)** - короткое имя решения, которое пойдёт в ADR на S05.

---

## Легенда STRIDE

* **S - Spoofing:** подмена идентичности/токена.
* **T - Tampering:** изменение данных/запросов/конфигурации.
* **R - Repudiation:** отрицание действий (нет аудита/трассировки).
* **I - Information disclosure:** утечка конфиденциальных данных (PII/секреты).
* **D - Denial of service:** отказ в обслуживании (ресурсное истощение/«залипание»).
* **E - Elevation of privilege:** повышение привилегий/обход RBAC/тенант-изоляции.

---

## Примеры строк (образец заполнения)

| Element                      | Data/Boundary | Threat (S/T/R/I/D/E) | Description                                            | NFR link (ID)                   | Mitigation idea (ADR later)               |
| ---------------------------- | ------------- | -------------------- | ------------------------------------------------------ | ------------------------------- | ----------------------------------------- |
| Edge: Internet → API         | JWT / public  | S                    | Повтор/подмена токена, reuse истёкшего/украденного JWT | NFR-AuthN, NFR-RateLimit        | JWT TTL+Refresh, rate limit на `/auth/*`  |
| Node: Service                | Logs          | I                    | PII в логах и сообщениях об ошибках                    | NFR-Privacy/PII, NFR-API-Errors | Маскирование PII, RFC7807 без стэктрейсов |
| Edge: Service → External API | HTTP/gRPC     | D                    | Залипание без timeout/retry/circuit breaker            | NFR-Timeouts/Retry/CB           | Timeout≤2s, retry≤3 с джиттером, CB       |

> После заполнения матрицы **перенесите уникальные/объединённые риски** в `S04_risk_scoring.md` для приоритизации L×I (1-5) и выбора Top-5.

---

## Матрица для заполнения

| Element | Data/Boundary | Threat (S/T/R/I/D/E) | Description | NFR link (ID) | Mitigation idea (ADR later) |
| ------- | ------------- | -------------------- | ----------- | ------------- | --------------------------- |
|  Edge: Internet → API         | JWT / public  | S                    | Повтор/подмена токена, reuse истёкшего/украденного JWT | NFR-AuthN, NFR-RateLimit        | JWT TTL+Refresh, rate limit на `/auth/*`  |               |                             |
| Edge: Internet → API         | JSON   | I | Слабый CORS/утечка через Referrer  | NFR-Privacy/PII, NFR-API-Contract | строгий CORS (origin, методы, заголовки), Referrer-Policy: `no-referrer `                           |
| Node: API Gateway / Auth Controller | JWT / public | S  | Проброс доверия внутренним сервисам без проверки токена сервиса |NFR-AuthN (service-to-service) | {<br> "title": "Token validation failed",<br> "status": 401,<br> "detail": "JWT token expired at 1672531200",<br> "instance": "/api/users/profile"<br>} | 
| Node: API Gateway / Auth Controller | Logs | R  | Логи без user/tenant/route | NFR-Observability/Logging |log examples: <br>json <br>{"timestamp":"2024-01-15T10:30:00Z","level":"INFO","correlation_id":"prof-123-abc","service":"api-gateway","method":"PUT","path":"/api/profile","status":200,"latency_ms":45}<br>{"timestamp":"2024-01-15T10:30:00Z","level":"INFO","correlation_id":"prof-123-abc","service":"profile-service","operation":"updateProfile","user_id":"usr-456","duration_ms":32}<br>{"timestamp":"2024-01-15T10:30:01Z","level":"ERROR","correlation_id":"prof-123-abc","service":"profile-service","error":"Validation failed","stack_trace":"..."}|                             |
| Edge: Service → External API | DTO | I | Передача PII/секретов в query/логи | NFR-Privacy/PII | только в заголовках/теле, redaction логов |
| Edge: Service → External API | JWT / Response | S | Доверие внешнему без проверки (pinning/issuer) | NFR-AuthN | TLS-pinning (где возможно), проверка CA, валидация ответов |
| Node: Service | Logs | E | Обход бизнес-контролей (feature-flags/параметры) | NFR-AuthZ/RBAC | серверные гварды, неизменяемые флаги на сервере |
| Edge: Service → DB        | audit event              |   R                   |Нет отзыв-идентификаторов/idempotency для платёжных/критичных вызовов             | NFR-Audit              | idempotency-key, лог-журнал                            |
| Edge: Service → DB         | SQL               | T                     | Tampering данных/схемы без целостности            | NFR-Data-Integrity              | строгая проверка схем/подписей, idempotency keys                            |
| Node: DB | SQL              | D | Долгие запросы/нет индексов | NFR-DoS/Resilience | индексы, лимиты на LIKE %...%, пагинация |
| Node: DB | SQL | I | PII без маскирования/шифрования/бэкапы открыты | NFR-Privacy/PII | колоночное шифрование/маскирование, контроль доступа к бэкапам |

---

## Самопроверка (быстро)

* [ ] Пройдены **все узлы и рёбра** вашей DFD.
* [ ] У угроз стоят **NFR link (ID)** или пометка `need NFR`.
* [ ] У каждой угрозы есть понятная **Mitigation idea** (будущий ADR).
* [ ] Нет «воды»: кратко, по делу, без оценок L/I (они - в `S04_risk_scoring.md`).
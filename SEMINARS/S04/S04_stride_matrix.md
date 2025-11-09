# S04 - STRIDE per element (матрица)

## Легенда STRIDE

* **S - Spoofing:** подмена идентичности/токена.
* **T - Tampering:** изменение данных/запросов/конфигурации.
* **R - Repudiation:** отрицание действий (нет аудита/трассировки).
* **I - Information disclosure:** утечка конфиденциальных данных (PII/секреты).
* **D - Denial of service:** отказ в обслуживании (ресурсное истощение/«залипание»).
* **E - Elevation of privilege:** повышение привилегий/обход RBAC/тенант-изоляции.

## Матрица

| Element                   | Data/Boundary               | Threat (S/T/R/I/D/E) | Description                                                                                                               | NFR link (ID)                                                                         | Mitigation idea (ADR later)                                                                                              |
|---------------------------|-----------------------------|----------------------|---------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| **Internet → API (edge)** | JWT / public API            | S, I, D              | JWT replay/forgery (stolen tokens); detailed errors leak PII; lack of rate-limiting enables DoS                           | **NFR-SEC-AUTH-002, NFR-SEC-PII-001, NFR-SEC-DOS-001, NFR-SEC-TLS-001**               | Short JWT TTL + refresh; token revocation path; redact errors (RFC7807); global rate-limits / WAF                        |
| **API Gateway (node)**    | Requests / Headers / Logs   | T, R, I, D           | Header injection/tamper; missing correlation_id → poor traceability; PII in logs; unlimited concurrency → DoS             | **NFR-SEC-VALID-001, NFR-SEC-PII-001, NFR-SEC-DOS-001, (need NFR: Logging)**          | Normalize/whitelist headers; inject and propagate correlation_id; structured logs with PII redaction; concurrency quotas |
| **Store Service (node)**  | Business logic              | T, D, E, I           | Client-side tampering (price/cart manipulation); race on stock (oversell); elevation of privilege; PII leakage in backups | **NFR-SEC-VALID-001, NFR-SEC-AUTHZ-001, NFR-SEC-PII-001, (need NFR: Data-Integrity)** | Server-side ownership checks; DB transactions/locking; idempotency; minimize PII in snapshots; backup controls           |
| **DB / Storage (node)**   | Orders / persistent data    | T, I, R, E           | Unauthorized tampering of persistent data; PII at rest not encrypted; missing audit trail for admin ops                   | **NFR-SEC-CONF-001, NFR-SEC-PII-001, (need NFR: Audit)**                              | Least-privilege DB accounts; column-level encryption; enable immutable audit logging; restrict backup access             |
| **External (edge)**       | Payment & Fulfillment calls | S, D, R, T           | Missing idempotency/timeouts → double-charge or hangs; forged external callbacks; insufficient auth; tampered responses   | **NFR-API-IDEMP-001, NFR-CMP-PCI-001, NFR-SEC-INT-001, (need NFR: Timeouts)**         | Idempotency keys; timeouts+retry+CB; validate callbacks (HMAC); strong credentials / mTLS; schema validation             |
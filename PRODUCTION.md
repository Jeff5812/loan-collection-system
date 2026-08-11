# Production Readiness — Loan Collections Router

Honest snapshot of this workflow's current state, as exported. No aspirational claims — a category is only marked "Built" if the workflow itself implements it today.

| Category | Status | Notes |
|---|---|---|
| Architecture | Built | Scheduled trigger, batch loop over accounts, AI risk-scoring agent, switch-based routing to WhatsApp/SMS, legal referral, or human review; separate webhook-driven payment flow. |
| Security | Not addressed | Payment webhooks (`Show Payment Page`, `Confirm Payment`) are unauthenticated — no HMAC signing, bearer token, or IP allowlisting on the inbound webhooks. |
| Secrets | Built | All external calls (Postgres, Gmail, Twilio, WhatsApp, Gemini) use n8n's built-in credential store rather than hardcoded values. No `.env`/secrets-vault layer beyond n8n's own encryption. |
| Database | Built | Postgres nodes handle fetching delinquent accounts, marking accounts contacted, clearing delinquent status, and logging payments/executions. Schema/migrations are not included in this repo. |
| Error handling | Designed | A workflow-level `errorWorkflow` is configured in the n8n settings, so failures route to a separate error-handling workflow — but that workflow isn't part of this export, and no node-level `continueOnFail` / try-catch branching exists inside this workflow. |
| Retry strategy | Not addressed | No node has `retryOnFail` enabled; a single failed HTTP/API call fails the execution. |
| Idempotency | Not addressed | No dedup keys or "already processed" checks — the payment webhook and reminder sends can be re-triggered without protection against duplicate processing. |
| Monitoring | Not addressed | No health checks, uptime monitoring, or alerting beyond the error workflow hookup. |
| Logging | Built | `Log Execution` and `Log Payment` Postgres nodes persist a record of contacts made and payments received. |
| Backups | Not addressed | No backup/retention strategy for the Postgres database is defined in this workflow. |
| Privacy | Not addressed | No data minimization, redaction, or retention policy applied to borrower PII (phone numbers, payment details) as it flows through the workflow/logs. |
| CI/CD | Not addressed | No versioning, staging/prod promotion, or automated deploy pipeline — this is a manual-import n8n workflow. |
| Scalability | Designed | The batch loop (`Loop Accounts`) processes accounts sequentially, which is safe but not parallelized; fine for current volumes, would need batching/concurrency tuning at higher scale. |
| Human fallback | Built | Low-confidence AI risk assessments are explicitly routed to a "Flag for Human Review" step rather than auto-actioned. |

**Summary:** the core collections logic, routing, and payment flow are fully built and functional. The gaps are concentrated in production-hardening: webhook authentication, retries, idempotency, and monitoring are the highest-value next steps before this handles real money at scale.

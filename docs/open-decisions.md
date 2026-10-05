# Open Decisions & Risks

Items that need a decision before the related work can be finished. When one is decided, record it as an ADR and remove it from here.

| # | Decision | Needs input from | Blocks | Notes |
|---|---|---|---|---|
| 1 | **Payment gateway** (Razorpay / Cashfree / PhonePe PG / other) | Business | Payments integration | Compare settlement speed, fees, KYC timeline, webhook reliability and refund APIs. |
| 2 | **Offline authorization policy**: `idTokenInfo.expiryDate` window, cache lifetime, whether unknown tokens are accepted offline | Business + engineering | Auth handlers, boot-time config push | Short windows mean faster revocation and more backend load. Long windows mean better offline resilience and staler revocation. |
| 3 | **`meter_values` retention**: raw data online for how long, then downsample / archive to object storage | Business + compliance | Partition maintenance job | Partitioning is already in place, so either answer is cheap to implement. |
| 4 | **Data residency / compliance for payment data** | Compliance / legal | Cloud region and provider choice | Likely requires India-hosted infrastructure. Decide before provisioning production. |
| 5 | **Firmware rollout and rollback strategy** | Engineering + hardware vendor | Firmware management features | Staged rollout (canary sites first), and what "rollback" means when the vendor only supports forward updates. |
| 6 | **Multi-tenancy / franchisee model** (deferred) | Business | Post-MVP | When built: tenant isolation model, franchisee onboarding, revenue split, driver-app white-labelling. `sites` table already in place (ADR-0007). |
| 7 | **Charger authentication on connect** (OCPP security profile: Basic auth over TLS vs client certificates) | Engineering + hardware vendor | Production connections | Phase 2, but firmware capability should be confirmed early. |
| 8 | **Command reply timeout and offline-command behaviour** (fail immediately vs queue until reconnect) | Engineering | CSMS command API | Proposed: fail fast with 409/504 for interactive commands, queue only revocation pushes. |
| 9 | **CD pipeline and cloud provider** | Engineering | Staging environment | CI exists. Image build, registry and rollout are undefined. |

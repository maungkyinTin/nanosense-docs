# Medical Intelligence API Reference

**Base URL:** `https://api.medintelligent.ai` (production) | `http://localhost:8000` (local)
**Version:** 3.0
**Interactive docs:** `{base_url}/docs` (Swagger UI) | `{base_url}/redoc` (ReDoc)

---

## Authentication

All protected endpoints require one of the following:

### JWT Bearer Token

Include in the `Authorization` header:

```
Authorization: Bearer <access_token>
```

Obtain a token by logging in or registering a tenant. Tokens expire after 30 minutes by default.

### API Key

Include in the `X-API-Key` header:

```
X-API-Key: mrag_<your_key>
```

API keys are scoped to a tenant and never expire until revoked. The plaintext key is shown once at creation — store it securely.

### Roles

| Role | Scope |
|------|-------|
| `admin` | Full tenant management, billing, reports, user management |
| `clinician` | Queries, analytics, FHIR read access |
| `user` | Queries only |
| `readonly` | Read-only access to own tenant data |

---

## Error Responses

All errors return a consistent JSON envelope:

```json
{
  "error": "Human-readable error message",
  "timestamp": "2026-05-01T12:00:00.000000",
  "path": "https://api.medintelligent.ai/query"
}
```

| Status | Meaning |
|--------|---------|
| `400` | Invalid request body or parameters |
| `401` | Missing or expired authentication |
| `403` | Insufficient role / wrong tenant |
| `404` | Resource not found |
| `409` | Conflict (duplicate, already suspended, etc.) |
| `429` | Rate limit exceeded |
| `500` | Internal server error |

---

## Rate Limits

Rate limits are enforced per-client IP using a sliding window.

| Endpoint | Limit |
|----------|-------|
| `POST /auth/login` | 10/min |
| `POST /query` | 30/min |
| `POST /tenants/register-partner` | 5/min |
| `PATCH /tenants/{id}/budget` | 10/min |
| `GET /imaging/studies` | 30/min |
| `GET /imaging/report/{uid}` | 30/min |

When exceeded, the response includes `Retry-After` and `X-RateLimit-Reset` headers.

---

## Plans & Mode Access

| Plan | Queries/mo | Included Tokens/mo | Query Modes | Rate Limit |
|------|--------:|-------------------:|-------------|--------:|
| **Starter** | 1,000 | 100K | fast, rag_cag | 10 RPM |
| **Professional** | 10,000 | 1M | fast, deep, rag_cag | 30 RPM |
| **Enterprise** | 100,000 | 10M | fast, deep, rag_cag, mcp | 100 RPM |
| **Partner Bundled** | 50,000 | Custom | fast, deep, rag_cag | 30 RPM |

All plans also have access to `radiology` and `auto` modes.

**Overage rates** (applied when included token budget is exceeded):

| Plan | Overage Rate |
|------|-------------|
| Starter | $0.002 per 1K tokens |
| Professional | $0.0015 per 1K tokens |
| Enterprise | $0.001 per 1K tokens |
| Partner Bundled | N/A (custom budget) |

---

# Endpoints

## Registration & Login

### Register a Tenant

```
POST /tenants/register
```

Self-service clinic registration. Returns a tenant ID, admin user, API key, JWT, and sandbox provision ID for immediate use.

**Request:**

```json
{
  "name": "Sunrise Clinic",
  "admin_email": "admin@sunrise.com",
  "plan": "starter",
  "admin_username": "dr.admin",
  "admin_password": "S3cureP@ss!"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string (2-200) | yes | Clinic display name |
| `admin_email` | string | yes | Admin email address |
| `plan` | string | yes | `starter`, `professional`, or `enterprise` |
| `admin_username` | string (3-50) | yes | Admin login username |
| `admin_password` | string (8-128) | yes | Admin login password |

**Response:** `201 Created`

```json
{
  "tenant_id": "t-sunrise-clinic-abc123",
  "name": "Sunrise Clinic",
  "plan": "starter",
  "admin_user": {
    "username": "dr.admin",
    "email": "admin@sunrise.com",
    "role": "admin"
  },
  "api_key": {
    "key_id": "k-abc123",
    "api_key": "mrag_live_abc123...",
    "key_prefix": "mrag_live",
    "label": "default"
  },
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "email_verification_required": false,
  "sandbox_provision_id": "provision-abc123",
  "onboarding_steps": ["verify_email", "explore_sandbox", "first_query"]
}
```

> **Important:** The `api_key.api_key` value is shown only once. Store it securely.

---

### Login

```
POST /auth/login
```

Exchange credentials for a JWT access token.

**Request:**

```json
{
  "username": "dr.admin",
  "password": "S3cureP@ss!"
}
```

**Response:** `200 OK`

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "bearer",
  "expires_in": 1800
}
```

---

## Querying

### Submit a Query

```
POST /query
```

Process a medical question using the specified RAG mode. Requires authentication.

**Request:**

```json
{
  "question": "What medications is this patient currently taking?",
  "mode": "fast",
  "patient_id": "8c8e1c9a-b310-43c6-33a7-ad11bad21c40",
  "include_sql": false,
  "include_knowledge_graph": false,
  "session_id": null,
  "rag_cag_mode": "smart_caching"
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `question` | string (1-1000) | *required* | Medical question |
| `mode` | string | `"fast"` | `fast`, `deep`, `rag_cag`, `mcp`, `radiology`, `auto` |
| `patient_id` | string | `null` | Scope query to a specific patient |
| `include_sql` | bool | `false` | Include generated SQL in response |
| `include_knowledge_graph` | bool | `false` | Include knowledge graph data |
| `session_id` | string | `null` | Session ID for conversation tracking |
| `rag_cag_mode` | string | `"smart_caching"` | RAG+CAG sub-mode: `smart_caching`, `direct_data`, `hybrid` |

**Response:** `200 OK`

```json
{
  "query_id": "q-a1b2c3d4",
  "question": "What medications is this patient currently taking?",
  "answer": "Based on the clinical records, the patient is currently prescribed...",
  "mode": "fast",
  "processing_time": 0.42,
  "confidence_score": 0.91,
  "sources_used": ["patients", "medication_requests"],
  "sql_query": null,
  "knowledge_graph": null,
  "metadata": {
    "tenant_id": "t-sunrise-clinic-abc123",
    "tokens_used": 245,
    "cache_hit": false
  }
}
```

### Query Modes

| Mode | Description | Best For |
|------|-------------|----------|
| `fast` | SQL generation + direct data lookup | Quick EHR data retrieval |
| `deep` | Multi-step reasoning with planning | Complex clinical questions |
| `rag_cag` | Retrieval + caching for cost efficiency | Balanced accuracy and cost |
| `mcp` | Multi-source orchestration (EHR + KB + guidelines) | Comprehensive analysis |
| `radiology` | Semantic search over imaging reports | Imaging-focused queries |
| `auto` | Automatic mode selection based on question | General use |

---

## Tenant Management

### Get Tenant Details

```
GET /tenants/{tenant_id}
```

Returns tenant profile. Non-admin users can only view their own tenant.

**Response:** `200 OK`

```json
{
  "id": "t-sunrise-clinic-abc123",
  "name": "Sunrise Clinic",
  "plan": "starter",
  "status": "active",
  "admin_email": "admin@sunrise.com"
}
```

---

### Update Tenant

```
PATCH /tenants/{tenant_id}
```

Update tenant name, email, or plan. Requires admin role.

**Request:**

```json
{
  "name": "Sunrise Medical Group",
  "plan": "professional"
}
```

All fields are optional.

---

### API Key Management

#### Generate a New Key

```
POST /tenants/{tenant_id}/keys
```

**Request:**

```json
{
  "label": "production-backend"
}
```

**Response:** `200 OK`

```json
{
  "key_id": "k-new-key-id",
  "api_key": "mrag_live_xyz789...",
  "key_prefix": "mrag_live",
  "label": "production-backend"
}
```

#### List Keys

```
GET /tenants/{tenant_id}/keys
```

Returns metadata only (no plaintext keys).

```json
[
  {
    "key_id": "k-new-key-id",
    "key_prefix": "mrag_live",
    "label": "production-backend",
    "status": "active"
  }
]
```

#### Revoke a Key

```
DELETE /tenants/{tenant_id}/keys/{key_id}
```

**Response:** `200 OK`

```json
{
  "status": "revoked",
  "key_id": "k-old-key-id"
}
```

---

### User Management

#### Create a User

```
POST /tenants/{tenant_id}/users
```

Requires admin role within the tenant.

**Request:**

```json
{
  "username": "nurse.jones",
  "email": "jones@sunrise.com",
  "password": "SecurePass123!",
  "role": "clinician"
}
```

---

### Sandbox Data

#### Check Sandbox Status

```
GET /tenants/{tenant_id}/sandbox/status
```

Poll the status of sandbox data seeding. Status values: `pending`, `running`, `ready`, `failed`.

#### Trigger Sandbox Provision

```
POST /tenants/{tenant_id}/sandbox/provision
```

Start sandbox data seeding with synthetic patient data.

---

## Billing & Usage

### Get Usage

```
GET /billing/usage?period=2026-05
```

Returns token usage for the current or specified billing period. Requires admin role.

**Response:** `200 OK`

```json
{
  "tenant_id": "t-sunrise-clinic-abc123",
  "billing_period": "2026-05",
  "total_queries": 142,
  "input_tokens": 28400,
  "output_tokens": 19600,
  "total_tokens": 48000,
  "included_tokens": 50000,
  "overage_tokens": 0,
  "estimated_cost_usd": 0.00,
  "plan": "professional"
}
```

---

### Get Usage Breakdown

```
GET /billing/usage/breakdown?period=2026-05
```

Per-mode token breakdown.

**Response:** `200 OK`

```json
{
  "tenant_id": "t-sunrise-clinic-abc123",
  "billing_period": "2026-05",
  "breakdown": [
    { "rag_mode": "fast", "queries": 100, "input_tokens": 15000, "output_tokens": 10000, "total_tokens": 25000 },
    { "rag_mode": "deep", "queries": 42, "input_tokens": 13400, "output_tokens": 9600, "total_tokens": 23000 }
  ]
}
```

---

### Billing Status

```
GET /billing/status
```

Current subscription and cost status.

**Response:** `200 OK`

```json
{
  "tenant_id": "t-sunrise-clinic-abc123",
  "plan": "professional",
  "stripe_configured": true,
  "subscription_status": "active",
  "total_tokens_this_month": 48000,
  "included_tokens": 50000,
  "overage_tokens": 0,
  "estimated_cost_usd": 0.00
}
```

---

### Create Subscription

```
POST /billing/subscribe
```

Create or replace a Stripe subscription. Requires admin role.

**Request:**

```json
{
  "plan": "professional",
  "payment_method_id": "pm_1234567890"
}
```

**Response:** `200 OK`

```json
{
  "tenant_id": "t-sunrise-clinic-abc123",
  "stripe_customer_id": "cus_abc123",
  "subscription_id": "sub_xyz789",
  "subscription_item_id": "si_item123",
  "status": "active"
}
```

---

### Sync Usage to Stripe

```
POST /billing/sync-stripe?period=2026-05
```

Push current-period token usage to Stripe for metered billing. Requires admin role.

---

## Clinical Analytics

### Submit Query Feedback

```
POST /analytics/queries/{query_id}/feedback
```

Rate a query response for quality tracking.

**Request:**

```json
{
  "rating": 4,
  "feedback_tags": ["accurate", "helpful"],
  "patient_id": "patient-uuid",
  "flagged_urgent": false
}
```

| Field | Type | Description |
|-------|------|-------------|
| `rating` | int (1-5) | Quality score: 1 (poor) to 5 (excellent) |
| `feedback_tags` | string[] | Tags: `accurate`, `helpful`, `incomplete`, `needs_improvement`, `urgent` |
| `patient_id` | string | Optional patient ID (hashed before storage for HIPAA) |
| `flagged_urgent` | bool | Flag if response contained urgent clinical findings |

---

### Clinical Outcomes

```
GET /tenants/{tenant_id}/analytics/outcomes
```

Summary of clinical query quality: average ratings, feedback breakdown, urgent finding rate, quality trends.

---

### Cohort Management

#### Create a Cohort

```
POST /tenants/{tenant_id}/analytics/cohorts
```

**Request:**

```json
{
  "name": "Hypertension Patients 65+",
  "description": "Patients over 65 with hypertension diagnosis",
  "filter_criteria": {
    "conditions": ["hypertension"],
    "age_min": 65,
    "rag_modes": ["fast", "deep"]
  }
}
```

Supported filter keys: `conditions` (string[]), `medications` (string[]), `age_min` (int), `age_max` (int), `rag_modes` (string[]).

#### List Cohorts

```
GET /tenants/{tenant_id}/analytics/cohorts
```

#### Get Cohort Analytics

```
GET /tenants/{tenant_id}/analytics/cohorts/{cohort_id}
```

Returns query volume, average confidence, and feedback quality for the cohort.

---

### Population Trends

```
GET /tenants/{tenant_id}/analytics/population-trends
```

Population health trends: condition prevalence, query volume trends, confidence score trends.

---

### Urgent Alerts

#### List Alerts

```
GET /tenants/{tenant_id}/analytics/alerts
```

Returns urgent clinical finding alerts for the tenant.

#### Acknowledge an Alert

```
PATCH /tenants/{tenant_id}/analytics/alerts/{alert_id}/acknowledge
```

**Request:**

```json
{
  "notes": "Reviewed with attending physician, follow-up scheduled."
}
```

---

### Scheduled Reports

#### Schedule a Report

```
POST /tenants/{tenant_id}/analytics/reports
```

**Request:**

```json
{
  "report_type": "daily_usage",
  "schedule": "daily",
  "delivery_email": "admin@sunrise.com",
  "output_format": "pdf"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `report_type` | string | `daily_usage`, `weekly_usage`, `outcomes_summary`, `cohort_export` |
| `schedule` | string | `daily` or `weekly` |
| `delivery_email` | string | Optional email delivery address |
| `output_format` | string | `json`, `csv`, or `pdf` |

**Response:** `201 Created`

#### List Reports

```
GET /tenants/{tenant_id}/analytics/reports
```

#### Disable a Report

```
DELETE /tenants/{tenant_id}/analytics/reports/{report_id}
```

---

## FHIR R4 API

All FHIR endpoints are tenant-scoped via row-level security.

### Capability Statement

```
GET /fhir/metadata
```

Returns the FHIR R4 CapabilityStatement. No authentication required (per FHIR spec for discovery).

---

### Patient Resources

#### Search Patients

```
GET /fhir/Patient?gender=female&birthdate=gt1990-01-01&_count=20&_offset=0
```

| Parameter | Description |
|-----------|-------------|
| `gender` | `male`, `female`, `other`, `unknown` |
| `birthdate` | Date comparator prefix: `gt`, `lt`, `ge`, `le`, `eq` |
| `_count` | Results per page (max 200) |
| `_offset` | Pagination offset |

Returns a FHIR Bundle of Patient resources.

#### Read Patient

```
GET /fhir/Patient/{patient_id}
```

#### Patient Everything

```
GET /fhir/Patient/{patient_id}/$everything?_count=200
```

Returns all clinical resources for the patient (encounters, conditions, medications, observations, procedures, immunizations, imaging studies, diagnostic reports) in a single Bundle. Max 1000 entries.

---

### Clinical Resources

All clinical resource endpoints follow the same pattern:

| Resource | Search Path | Filters |
|----------|-------------|---------|
| Encounter | `GET /fhir/Encounter` | `subject`, `_class`, `date`, `_count`, `_offset` |
| Condition | `GET /fhir/Condition` | `subject`, `code`, `onset_date`, `_count`, `_offset` |
| MedicationRequest | `GET /fhir/MedicationRequest` | `subject`, `status`, `_count`, `_offset` |
| Procedure | `GET /fhir/Procedure` | `subject`, `code`, `_count`, `_offset` |
| Observation | `GET /fhir/Observation` | `subject`, `code`, `date`, `_count`, `_offset` |
| Immunization | `GET /fhir/Immunization` | `subject`, `vaccine_code`, `_count`, `_offset` |
| AllergyIntolerance | `GET /fhir/AllergyIntolerance` | `subject`, `code`, `_count`, `_offset` |
| ImagingStudy | `GET /fhir/ImagingStudy` | `subject`, `modality`, `_count`, `_offset` |
| DiagnosticReport | `GET /fhir/DiagnosticReport` | `subject`, `status`, `_count`, `_offset` |

Each resource also supports read by ID: `GET /fhir/{ResourceType}/{id}`

---

### Ingest FHIR Bundle

```
POST /fhir/Bundle
```

Ingest a FHIR transaction or batch Bundle from an EHR system. Resources are upserted by identifier.

**Request:** A valid FHIR R4 Bundle resource with `type: "transaction"` or `"batch"`.

---

## Medical Imaging

### Semantic Search

```
POST /imaging/search
```

Search over radiology reports using natural language. Results are ranked by vector similarity.

**Request:**

```json
{
  "question": "lung nodule findings in chest CT",
  "modality": "CT",
  "patient_id": null,
  "k": 5,
  "synthesise": true
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `question` | string (1-800) | *required* | Clinical question or finding description |
| `modality` | string | `null` | DICOM modality filter: `CT`, `MRI`, `CR`, `DX`, `US`, `PT`, `NM`, `MG` |
| `patient_id` | string | `null` | Scope to one patient |
| `k` | int (1-20) | `5` | Max results |
| `synthesise` | bool | `true` | LLM synthesis over raw hits |

**Response:** `200 OK`

```json
{
  "query_id": "q-img-abc123",
  "question": "lung nodule findings in chest CT",
  "answer": "Based on radiology reports, two chest CT scans show...",
  "sources": ["study-001", "study-002"],
  "hits": [
    {
      "study_uid": "1.2.3.4.5",
      "modality": "CT",
      "report_text": "...",
      "similarity_score": 0.94
    }
  ],
  "confidence_score": 0.88,
  "mode": "radiology_rag",
  "processing_time": 1.2,
  "imaging_available": true
}
```

---

### Ingest DICOM

```
POST /imaging/ingest
```

Upload a DICOM file. Extracts metadata and indexes the report text for semantic search. No pixel data is stored.

**Request:** `multipart/form-data` with a `.dcm` file. Requires admin or clinician role.

---

### List Imaging Studies

```
GET /imaging/studies?patient_id=abc&modality=CT&limit=20&offset=0
```

| Parameter | Description |
|-----------|-------------|
| `patient_id` | Filter by patient |
| `modality` | DICOM modality filter |
| `from_date` | Start date (ISO 8601) |
| `to_date` | End date (ISO 8601) |
| `report_status` | `pending`, `preliminary`, `final`, `addendum` |
| `limit` | Results per page |
| `offset` | Pagination offset |

---

### Get Imaging Report

```
GET /imaging/report/{study_uid}
```

Returns the full study record including report text.

---

## CDS Hooks (EHR Integration)

### Service Discovery

```
GET /cds-services
```

Returns the CDS Hooks service list per HL7 spec. No authentication required.

### Patient View Hook

```
POST /cds-services/medintelligent-patient-summary
```

Called by the EHR when a clinician opens a patient chart. Returns an AI-generated summary card with a SMART app launch link.

**Request:** Standard CDS Hooks request with `patient-view` hook context and prefetch data (patient, conditions, medications, observations).

---

## Health & Observability

### Health Check

```
GET /health
```

No authentication required. Returns component health status.

**Response:** `200 OK`

```json
{
  "status": "healthy",
  "uptime_seconds": 86400,
  "components": {
    "vector_store": "available",
    "deep_thinking_rag": "available",
    "rag_cag_system": "available",
    "knowledge_graph": "available",
    "mcp_client": "available",
    "database": "available"
  }
}
```

Status values: `healthy` (all components up), `degraded` (some down), `unhealthy` (all critical components down).

---

### System Stats

```
GET /stats
```

**Response:** `200 OK`

```json
{
  "total_queries": 1523,
  "fast_queries": 980,
  "deep_queries": 243,
  "rag_cag_queries": 200,
  "mcp_queries": 100,
  "average_response_time": 0.85,
  "success_rate": 0.993,
  "last_24h_queries": 342
}
```

---

### SLA Alerts

```
GET /alerts
```

No authentication required. Evaluates current metrics against SLA thresholds. Designed for Prometheus Alertmanager scraping and on-call dashboards.

**Response:** `200 OK`

```json
{
  "status": "yellow",
  "alerts": [
    {
      "name": "high_error_rate",
      "severity": "warning",
      "message": "Error rate 2.1% exceeds SLA threshold (1.0%)",
      "value": 2.1,
      "threshold": 1.0,
      "fired_at": "2026-05-01T12:00:00+00:00",
      "runbook": "1. Check /health for component status. 2. Review logs..."
    }
  ],
  "metrics": {
    "error_rate_pct": 2.1,
    "p99_latency_seconds": 3.2,
    "hallucination_rate_pct": 5.0,
    "health_status": "healthy",
    "health_score": 1.0,
    "total_requests": 1000,
    "total_errors": 21
  },
  "thresholds": {
    "error_rate_pct": 1.0,
    "p99_latency_seconds": 5.0,
    "hallucination_rate_pct": 10.0,
    "health_degraded_max_seconds": 300
  },
  "evaluated_at": "2026-05-01T12:00:00+00:00"
}
```

| Alert | Warning | Critical |
|-------|---------|----------|
| Error rate | > 1% | > 5% |
| P99 latency | > 5s | > 10s |
| Hallucination rate | > 10% | > 20% |
| Health degraded | any degradation | > 300s degraded or unhealthy |

---

## Admin Endpoints

All admin endpoints require `admin` or `service` role.

### List Tenants

```
GET /admin/tenants?status=active&plan=professional&search=sunrise&limit=20&offset=0
```

---

### Tenant Detail

```
GET /admin/tenants/{tenant_id}
```

Returns enriched tenant data: users, API keys, sandbox status, usage, quota.

---

### Suspend / Reactivate

```
POST /admin/tenants/{tenant_id}/suspend
POST /admin/tenants/{tenant_id}/reactivate
```

Suspending flushes tenant caches. Returns `409` if the tenant is already in the target state.

---

### Audit Logs

```
GET /admin/tenants/{tenant_id}/audit?limit=100&offset=0&start_date=2026-04-01&end_date=2026-05-01
GET /admin/audit?tenant_id=t-abc&limit=100
```

HIPAA-compliant audit log (per 164.312(b)). Records every API request with caller identity, action, resource, and timestamp. Patient IDs are hashed with HMAC-SHA256.

---

### Platform Stats

```
GET /admin/stats
```

Aggregate platform statistics: tenant count, user count, query totals, plan distribution.

---

### Report Management (Admin)

```
GET  /admin/reports?status=active&report_type=daily_usage&tenant_id=t-abc&limit=20
GET  /admin/reports/summary
PATCH /admin/reports/{report_id}/pause
PATCH /admin/reports/{report_id}/resume
POST  /admin/reports/{report_id}/trigger
```

---

### Sync All Tenants to Stripe

```
POST /billing/sync-all?period=2026-05
```

Idempotent batch sync of all tenants with active Stripe subscriptions. Requires admin role.

**Response:** `200 OK`

```json
{
  "synced": 12,
  "failed": 0,
  "skipped": 3,
  "billing_period": "2026-05"
}
```

---

## Webhooks

### Stripe Webhook

```
POST /billing/webhook
```

Receives Stripe events. Authenticated via Stripe signature verification (not JWT). Configure in Stripe Dashboard with your `STRIPE_WEBHOOK_SECRET`.

**Handled events:**
- `customer.subscription.deleted` — suspends the tenant
- `customer.subscription.updated` — logs status change
- `invoice.payment_failed` — logs payment failure

---

### Telemedicine Webhooks

```
POST /webhooks/telemedicine/subscriber
POST /webhooks/telemedicine/appointment
POST /webhooks/telemedicine/session
```

Authenticated via HMAC-SHA256 signature in `X-Telemedicine-Signature` header. Format: `t=<timestamp>,v1=<signature>`. 5-minute timestamp tolerance.

**Subscriber events:** `subscriber.created` (provisions tenant), `subscriber.plan_changed` (adjusts budget), `subscriber.cancelled` (suspends tenant).

---

## Quickstart Example

Register, log in, and run your first query in three API calls:

```bash
# 1. Register your clinic
curl -X POST https://api.medintelligent.ai/tenants/register \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My Clinic",
    "admin_email": "admin@myclinic.com",
    "plan": "starter",
    "admin_username": "admin",
    "admin_password": "MySecurePass123!"
  }'
# Save the access_token and api_key from the response

# 2. Run a query using the JWT from registration
curl -X POST https://api.medintelligent.ai/query \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <access_token>" \
  -d '{
    "question": "What patients have a diagnosis of hypertension?",
    "mode": "fast"
  }'

# 3. Check your usage
curl https://api.medintelligent.ai/billing/usage \
  -H "Authorization: Bearer <access_token>"
```

---

## Integration Guide: FHIR + CDS Hooks

### Connecting to an EHR

1. **Register your tenant** via `POST /tenants/register`
2. **Ingest patient data** via `POST /fhir/Bundle` with a FHIR transaction Bundle from your EHR
3. **Configure CDS Hooks** in your EHR to point at `GET /cds-services` for discovery
4. **Run queries** via `POST /query` using patient IDs from your FHIR data
5. **Review outcomes** via `GET /tenants/{id}/analytics/outcomes`

### FHIR Resource Support

Fully supported (read + search + ingest): Patient, Encounter, Condition, MedicationRequest, Procedure, Observation, Immunization, AllergyIntolerance, ImagingStudy, DiagnosticReport.

Declared in CapabilityStatement (stub): CarePlan, CareTeam, Device, DocumentReference, Goal, Location, Medication, Organization, Practitioner, PractitionerRole, Provenance, RelatedPerson, ServiceRequest.

---

## SDKs & Client Libraries

Interactive API documentation with try-it-out capability is available at `/docs` (Swagger UI). OpenAPI 3.0 schema is available at `/openapi.json` for client generation with tools like `openapi-generator`.

```bash
# Generate a Python client
openapi-generator generate -i https://api.medintelligent.ai/openapi.json -g python -o ./sdk

# Generate a TypeScript client
openapi-generator generate -i https://api.medintelligent.ai/openapi.json -g typescript-fetch -o ./sdk
```

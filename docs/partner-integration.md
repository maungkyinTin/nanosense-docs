# NanoSense Partner Integration Guide

This guide covers integrating your application with the NanoSense Medical RAG API as a partner. It includes BFF proxy templates, webhook coupling, FHIR data integration, and security requirements.

---

## Architecture Overview

### Browser-Based Integrations: Use a BFF

If your application has a browser-based frontend, your API key must **never** be exposed to the browser. Use a Backend-For-Frontend (BFF) proxy:

```
Partner Browser  →  Partner BFF (your backend)  →  NanoSense RAG API
                     X-API-Key injected here          POST /query
                     (never reaches browser)           GET /health
```

The BFF:
- Injects the `mrag_...` API key server-side on every upstream request
- Enforces mode restrictions (`mcp` is always blocked for browser clients)
- Forwards rate-limit headers (`X-RateLimit-*`, `X-Quota-*`) to the frontend
- Sanitizes errors so internal URLs and API keys are never leaked to the browser

### Server-to-Server Integrations: Use the SDK Directly

If your integration is backend-to-backend with no browser involvement, skip the BFF and use the Python or TypeScript SDK directly. See the [SDK Reference](sdk-reference.md).

```
Partner Backend  →  NanoSense RAG API  (SDK or direct HTTP, server-side only)
```

---

## Partner Onboarding Flow

1. **Register a partner tenant** via `POST /tenants/register-partner`. This returns an `mrag_...` API key. Store it securely — it is shown only once.

2. **Copy a BFF template** from the options below (Express.js or FastAPI). The template is production-ready except for the auth middleware placeholder.

3. **Set environment variables** on your BFF server:
   - `RAG_API_BASE_URL=https://api.nanosense.net`
   - `RAG_API_KEY=mrag_your_key_here`
   These values must remain on your server. Never set them in frontend code or commit them to source control.

4. **Replace the auth placeholder** in the BFF with your real authentication (Auth0, Cognito, custom JWT, etc.).

5. **Your frontend calls the BFF** at `POST /api/rag/query` using the `useRAGQuery` hook or any HTTP client. The BFF forwards the request upstream with the API key injected.

---

## BFF Templates

### Express.js BFF (Node 18+, ES modules)

**Environment variables:**

| Variable | Required | Description |
|----------|----------|-------------|
| `RAG_API_BASE_URL` | Yes | NanoSense API base URL |
| `RAG_API_KEY` | Yes | Your `mrag_...` API key |
| `PORT` | No | Port to listen on (default `3001`) |
| `ALLOWED_ORIGINS` | No | Comma-separated CORS origins (default `http://localhost:3000`) |
| `SESSION_SECRET` | No | Session signing secret — **change in production** |

**BFF routes:**

| Route | Upstream | Auth required |
|-------|----------|--------------|
| `POST /api/rag/query` | `POST /query` | Yes — via `requireAuth()` middleware |
| `GET /api/rag/health` | `GET /health` | No — safe for load balancers and k8s probes |

**Key behaviors:**
- `mcp` mode is hard-blocked with a 400 response. Enterprise tenants needing MCP should use the server-side SDK.
- Rate-limit headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `X-Quota-Remaining`, `X-Quota-Warning`) are forwarded from the upstream response to the browser.
- Queries time out after 90 seconds (sufficient for deep mode).
- Errors are sanitized — the `RAG_API_BASE_URL` and `RAG_API_KEY` values are never echoed in error responses.

**Auth placeholder — replace before deploying:**

The template ships with a session-based auth stub:

```js
function requireAuth(req, res, next) {
  if (!req.session?.userId) {
    return res.status(401).json({ error: 'Unauthorized', code: 401 });
  }
  next();
}
```

Replace this with your real auth. For example, Auth0 JWT validation:

```js
import { auth } from 'express-oauth2-jwt-bearer';

const requireAuth = auth({
  audience: process.env.AUTH0_AUDIENCE,
  issuerBaseURL: process.env.AUTH0_ISSUER_BASE_URL,
});
```

**Install and run:**

```bash
npm install express cors express-session
node server.js
```

---

### FastAPI BFF (Python 3.12+, httpx)

**Environment variables:**

| Variable | Required | Description |
|----------|----------|-------------|
| `RAG_API_BASE_URL` | Yes | NanoSense API base URL |
| `RAG_API_KEY` | Yes | Your `mrag_...` API key |
| `PORT` | No | Port to listen on (default `3001`) |
| `ALLOWED_ORIGINS` | No | Comma-separated CORS origins (default `http://localhost:3000`) |

**BFF routes:**

| Route | Upstream | Auth required |
|-------|----------|--------------|
| `POST /api/rag/query` | `POST /query` | Yes — via `get_current_user()` dependency |
| `GET /api/rag/health` | `GET /health` | No |

**Request body** for `POST /api/rag/query`:

```json
{
  "question": "What medications is patient P001 taking?",
  "mode": "fast",
  "patient_id": "P001"
}
```

`mode` is validated by Pydantic — only `fast`, `deep`, and `rag_cag` are accepted (`mcp` returns 422).

**Auth placeholder — replace before deploying:**

```python
async def get_current_user(authorization: str | None = Header(default=None)) -> str:
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Missing or invalid Authorization header")
    token = authorization.removeprefix("Bearer ").strip()
    # Replace with: decode JWT, look up session, call identity provider, etc.
    return token
```

**Install and run:**

```bash
pip install fastapi httpx uvicorn
uvicorn server:app --host 0.0.0.0 --port 3001
```

---

### React Hook (`useRAGQuery`)

A TypeScript React hook that calls your BFF (not the RAG API directly).

```tsx
import { useRAGQuery } from './useRAGQuery';

function QueryPanel({ patientId }: { patientId: string }) {
  const { query, isLoading, error, data, rateLimits } = useRAGQuery();

  const handleSubmit = async (question: string) => {
    await query(question, 'fast', patientId);
  };

  return (
    <div>
      {isLoading && <p>Loading...</p>}
      {error && <p className="error">{error}</p>}
      {data && (
        <div>
          <p>{data.answer}</p>
          <p>Confidence: {data.confidence_score}</p>
        </div>
      )}
      {rateLimits.remaining !== null && (
        <p>Requests remaining: {rateLimits.remaining}</p>
      )}
    </div>
  );
}
```

**Hook API:**

```ts
useRAGQuery({ bffBaseUrl?: string })
```

| Property | Type | Description |
|----------|------|-------------|
| `query(question, mode?, patient_id?)` | function | Send a query to the BFF |
| `isLoading` | boolean | True while request is in flight |
| `error` | string \| null | Error message if the request failed |
| `data` | `RAGQueryResponse \| null` | Successful response |
| `rateLimits` | `{ limit, remaining, reset }` | Parsed `X-RateLimit-*` headers |

**Behaviors:**
- In-flight requests are cancelled with `AbortController` when a new query fires or the component unmounts.
- 429 responses surface a user-friendly rate-limit message including the retry delay.
- 401 responses prompt "Session expired. Please log in again."
- Session cookies are sent to the BFF via `credentials: 'include'`.

---

## Server-to-Server Integration (No BFF)

For backend-to-backend integrations, use the SDK directly. The API key stays on your server and is never involved in a browser request.

```python
import os
from medintelligent import MedIntelligentClient

client = MedIntelligentClient(
    base_url=os.environ["NANOSENSE_BASE_URL"],
    api_key=os.environ["NANOSENSE_API_KEY"],
)

resp = client.query(
    "What is the current medication list for patient P007?",
    mode="rag_cag",
    patient_id="P007",
)
print(resp.answer)
```

```ts
import { MedIntelligentClient } from "@medintelligent/sdk";

const client = new MedIntelligentClient({
  baseUrl: process.env.NANOSENSE_BASE_URL!,
  apiKey: process.env.NANOSENSE_API_KEY,
});

const resp = await client.query({
  question: "What is the current medication list for patient P007?",
  mode: "rag_cag",
  patient_id: "P007",
});
```

---

## Webhook Integration (Telemedicine Coupling)

If your platform manages subscriber lifecycle, configure NanoSense to receive webhook events. This allows tenant provisioning to be driven by your subscription system rather than requiring manual registration calls.

### Webhook Endpoints

| Endpoint | Event Types | Description |
|----------|-------------|-------------|
| `POST /webhooks/telemedicine/subscriber` | `subscriber.created`, `subscriber.updated`, `subscriber.plan_changed`, `subscriber.cancelled` | Subscriber lifecycle events |
| `POST /webhooks/telemedicine/appointment` | appointment events | Appointment scheduling events |
| `POST /webhooks/telemedicine/session` | session events | Clinical session events |

### Signature Verification

All webhooks are authenticated with HMAC-SHA256. NanoSense will reject requests that fail signature verification or whose timestamp is more than 5 minutes old.

Each request includes:
- `X-Telemedicine-Signature: sha256=<hex_digest>` — HMAC-SHA256 of the raw request body using the shared `TELEMEDICINE_WEBHOOK_SECRET`
- `X-Telemedicine-Timestamp` — Unix timestamp of the event

Share the same secret in both your platform's outgoing webhook config and NanoSense's `TELEMEDICINE_WEBHOOK_SECRET` environment variable.

### Subscriber Lifecycle Events

When NanoSense receives a `subscriber.created` event, it automatically provisions a `partner_bundled` tenant with your custom token budget. The provisioned tenant gets a `mrag_...` API key that your platform can use for subsequent RAG queries on behalf of that subscriber.

Example `subscriber.created` payload:

```json
{
  "event": "subscriber.created",
  "timestamp": 1748000000,
  "data": {
    "subscriber_id": "sub_abc123",
    "email": "patient@example.com",
    "token_budget": 500000,
    "metadata": {
      "specialty": "cardiology",
      "timezone": "America/New_York"
    }
  }
}
```

A `subscriber.cancelled` event suspends the corresponding partner tenant and stops query processing.

---

## FHIR Data Integration

Push patient EHR data to NanoSense via `POST /fhir/Bundle`. Once ingested, the data is immediately available for RAG queries.

### Ingest a FHIR Bundle

```http
POST /fhir/Bundle
X-API-Key: mrag_your_key_here
Content-Type: application/json

{
  "resourceType": "Bundle",
  "type": "transaction",
  "entry": [
    {
      "resource": {
        "resourceType": "Patient",
        "id": "patient-001",
        "name": [{ "family": "Smith", "given": ["Jane"] }],
        "birthDate": "1980-03-15",
        "gender": "female"
      },
      "request": { "method": "PUT", "url": "Patient/patient-001" }
    },
    {
      "resource": {
        "resourceType": "Condition",
        "id": "cond-001",
        "subject": { "reference": "Patient/patient-001" },
        "code": {
          "coding": [{ "system": "http://snomed.info/sct", "code": "44054006", "display": "Diabetes mellitus type 2" }]
        },
        "clinicalStatus": { "coding": [{ "code": "active" }] }
      },
      "request": { "method": "PUT", "url": "Condition/cond-001" }
    }
  ]
}
```

**Response:**

```json
{
  "resourceType": "Bundle",
  "type": "transaction-response",
  "entry": [
    { "response": { "status": "201 Created", "location": "Patient/patient-001" } },
    { "response": { "status": "201 Created", "location": "Condition/cond-001" } }
  ]
}
```

### Supported Resource Types

| Resource Type | Read | Search | Ingest |
|---------------|------|--------|--------|
| Patient | Yes | Yes | Yes |
| Encounter | Yes | Yes | Yes |
| Condition | Yes | Yes | Yes |
| MedicationRequest | Yes | Yes | Yes |
| Procedure | Yes | Yes | Yes |
| Observation | Yes | Yes | Yes |
| Immunization | Yes | Yes | Yes |
| AllergyIntolerance | Yes | Yes | Yes |
| ImagingStudy | Yes | Yes | Yes |
| DiagnosticReport | Yes | Yes | Yes |

Resources not in this list return 422.

### Read Back Ingested Data

After ingest, data is immediately queryable via the FHIR search endpoints:

```http
GET /fhir/Condition?subject=patient-001
X-API-Key: mrag_your_key_here
```

Or retrieve everything for a patient:

```http
GET /fhir/Patient/patient-001/$everything
X-API-Key: mrag_your_key_here
```

---

## Partner Plan Details

| Feature | Partner Bundled |
|---------|----------------|
| Queries/month | 50,000 |
| Users | 5 |
| RAG Modes | fast, deep, rag_cag |
| Rate Limit | 30 RPM |
| Token Budget | Custom (set at registration) |
| Overage | N/A (custom budget — no overage) |

Note: `mcp` mode is not available on Partner Bundled plans. For MCP access, contact support about Enterprise plans.

---

## Common Issues

### CORS errors in browser

The BFF must set `ALLOWED_ORIGINS` to include your frontend's origin. The RAG API itself does not accept direct browser requests from partner frontends.

```bash
# .env
ALLOWED_ORIGINS=https://app.yourcompany.com,https://staging.yourcompany.com
```

### 401 on BFF query route

The auth middleware placeholder in the BFF template rejects all requests that lack a valid session or token. The BFF is intentionally non-functional until you replace the placeholder with your real authentication logic.

### mcp mode 400

`mcp` mode is intentionally blocked at the BFF layer — it returns a 400 before the request reaches the RAG API. Enterprise tenants that need MCP access should use the server-side Python or TypeScript SDK, not the browser BFF.

### 413 on FHIR bundle ingest

Large bundles (many resources) may exceed the default request body size limit in your BFF or API gateway. Increase the body size limit, or split the bundle into smaller transaction batches.

### Webhook signature failures

Verify that `TELEMEDICINE_WEBHOOK_SECRET` matches exactly between your platform's outgoing config and NanoSense's environment. Also ensure your system clock is within 5 minutes of UTC — expired timestamps are rejected.

---

## Coupling Requirements

### Prerequisites for Partner Integration

Before coupling your application with NanoSense, verify the following:

| Requirement | Description |
|-------------|-------------|
| **Tenant registration** | Register via `POST /tenants/register-partner` with `X-Partner-Key` header. Returns `mrag_...` API key (shown once). |
| **FHIR R4 Bundle format** | Patient data must be a valid FHIR R4 Bundle with `type: "transaction"`. Each entry needs a `resource` and `request` block. |
| **Supported resource types** | Patient, Encounter, Condition, MedicationRequest, Procedure, Observation, Immunization, AllergyIntolerance. Other types return 422. |
| **Request body limit** | Maximum 1 MB per request. Large bundles must be split into batches. |
| **TLS 1.2+** | All API traffic must use HTTPS. Plaintext HTTP is rejected. |
| **API key security** | Store `mrag_...` keys server-side only. Never embed in client-side code or source control. |

### Data Flow Requirements

```
Partner System                            NanoSense RAG API
──────────────                            ──────────────────
1. Register partner tenant  ────────────► POST /tenants/register-partner
   X-Partner-Key: <shared_key>            ← { api_key: "mrag_..." }

2. Push FHIR Bundle         ────────────► POST /fhir/Bundle
   X-API-Key: mrag_...                    ← { type: "transaction-response" }
   (batch if > 1 MB)                        Per-resource status codes

3. Query patient data        ────────────► POST /query
   X-API-Key: mrag_...                    ← { answer, confidence, sources }
   { question, mode, patient_id }

4. Check billing (optional)  ────────────► GET /billing/usage
   X-API-Key: mrag_...                    ← { total_tokens, included_tokens }
```

### Validated Coupling Points

The following integration points have been validated in production with real patient data:

| Coupling Point | Validated | Notes |
|----------------|-----------|-------|
| Tenant registration | Yes | Returns API key, tenant_id, plan details |
| FHIR Bundle ingest | Yes | 1,068 resources across 8 types; batched for 1 MB limit |
| FHIR read-back | Yes | Patient, Condition, MedicationRequest, Observation search |
| RAG query (fast) | Yes | < 4s response time, clinically accurate answers |
| RAG query (deep) | Yes | Multi-step reasoning, higher latency (~5s) |
| RAG query (rag_cag) | Yes | Retrieval + caching, balanced performance |
| Mode enforcement | Yes | Partner Bundled: fast/deep/rag_cag allowed; mcp blocked |
| Rate-limit headers | Yes | X-RateLimit-Limit/Remaining/Reset on all responses |
| Billing metering | Yes | Token usage tracked per tenant with correct plan limits |
| Tenant isolation | Yes | RLS enforced; API keys scoped to single tenant |

### Rate Limits for Partners

| Plan | Queries/Month | Rate Limit | Included Tokens |
|------|--------:|--------:|--------:|
| Partner Bundled | 50,000 | 30 RPM | Custom budget |

Partners who exceed the rate limit receive HTTP 429 with `Retry-After` and `X-RateLimit-Reset` headers.

### FHIR Bundle Batching Guide

The API enforces a 1 MB request body limit. For large patient bundles, split resources into batches:

| Batch | Resource Types | Typical Size |
|-------|---------------|-------------|
| 1 | Patient, Encounter, Condition, MedicationRequest, Immunization, AllergyIntolerance | ~280 KB |
| 2 | Observation, Procedure | ~520 KB |
| 3 | DiagnosticReport (optional) | ~235 KB |

Each batch is a separate `POST /fhir/Bundle` request. Resources are upserted by ID — re-ingesting the same bundle is idempotent.

---

## Security Checklist for Partners

Before going to production, verify the following:

- [ ] API key (`mrag_...`) is stored in a server-side environment variable only — not in frontend code, not in source control, not in client-side configuration files
- [ ] BFF auth middleware (`requireAuth` / `get_current_user`) has been replaced with real authentication
- [ ] `ALLOWED_ORIGINS` is restricted to your actual frontend domain(s)
- [ ] Session store uses Redis or a database in production (not the default in-memory store in the Express template)
- [ ] HTTPS is enforced end-to-end on the BFF
- [ ] Rate-limit headers are surfaced to users so they can see when they are approaching limits
- [ ] `SESSION_SECRET` (Express) has been changed from the default to a strong random value

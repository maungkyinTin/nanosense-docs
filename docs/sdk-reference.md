# NanoSense SDK Reference

Client libraries for the NanoSense Medical RAG API.

**Base URL:** `https://api.nanosense.net`

---

## Installation

### Python

```bash
pip install medintelligent
```

Requires Python 3.10+. The SDK depends on `httpx`.

### TypeScript / JavaScript

```bash
npm install @medintelligent/sdk
```

Requires Node.js 18+ or any modern browser runtime. The SDK uses the standard `fetch` API and has no required dependencies.

---

## Quick Start

### Python (API key auth)

```python
from medintelligent import MedIntelligentClient

client = MedIntelligentClient(
    base_url="https://api.nanosense.net",
    api_key="mrag_your_key_here",
)

resp = client.query("What medications is patient P001 taking?")
print(resp.answer)
print(f"Confidence: {resp.confidence_score}")
```

### TypeScript (API key auth)

```ts
import { MedIntelligentClient } from "@medintelligent/sdk";

const client = new MedIntelligentClient({
  baseUrl: "https://api.nanosense.net",
  apiKey: "mrag_your_key_here",
});

const resp = await client.query({ question: "What medications is patient P001 taking?" });
console.log(resp.answer);
console.log("Confidence:", resp.confidence_score);
```

---

## Authentication

Both SDKs support two authentication methods.

### API Key

Pass the `mrag_...` key obtained at tenant registration. The key is sent as `X-API-Key` on every request and never expires until revoked. Store it server-side; never embed it in browser code.

**Python:**
```python
client = MedIntelligentClient(
    base_url="https://api.nanosense.net",
    api_key="mrag_your_key_here",
)
```

**TypeScript:**
```ts
const client = new MedIntelligentClient({
  baseUrl: "https://api.nanosense.net",
  apiKey: "mrag_your_key_here",
});
```

### JWT (username + password)

Suitable for interactive applications where users log in directly. Tokens expire after 30 minutes.

**Python:**
```python
client = MedIntelligentClient(base_url="https://api.nanosense.net")
client.login(username="admin@clinic.com", password="s3cureP@ss!")
# JWT is stored on the client and used automatically for subsequent calls
resp = client.query("List active conditions for patient P042")
```

**TypeScript:**
```ts
const client = new MedIntelligentClient({ baseUrl: "https://api.nanosense.net" });
await client.login({ username: "admin@clinic.com", password: "s3cureP@ss!" });
const resp = await client.query({ question: "List active conditions for patient P042" });
```

---

## Environment Variables

The recommended pattern is to read credentials from environment variables rather than hardcoding them.

| Variable | Description |
|----------|-------------|
| `NANOSENSE_BASE_URL` | API base URL (e.g. `https://api.nanosense.net`) |
| `NANOSENSE_API_KEY` | API key (`mrag_...`) |

**Python:**
```python
import os
from medintelligent import MedIntelligentClient

client = MedIntelligentClient(
    base_url=os.environ["NANOSENSE_BASE_URL"],
    api_key=os.environ["NANOSENSE_API_KEY"],
)
```

**TypeScript:**
```ts
const client = new MedIntelligentClient({
  baseUrl: process.env.NANOSENSE_BASE_URL!,
  apiKey: process.env.NANOSENSE_API_KEY,
});
```

---

## Python SDK

### `MedIntelligentClient`

Synchronous HTTP client. Uses `httpx.Client` internally. Supports context manager (`with` statement) for automatic cleanup.

```python
MedIntelligentClient(
    base_url: str,           # required — API base URL
    api_key: str | None,     # optional — mrag_... key
    timeout: float = 60.0,   # request timeout in seconds
)
```

#### Context Manager

```python
with MedIntelligentClient(base_url="...", api_key="mrag_...") as client:
    resp = client.query("What are the latest vitals for patient P007?")
```

### `AsyncMedIntelligentClient`

Async client built on `httpx.AsyncClient`. Use with `async with` for cleanup.

```python
AsyncMedIntelligentClient(
    base_url: str,
    api_key: str | None,
    timeout: float = 60.0,
)
```

```python
import asyncio
from medintelligent import AsyncMedIntelligentClient

async def main():
    async with AsyncMedIntelligentClient(
        base_url="https://api.nanosense.net",
        api_key="mrag_your_key_here",
    ) as client:
        resp = await client.query("What are the latest vitals for patient P007?")
        print(resp.answer)

asyncio.run(main())
```

---

## TypeScript SDK

### `MedIntelligentClient`

All methods return `Promise<T>`. Works in Node.js and modern browsers.

```ts
new MedIntelligentClient({
  baseUrl: string,          // required — API base URL
  apiKey?: string,          // optional — mrag_... key
  timeoutMs?: number,       // default 60000 (60s)
  fetch?: typeof fetch,     // optional custom fetch implementation
})
```

---

## Methods

Both SDKs expose the same methods. Python uses `snake_case`; TypeScript uses `camelCase`.

### Authentication

#### `login(username, password)`

Authenticate and store the JWT for subsequent calls.

**Python:** `client.login(username="...", password="...")`
**TypeScript:** `await client.login({ username: "...", password: "..." })`

Returns a `TokenResponse` with `access_token`, `token_type`, and `expires_in`.

---

### Core

#### `health()`

Check API availability.

```python
status = client.health()
print(status.status)        # "healthy"
print(status.version)       # "3.0"
```

```ts
const status = await client.health();
console.log(status.status, status.version);
```

#### `stats()`

Return system-wide query statistics (total queries, uptime, etc.).

#### `query(question, ...)`

Submit a medical RAG query.

**Python:**
```python
resp = client.query(
    "What is the current blood pressure trend for patient P007?",
    mode="rag_cag",            # "fast" | "deep" | "rag_cag" | "mcp"
    patient_id="P007",
    include_sql=True,          # return generated SQL
    include_knowledge_graph=False,
    session_id="sess-abc123",  # for multi-turn conversations
    rag_cag_mode="smart_caching",  # sub-mode when mode="rag_cag"
)
print(resp.answer)
print(resp.confidence_score)
print(resp.sql_query)
```

**TypeScript:**
```ts
const resp = await client.query({
  question: "What is the current blood pressure trend for patient P007?",
  mode: "rag_cag",
  patient_id: "P007",
  include_sql: true,
});
console.log(resp.answer, resp.confidence_score);
```

`QueryResponse` fields:

| Field | Type | Description |
|-------|------|-------------|
| `query_id` | string | Unique query identifier |
| `question` | string | Echo of the submitted question |
| `answer` | string | LLM-generated answer |
| `mode` | string | Mode used |
| `processing_time` | float | Seconds elapsed |
| `confidence_score` | float \| null | 0–1 confidence estimate |
| `sources_used` | string[] | Source identifiers used |
| `sql_query` | string \| null | Generated SQL (if `include_sql=true`) |
| `knowledge_graph` | object \| null | KG payload (if requested) |
| `metadata` | object | Additional metadata |

**Query modes:**

| Mode | Description | Plan requirement |
|------|-------------|-----------------|
| `fast` | Single-pass SQL + LLM answer. Lowest latency (~5–10s). | All plans |
| `rag_cag` | RAG with context-aware generation. Best accuracy for most queries. | All plans |
| `deep` | Multi-step reasoning, cross-source synthesis. Higher latency (~30s). | Professional and above |
| `mcp` | Multi-Context Processing. Highest capability. | Enterprise only |
| `radiology` | Semantic search over radiology reports. | All plans |
| `auto` | Automatic mode selection based on question type. | All plans |

---

### Imaging

#### `imaging_search(question, ...)`

Semantic search over radiology reports.

**Python:**
```python
resp = client.imaging_search(
    "lung nodule findings in chest CT",
    modality="CT",       # optional DICOM modality filter
    patient_id="P007",   # optional patient scope
    k=5,                 # max results (1–20)
    synthesise=True,     # LLM synthesis over raw hits
)
print(resp.answer)
for hit in resp.hits:
    print(hit["study_uid"], hit["similarity_score"])
```

**TypeScript:**
```ts
const resp = await client.imagingSearch({
  question: "lung nodule findings in chest CT",
  modality: "CT",
  patient_id: "P007",
});
```

#### `imaging_ingest(file)`

Upload a DICOM file for indexing. Requires admin or clinician role.

**Python:**
```python
result = client.imaging_ingest("/path/to/scan.dcm")
```

**TypeScript:**
```ts
// Browser
const file = fileInput.files[0];
const result = await client.imagingIngest(file);

// Node.js
import { readFileSync } from "fs";
const blob = new Blob([readFileSync("scan.dcm")]);
const result = await client.imagingIngest(blob, "scan.dcm");
```

---

### Tenant Management

#### `register_tenant(org_name, email, password, *, tier="sandbox")`

Self-serve tenant registration. The returned API key is shown **only once** — store it immediately.

**Python:**
```python
tenant = client.register_tenant(
    org_name="Sunrise Clinic",
    email="admin@sunrise.com",
    password="S3cureP@ss!",
    tier="starter",
)
print("Your API key:", tenant.api_key)  # save this!
print("Tenant ID:", tenant.tenant_id)
```

**TypeScript:**
```ts
const tenant = await client.registerTenant({
  org_name: "Sunrise Clinic",
  email: "admin@sunrise.com",
  password: "S3cureP@ss!",
  tier: "starter",
});
console.log("Your API key:", tenant.api_key);
```

#### `get_tenant(tenant_id)` / `getTenant(tenantId)`

Retrieve tenant details.

#### `create_api_key(tenant_id, *, label, tier)` / `createApiKey(tenantId, req?)`

Generate a new API key for a tenant. Key is shown only once.

**Python:**
```python
key = client.create_api_key(tenant_id, label="production", tier="professional")
print(key.api_key)
```

#### `list_api_keys(tenant_id)` / `listApiKeys(tenantId)`

List all API keys for a tenant (key values are masked).

#### `sandbox_status(tenant_id)` / `sandboxStatus(tenantId)`

Check sandbox readiness and patient count.

#### `sandbox_provision(tenant_id)` / `sandboxProvision(tenantId)`

Provision the sandbox with synthetic EHR data.

---

### Billing

#### `billing_usage()` / `billingUsage()`

Return current token usage for the authenticated tenant.

```python
usage = client.billing_usage()
print(f"Tokens used: {usage.tokens_used} / {usage.tokens_included}")
```

#### `billing_status()` / `billingStatus()`

Return subscription status including Stripe customer and subscription IDs.

#### `billing_subscribe(tier, *, payment_method_id=None)` / `billingSubscribe(req)`

Subscribe to or upgrade a plan tier.

```python
client.billing_subscribe("professional", payment_method_id="pm_xxx")
```

---

## FHIR Data Ingest

Push patient data directly via the `POST /fhir/Bundle` API endpoint. The SDKs do not wrap FHIR ingest in a typed method — send the raw FHIR R4 Bundle as JSON.

**Python:**
```python
import json, httpx

bundle = json.load(open("patient_bundle.json"))

resp = httpx.post(
    "https://api.nanosense.net/fhir/Bundle",
    json=bundle,
    headers={"X-API-Key": "mrag_your_key_here"},
    timeout=120,
)
resp.raise_for_status()
print(resp.json())  # { "type": "transaction-response", ... }
```

**TypeScript:**
```ts
const bundle = await fetch("patient_bundle.json").then(r => r.json());

const resp = await fetch("https://api.nanosense.net/fhir/Bundle", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "X-API-Key": "mrag_your_key_here",
  },
  body: JSON.stringify(bundle),
});

const result = await resp.json();
console.log(result.type); // "transaction-response"
```

Supported resource types: Patient, Encounter, Condition, MedicationRequest, Procedure, Observation, Immunization, AllergyIntolerance, ImagingStudy, DiagnosticReport.

---

## Error Handling

Both SDKs raise typed exceptions for API errors. Catch the base `MedIntelligentError` for a catch-all, or specific subclasses for fine-grained handling.

### Python Exceptions

| Exception | HTTP Status | When raised |
|-----------|-------------|-------------|
| `MedIntelligentError` | any | Base class for all SDK errors |
| `AuthenticationError` | 401 | Missing or expired credentials |
| `AuthorizationError` | 403 | Insufficient role or plan |
| `NotFoundError` | 404 | Resource not found |
| `ValidationError` | 422 | Invalid request body or parameters |
| `RateLimitError` | 429 | Rate limit exceeded |
| `ServerError` | 5xx | Internal server error |

```python
from medintelligent import MedIntelligentClient
from medintelligent.exceptions import (
    AuthorizationError,
    RateLimitError,
    MedIntelligentError,
)

client = MedIntelligentClient(
    base_url="https://api.nanosense.net",
    api_key="mrag_your_key_here",
)

try:
    resp = client.query("What are the latest labs?", mode="deep")
except AuthorizationError:
    print("Deep mode not available on your plan. Upgrade to Professional.")
except RateLimitError as e:
    print(f"Rate limited: {e.detail}")
except MedIntelligentError as e:
    print(f"API error {e.status_code}: {e.detail}")
```

### TypeScript Errors

| Error Class | HTTP Status | When raised |
|-------------|-------------|-------------|
| `MedIntelligentError` | any | Base class for all SDK errors |
| `AuthenticationError` | 401 | Missing or expired credentials |
| `AuthorizationError` | 403 | Insufficient role or plan |
| `NotFoundError` | 404 | Resource not found |
| `ValidationError` | 422 | Invalid request body or parameters |
| `RateLimitError` | 429 | Rate limit exceeded |
| `ServerError` | 5xx | Internal server error |

```ts
import {
  MedIntelligentClient,
  AuthorizationError,
  RateLimitError,
  MedIntelligentError,
} from "@medintelligent/sdk";

const client = new MedIntelligentClient({
  baseUrl: "https://api.nanosense.net",
  apiKey: "mrag_your_key_here",
});

try {
  const resp = await client.query({ question: "What are the latest labs?", mode: "deep" });
} catch (e) {
  if (e instanceof AuthorizationError) {
    console.error("Deep mode not available on your plan.");
  } else if (e instanceof RateLimitError) {
    console.error("Rate limited:", e.detail);
  } else if (e instanceof MedIntelligentError) {
    console.error(`API error ${e.statusCode}:`, e.detail);
  }
}
```

---

## Async Usage

### Python asyncio

```python
import asyncio
from medintelligent import AsyncMedIntelligentClient

async def run_queries():
    async with AsyncMedIntelligentClient(
        base_url="https://api.nanosense.net",
        api_key="mrag_your_key_here",
    ) as client:
        # Run two queries concurrently
        fast, deep = await asyncio.gather(
            client.query("What medications is patient P001 taking?", mode="fast"),
            client.query("Summarize P001 clinical history", mode="rag_cag"),
        )
        print(fast.answer)
        print(deep.answer)

asyncio.run(run_queries())
```

### TypeScript async/await

All TypeScript SDK methods return `Promise<T>` and are always async.

```ts
import { MedIntelligentClient } from "@medintelligent/sdk";

const client = new MedIntelligentClient({
  baseUrl: "https://api.nanosense.net",
  apiKey: "mrag_your_key_here",
});

// Run two queries concurrently
const [fast, deep] = await Promise.all([
  client.query({ question: "What medications is patient P001 taking?", mode: "fast" }),
  client.query({ question: "Summarize P001 clinical history", mode: "rag_cag" }),
]);

console.log(fast.answer);
console.log(deep.answer);
```

---

## Complete Example: Clinical Workflow

The following example demonstrates a typical clinical query workflow: authenticate, check health, run queries in multiple modes, and inspect billing usage.

```python
import os
from medintelligent import MedIntelligentClient
from medintelligent.exceptions import MedIntelligentError

client = MedIntelligentClient(
    base_url=os.environ["NANOSENSE_BASE_URL"],
    api_key=os.environ["NANOSENSE_API_KEY"],
)

# 1. Check health
health = client.health()
assert health.status == "healthy", f"API not healthy: {health.status}"

# 2. Fast query — current medications
meds = client.query(
    "What medications is patient P007 currently taking?",
    mode="fast",
    patient_id="P007",
)
print("Medications:", meds.answer)
print(f"  Confidence: {meds.confidence_score:.0%}  Time: {meds.processing_time:.1f}s")

# 3. RAG+CAG query — clinical summary
summary = client.query(
    "Provide a clinical summary for patient P007 including conditions, medications, and recent vitals.",
    mode="rag_cag",
    patient_id="P007",
)
print("Summary:", summary.answer)

# 4. Check billing usage
usage = client.billing_usage()
print(f"Tokens used this month: {usage.tokens_used:,}")
```

---

## Method Reference

### Python — `MedIntelligentClient` and `AsyncMedIntelligentClient`

| Method | Description |
|--------|-------------|
| `login(username, password)` | Authenticate and store JWT |
| `health()` | API health status |
| `stats()` | System-wide query statistics |
| `query(question, *, mode, patient_id, ...)` | Submit a RAG query |
| `imaging_search(question, *, modality, patient_id, k, synthesise)` | Semantic search over radiology reports |
| `imaging_ingest(dicom_file_path)` | Upload a DICOM file |
| `imaging_status()` | Imaging service availability |
| `register_tenant(org_name, email, password, *, tier)` | Self-serve tenant registration |
| `get_tenant(tenant_id)` | Retrieve tenant details |
| `create_api_key(tenant_id, *, label, tier)` | Generate a new API key |
| `list_api_keys(tenant_id)` | List API keys for a tenant |
| `billing_usage()` | Current token usage |
| `billing_status()` | Subscription status |
| `billing_subscribe(tier, *, payment_method_id)` | Subscribe or upgrade |
| `sandbox_status(tenant_id)` | Sandbox readiness and patient count |
| `sandbox_provision(tenant_id)` | Provision synthetic sandbox data |
| `close()` | Close the underlying HTTP client |

### TypeScript — `MedIntelligentClient`

| Method | Description |
|--------|-------------|
| `login(req)` | Authenticate and store JWT |
| `tenantLogin(req)` | Tenant-scoped login |
| `health()` | API health status |
| `stats()` | System-wide query statistics |
| `query(req)` | Submit a RAG query |
| `imagingSearch(req)` | Semantic search over radiology reports |
| `imagingIngest(file, filename?)` | Upload a DICOM file |
| `imagingStatus()` | Imaging service availability |
| `registerTenant(req)` | Self-serve tenant registration |
| `getTenant(tenantId)` | Retrieve tenant details |
| `createApiKey(tenantId, req?)` | Generate a new API key |
| `listApiKeys(tenantId)` | List API keys for a tenant |
| `billingUsage()` | Current token usage |
| `billingStatus()` | Subscription status |
| `billingSubscribe(req)` | Subscribe or upgrade |
| `sandboxStatus(tenantId)` | Sandbox readiness and patient count |
| `sandboxProvision(tenantId)` | Provision synthetic sandbox data |

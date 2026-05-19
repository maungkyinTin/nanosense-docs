# NanoSense Platform Validation Report

Production API validation results for all subscriber tiers. Tests run against the live API at `api.nanosense.net`.

---

## Test Summary

| Category | Tests | Status |
|----------|------:|--------|
| FHIR Bundle Ingest | 9 | All pass |
| Mode Access Enforcement | 12 | All pass |
| Rate-Limit Headers | 4 | All pass |
| Query Performance | 3 | All pass |
| Clinical Accuracy | 3 | All pass |
| Billing & Usage | 3 | All pass |
| Cross-Tenant Isolation | 2 | All pass |
| Response Structure | 3 | All pass |
| **Total** | **39** | **All pass** |

---

## Mode Access Matrix (Verified)

| Mode | Starter | Professional | Enterprise |
|------|---------|-------------|------------|
| fast | 200 | 200 | 200 |
| rag_cag | 200 | 200 | 200 |
| deep | 403 | 200 | 200 |
| mcp | 403 | 403 | 200 |

All mode restrictions enforced at the API layer. Blocked modes return HTTP 403 with a clear error message.

---

## Rate Limits (Verified)

| Plan | Documented RPM | Measured `X-RateLimit-Limit` | Match |
|------|---------------:|-----------------------------:|-------|
| Starter | 10 | 10 | Yes |
| Professional | 30 | 30 | Yes |
| Enterprise | 100 | 100 | Yes |

Every `/query` response includes `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset` headers.

---

## Query Performance

| Tier | Mode | Processing Time | Wall Clock |
|------|------|----------------:|-----------:|
| Starter | fast | 0.75s | 1.82s |
| Professional | fast | 0.60s | 1.66s |
| Enterprise | fast | 3.66s | 5.63s |

All queries completed within the 60-second SLA threshold. Wall clock includes network round-trip to the API Gateway.

---

## Billing Accuracy (Verified)

| Plan | Documented Included Tokens | Measured `included_tokens` | Match |
|------|---------------------------:|---------------------------:|-------|
| Starter | 100,000 | 100,000 | Yes |
| Professional | 1,000,000 | 1,000,000 | Yes |
| Enterprise | 10,000,000 | 10,000,000 | Yes |

Token usage metering and overage calculation are accurate per the documented rates.

---

## FHIR R4 Ingest Validation

Tested with a 1,068-resource Synthea patient bundle (3.6 MB, batched into 3 requests for the 1 MB body limit).

| Resource Type | Submitted | Created | Status |
|---------------|----------:|--------:|--------|
| Patient | 1 | 1 | Pass |
| Encounter | 68 | 68 | Pass |
| Condition | 49 | 49 | Pass |
| MedicationRequest | 48 | 48 | Pass |
| Procedure | 203 | 203 | Pass |
| Observation | 213 | 213 | Pass |
| Immunization | 13 | 13 | Pass |
| AllergyIntolerance | 12 | 12 | Pass |

All supported FHIR R4 resource types ingest correctly with accurate counts.

---

## Clinical Accuracy

After FHIR ingest, RAG queries return clinically relevant answers grounded in the ingested patient data:

- **Conditions query**: Returns documented conditions (ischemic heart disease, chronic sinusitis, chronic pain)
- **Medications query**: Returns active prescriptions (fluticasone, albuterol) with dosing details
- **Allergies query**: Returns documented allergies (aspirin, latex, bee venom, mold, animal dander)

All answers include confidence scores and source citations.

---

## Tenant Isolation

- Each API key is scoped to a single tenant
- `/billing/usage` returns only the authenticated tenant's data
- One tenant's API key cannot access another tenant's billing, patient data, or query history
- Row-Level Security (RLS) enforced at the PostgreSQL layer

---

## Test Methodology

- **API**: All tests hit the live production API via HTTPS
- **Authentication**: Real `mrag_...` API keys registered per tier
- **Patient data**: Synthea-generated FHIR R4 patient bundle (Annalisa973 Glover433)
- **Assertions**: HTTP status codes, response body fields, header values, clinical term matching
- **Framework**: pytest with httpx (synchronous HTTP client)
- **Duration**: Full suite completes in ~2 minutes

---

*Last validated: May 2026. Test suite: `tests/test_prod_subscriber_tiers.py` (39 tests).*

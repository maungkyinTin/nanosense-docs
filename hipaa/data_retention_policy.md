# DATA RETENTION AND DISPOSAL POLICY

**Policy Owner:** MedIntelligent, Inc.
**Effective Date:** _______________
**Last Reviewed:** May 2026
**Review Cadence:** Annual (or upon material regulatory change)
**Regulatory Basis:** 45 CFR 164.530(j), 45 CFR 164.312(b), 21 CFR Part 11

---

## 1. PURPOSE

This policy defines how MedIntelligent retains, archives, and disposes of data — including Protected Health Information (PHI) and electronic Protected Health Information (ePHI) — in compliance with HIPAA, HITECH, and applicable state retention laws. It applies to all data created, received, maintained, or transmitted by the MedIntelligent platform on behalf of Covered Entities.

---

## 2. SCOPE

This policy covers all data stored in:

- **PostgreSQL (RDS)** — clinical records, tenant data, user accounts, billing records
- **Audit logs** — API request logs, query-level audit trails, database-level query logs (pgaudit)
- **Consent records** — per-patient, per-purpose consent grants and revocations
- **Object storage (S3)** — archived audit logs, report exports, DICOM metadata
- **Vector stores** — radiology report embeddings (LanceDB)
- **Application logs** — structured JSON logs in CloudWatch
- **Backups** — RDS automated snapshots, manual snapshots

Data NOT in scope: Stripe payment records (retained by Stripe per their data policy), aggregated anonymous analytics with no PHI.

---

## 3. RETENTION PERIODS

### 3.1 Clinical Data

| Data Category | Table(s) | Minimum Retention | Basis | Disposal Method |
|---|---|---|---|---|
| Patient demographics | `patients` | 6 years | HIPAA 164.530(j) | Soft-delete |
| Encounters | `encounters` | 6 years | HIPAA 164.530(j) | Soft-delete |
| Conditions / Diagnoses | `conditions` | 6 years | HIPAA 164.530(j) | Soft-delete |
| Medication requests | `medication_requests` | 6 years | HIPAA 164.530(j) | Soft-delete |
| Procedures | `procedures` | 6 years | HIPAA 164.530(j) | Soft-delete |
| Observations / Labs | `observations` | 6 years | HIPAA 164.530(j) | Soft-delete |
| Immunizations | `immunizations` | 6 years | HIPAA 164.530(j) | Soft-delete |
| Imaging metadata | `imaging_studies` | 7 years | ACR/CMS + 21 CFR Part 11 | Soft-delete |

> **Note:** Some states require longer retention (e.g., 10 years in New York, 7 years in California). MedIntelligent applies the longer of the federal minimum or the Covered Entity's state requirement when specified in the Order Form.

### 3.2 Audit and Compliance Data

| Data Category | Table(s) | Minimum Retention | Basis | Disposal Method |
|---|---|---|---|---|
| API audit log | `audit_log` | 10 years | HIPAA 164.312(b), OCR guidance | Archive to S3, then delete from live DB |
| Database query audit | `query_audit_log` | 10 years | HIPAA 164.312(b) | Archive to S3 |
| Consent records | `consent_records` | 6 years from last activity | HIPAA 164.508 | Soft-delete |

### 3.3 Operational Data

| Data Category | Retention | Disposal Method |
|---|---|---|
| Application logs (CloudWatch) | 1 year (configurable) | CloudWatch log group TTL auto-expire |
| RDS automated backups | 7 days (configurable, minimum 7) | Automatic expiry by AWS |
| Manual RDS snapshots | Until explicitly deleted | Manual review during annual audit |
| Report exports (S3) | 90 days | S3 lifecycle rule auto-expire |
| Archived audit logs (S3) | 10 years from original event date | S3 lifecycle rule to Glacier after 1 year, expire after 10 years |

### 3.4 Tenant Data Upon Termination

When a Covered Entity's subscription is terminated (see BAA Section 2.6):

| Scenario | Timeline | Action |
|---|---|---|
| CE requests data return | Within 30 days | Export all tenant data as FHIR R4 Bundle + audit log NDJSON; deliver via secure channel |
| CE requests data destruction | Within 30 days | Soft-delete all clinical records; hard-delete after 6-year retention period; provide written certification |
| CE makes no election | 90 days post-termination | Treat as data destruction request; retain audit logs per Section 3.2 |

---

## 4. PHI HANDLING IN RETAINED DATA

All retained data containing PHI is subject to the following safeguards regardless of retention period:

### 4.1 Identifiers

- **Patient IDs** are stored as HMAC-SHA256 hashes (truncated to 16 characters) using a per-environment salt (`PHI_HASH_SALT`). Raw patient identifiers are never persisted in audit logs, application logs, or analytics tables.
- **Question text** from clinical queries is never stored in audit logs. Only `question_length` (integer) is recorded.
- **HIPAA Safe Harbor reference:** 45 CFR 164.514(b)

### 4.2 Encryption

- **At rest:** AES-256 via AWS KMS-managed keys (separate keys for RDS, S3, and Secrets Manager)
- **In transit:** TLS 1.2+ enforced on all database connections (`rds.force_ssl = 1`) and API traffic
- **Backups:** Inherit the encryption of the source RDS instance

### 4.3 Tenant Isolation

All clinical tables enforce PostgreSQL Row-Level Security (RLS). Each query session sets `app.current_tenant_id` via `SET LOCAL`, ensuring one tenant's data is never visible to another — including during retention enforcement.

### 4.4 Log Sanitization

Application logs pass through a PHI filter that replaces sensitive fields (`patient_id`, `patient_name`, `ssn`, `question`, `mrn`, `date_of_birth`) with `[PHI_REDACTED]` before output.

---

## 5. DISPOSAL METHODS

### 5.1 Soft-Delete

The primary disposal method for clinical records. Rows are marked `is_deleted = TRUE` and `deleted_at = NOW()` but remain in the database for the duration of the retention period. Soft-deleted rows are:

- Excluded from all API queries (WHERE filters enforce `is_deleted = FALSE`)
- Excluded from FHIR R4 search results
- Retained for audit, legal hold, and regulatory inspection
- Subject to the same encryption and access controls as active records

### 5.2 Hard-Delete

After the retention period expires and no legal hold is active, soft-deleted rows may be permanently removed. Hard-delete is used only when:

- The retention period has fully elapsed
- No litigation hold or regulatory investigation is pending
- The retention enforcer is run in live mode (`RETENTION_ENFORCE_DRY_RUN=false`)

### 5.3 Archive to S3

Used for high-volume audit data. Rows are exported as NDJSON (newline-delimited JSON) to an S3 bucket with:

- Server-side encryption (AES-256)
- S3 Object Lock (compliance mode) when enabled
- Lifecycle rules: transition to S3 Glacier after 1 year, expire after retention period
- Path convention: `audit-archive/{YYYY/MM/DD}/audit_log_{cutoff_date}.ndjson`

After successful upload and verification, the archived rows are deleted from the live `audit_log` table.

---

## 6. ENFORCEMENT

### 6.1 Automated Enforcement

Retention policies are stored in the `data_retention_policies` database table and enforced by `retention_enforcer.py`, which:

1. Reads active policies from the database
2. Identifies rows past the retention cutoff for each table
3. Applies the configured disposal method (soft-delete, hard-delete, or S3 archive)
4. Logs all actions (table, row count, cutoff date, strategy)

The enforcer is designed to run daily as an AWS Lambda function, ECS scheduled task, or cron job.

### 6.2 Dry-Run Mode

By default, the enforcer runs in dry-run mode (`RETENTION_ENFORCE_DRY_RUN=true`), reporting what would be affected without modifying any data. Live enforcement requires explicit opt-in.

### 6.3 Batch Processing

Enforcement processes rows in configurable batches (`RETENTION_BATCH_SIZE`, default 1,000) to avoid long-running transactions and lock contention on production databases.

### 6.4 Configuration

| Environment Variable | Default | Description |
|---|---|---|
| `RETENTION_ENFORCE_DRY_RUN` | `true` | Set to `false` for live enforcement |
| `RETENTION_BATCH_SIZE` | `1000` | Rows per batch operation |
| `AUDIT_ARCHIVE_S3_BUCKET` | *(none)* | S3 bucket for audit log archival |
| `AUDIT_ARCHIVE_AFTER_YEARS` | `10` | Years before audit rows are archived to S3 |

---

## 7. LEGAL HOLDS

When MedIntelligent is notified of pending litigation, regulatory investigation, or audit involving a specific tenant or time period:

1. A legal hold flag is set on the affected tenant or records
2. The retention enforcer skips all flagged records regardless of retention period
3. The hold is documented in the `data_retention_policies` notes and communicated to the Privacy Officer
4. The hold remains in effect until written release from legal counsel

---

## 8. TENANT RIGHTS

### 8.1 Data Export

Covered Entities may request a full export of their data at any time via:

- **FHIR R4 API:** `GET /fhir/Patient/{id}/$everything` for individual patients, or `GET /fhir/{ResourceType}` for bulk search
- **Audit log API:** `GET /admin/tenants/{tenant_id}/audit` with date range filters
- **Bulk export:** Requested through the admin dashboard or support channel; delivered as encrypted FHIR Bundle + NDJSON audit archive

### 8.2 Data Deletion Requests

Individual patient deletion requests (per 45 CFR 164.524) are processed as soft-deletes within 30 days. The Covered Entity must submit the request through the admin API or support channel with the patient identifier.

### 8.3 Accounting of Disclosures

MedIntelligent maintains audit logs sufficient to produce an accounting of disclosures per 45 CFR 164.528. Requests are fulfilled within 30 days via the audit log export.

---

## 9. ANNUAL REVIEW

This policy is reviewed annually by the Privacy Officer and Security Officer. The review verifies:

- [ ] Retention periods remain compliant with current federal and state regulations
- [ ] The retention enforcer has run successfully in the past 12 months
- [ ] Audit log archival is current (no backlog exceeding 30 days)
- [ ] No legal holds have expired without release
- [ ] Disposal certifications are on file for terminated tenants
- [ ] Backup retention periods align with this policy
- [ ] S3 lifecycle rules match the documented retention periods

---

## 10. REFERENCES

| Citation | Description |
|---|---|
| 45 CFR 164.530(j) | HIPAA documentation and retention requirements (6-year minimum) |
| 45 CFR 164.312(b) | HIPAA audit controls |
| 45 CFR 164.508 | Uses and disclosures requiring authorization (consent) |
| 45 CFR 164.524 | Individual right of access to PHI |
| 45 CFR 164.528 | Accounting of disclosures |
| 21 CFR Part 11 | Electronic records and signatures (imaging) |
| ACR Practice Parameters | Radiology record retention guidance |

---

## 11. RELATED DOCUMENTS

| Document | Location |
|---|---|
| Business Associate Agreement | `hipaa/baa_template.md` |
| HIPAA Gap Assessment Tool | `generate_hipaa_gap_report.py` |
| Retention Enforcer | `retention_enforcer.py` |
| HIPAA DB Migration (schema) | `hipaa_db_migration.py` |
| RDS Security Checker | `check_rds_security.py` |
| PHI Utilities | `phi_utils.py` |

---

*This policy is provided for informational purposes and does not constitute legal advice. Consult legal counsel to ensure compliance with all applicable federal, state, and local regulations. Last reviewed May 2026.*

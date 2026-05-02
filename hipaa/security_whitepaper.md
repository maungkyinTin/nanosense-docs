# SECURITY PRACTICES WHITEPAPER

**MedIntelligent, Inc.**
**Version:** 1.0
**Last Updated:** May 2026
**Classification:** Customer-Facing — No Confidential Implementation Details

---

## Executive Summary

MedIntelligent is a HIPAA-ready Clinical Decision Support (CDS) and Medical AI platform that processes Protected Health Information (PHI) on behalf of healthcare organizations. This document describes the administrative, physical, and technical safeguards MedIntelligent implements to protect the confidentiality, integrity, and availability of electronic PHI (ePHI), mapped to the HIPAA Security Rule (45 CFR Part 164, Subpart C).

MedIntelligent operates under a Business Associate Agreement (BAA) with each Covered Entity customer.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Data Encryption](#2-data-encryption)
3. [Authentication and Access Control](#3-authentication-and-access-control)
4. [Tenant Isolation](#4-tenant-isolation)
5. [Audit Logging](#5-audit-logging)
6. [PHI Protection](#6-phi-protection)
7. [Network Security](#7-network-security)
8. [Data Retention and Disposal](#8-data-retention-and-disposal)
9. [Incident Response](#9-incident-response)
10. [Business Continuity](#10-business-continuity)
11. [Vulnerability Management](#11-vulnerability-management)
12. [Subprocessors](#12-subprocessors)
13. [Compliance Validation](#13-compliance-validation)
14. [Customer Responsibilities](#14-customer-responsibilities)
15. [Contact](#15-contact)

---

## 1. Architecture Overview

MedIntelligent runs on Amazon Web Services (AWS) in the US-East-1 region. The architecture follows a defense-in-depth model with multiple security boundaries:

```
Internet
   │
   ▼
AWS API Gateway (TLS 1.2+, ACM certificate)
   │
   ▼ (VPC Link — private)
Internal ALB (private subnets only)
   │
   ▼
ECS Fargate Tasks (private subnets, no public IP)
   │
   ▼
RDS PostgreSQL (private subnets, no public access, KMS encryption)
```

**Key architectural decisions:**

- **No public database access.** RDS instances are in private subnets with security groups allowing inbound connections only from ECS tasks and the jumpbox.
- **No public IPs on compute.** ECS Fargate tasks run in private subnets; outbound traffic routes through a NAT Gateway.
- **API Gateway as the sole entry point.** External traffic enters only through AWS API Gateway with TLS termination, then traverses a VPC Link to the internal ALB. The ALB is not internet-facing.
- **VPC Flow Logs enabled.** All network traffic is logged to CloudWatch for forensic analysis.

---

## 2. Data Encryption

### 2.1 Encryption at Rest

| Data Store | Encryption | Key Management |
|---|---|---|
| RDS PostgreSQL | AES-256 | Customer-managed AWS KMS key with automatic annual rotation |
| RDS Backups | AES-256 | Inherits source instance KMS key |
| RDS Performance Insights | AES-256 | Dedicated KMS key |
| S3 (audit archives, reports) | AES-256 | Dedicated S3 KMS key with automatic rotation |
| Secrets Manager (DB credentials) | AES-256 | Dedicated application KMS key |

All KMS keys are customer-managed (not AWS-managed defaults) with automatic annual rotation enabled and configurable deletion windows to prevent accidental key loss.

### 2.2 Encryption in Transit

| Connection | Protocol | Enforcement |
|---|---|---|
| Client → API Gateway | TLS 1.2+ | ACM-managed certificate; HTTP not accepted |
| API Gateway → ALB (VPC Link) | TLS | Private network, VPC-internal |
| Application → RDS | TLS 1.2+ | RDS parameter `rds.force_ssl = 1` (server-enforced; plaintext connections rejected) |
| Application → AWS Services | TLS 1.2+ | AWS SDK default; VPC Gateway Endpoints for S3 and DynamoDB |

### 2.3 Sensitive Field Encryption

Beyond storage-level encryption, individual sensitive fields (e.g., SSN) are encrypted at the application layer using Fernet (AES-128-CBC with HMAC-SHA256 authentication) before database insertion. This provides defense-in-depth: even a database administrator with SELECT access sees only ciphertext.

**HIPAA references:** 45 CFR 164.312(a)(2)(iv) — Encryption and decryption; 45 CFR 164.312(e)(1) — Transmission security; 45 CFR 164.312(e)(2)(ii) — Encryption mechanism.

---

## 3. Authentication and Access Control

### 3.1 Authentication Schemes

MedIntelligent supports two authentication methods:

**JWT Bearer Tokens:**
- Algorithm: HMAC-SHA256 (HS256)
- Expiration: 30 minutes (configurable)
- Claims: user ID (`sub`), tenant ID, role, issued-at, expiration
- Obtained via `POST /auth/login` with username and password

**Tenant API Keys:**
- Prefixed format (`mrag_*`) for easy identification and secret scanning
- Stored as bcrypt hashes (plaintext never persisted after initial generation)
- Revocable at any time via the management API
- Scoped to a single tenant

### 3.2 Password Security

- Passwords hashed with bcrypt (adaptive cost factor)
- Minimum 8 characters enforced at the API boundary
- Passwords never logged, cached, or stored in plaintext

### 3.3 Brute-Force Protection

An in-memory login attempt tracker enforces account lockout:
- **Threshold:** 5 failed attempts (configurable)
- **Lockout duration:** 15 minutes (configurable)
- Failed attempt counts reset on successful login

### 3.4 Role-Based Access Control (RBAC)

Every API request is authorized against the caller's role:

| Role | Capabilities |
|---|---|
| `admin` | Full tenant management, billing, user management, reporting, analytics |
| `clinician` | Clinical queries, FHIR read access, feedback submission, analytics read |
| `user` | Clinical queries only |
| `readonly` | Read-only access to own tenant data |
| `service` | Platform-level operations (internal/partner integrations) |

Protected endpoints enforce role checks via a `require_role()` dependency that returns HTTP 403 for unauthorized roles.

### 3.5 Rate Limiting

Per-tenant rate limiting prevents abuse and ensures fair resource allocation:

| Plan | Rate Limit |
|---|---|
| Starter | 10 requests/minute |
| Professional | 30 requests/minute |
| Enterprise | 100 requests/minute |

Rate limits are enforced at the application layer with a token-bucket algorithm. Redis is used as the backing store in production; the system fails open (allows the request) if Redis is temporarily unavailable.

Additional per-endpoint limits protect sensitive operations (login: 10/min, partner registration: 5/min).

**HIPAA references:** 45 CFR 164.312(a)(1) — Access control; 45 CFR 164.312(d) — Person or entity authentication.

---

## 4. Tenant Isolation

MedIntelligent is a multi-tenant platform. Every tenant's data is logically isolated at the database level using PostgreSQL Row-Level Security (RLS).

### How RLS Works

1. Each clinical table includes a `tenant_id` column.
2. RLS policies enforce `WHERE tenant_id = current_setting('app.current_tenant_id')` on every SELECT, INSERT, UPDATE, and DELETE.
3. At the start of each API request, the application sets the session context: `SET ROLE app_tenant; SET LOCAL app.current_tenant_id = '<tenant>';`
4. The `app_tenant` database role is a non-superuser role subject to RLS enforcement — it cannot bypass policies.

**Result:** A query from Tenant A can never see, modify, or delete Tenant B's data, regardless of application-layer bugs. The database itself enforces isolation.

### Tenant Suspension

Administrators can suspend a tenant (`POST /admin/tenants/{id}/suspend`), which:
- Blocks all API queries for that tenant
- Flushes cached data for that tenant
- Preserves data for regulatory retention

**HIPAA reference:** 45 CFR 164.312(a)(1) — Access control (unique user identification, automatic logoff).

---

## 5. Audit Logging

MedIntelligent maintains three layers of audit logging:

### 5.1 API Audit Log

Every API request is recorded in the `audit_log` table with:

| Field | Description |
|---|---|
| `event_time` | UTC timestamp |
| `caller_sub` | Authenticated user identity (JWT `sub` claim) |
| `caller_role` | Role at time of request |
| `tenant_id` | Tenant context |
| `http_method` | GET, POST, PATCH, DELETE |
| `api_path` | Endpoint path |
| `response_status` | HTTP status code |
| `duration_ms` | Request processing time |
| `client_ip` | Source IP address |

For query endpoints, additional fields are recorded: `query_id`, `rag_mode`, `patient_id_hash` (never raw patient ID), `question_length` (never raw question text), `processing_ms`, `confidence_score`.

### 5.2 Database Query Audit Log

PostgreSQL `pgaudit` extension logs all DDL, write operations, and role changes at the database level. Logs are exported to CloudWatch for long-term retention. No raw SQL parameters are stored (protects PHI in query parameters).

### 5.3 Application Logs

Structured JSON logs (via `structlog`) include request-scoped context (tenant ID, query ID, RAG mode) and pass through a PHI filter that replaces sensitive fields with `[PHI_REDACTED]` before output.

### Audit Log Access

- Tenant-scoped: `GET /admin/tenants/{id}/audit` — filtered to a single tenant
- Platform-wide: `GET /admin/audit` — admin/service role only
- Both support date range, caller, and status filters
- Audit logs are retained for 10 years, then archived to S3

**HIPAA reference:** 45 CFR 164.312(b) — Audit controls.

---

## 6. PHI Protection

### 6.1 Patient Identifier Hashing

Raw patient identifiers are never stored in audit logs, application logs, or analytics tables. Before any persistence, patient IDs are transformed using HMAC-SHA256 with a per-environment salt, truncated to 16 characters. This one-way transformation satisfies the HIPAA Safe Harbor de-identification standard.

### 6.2 PHI Redaction for LLM Prompts

Before any patient data is sent to a large language model, a defense-in-depth redaction filter scans for and replaces all 18 HIPAA Safe Harbor identifiers:

- Names, geographic identifiers below state level, dates (except year)
- Phone/fax numbers, email addresses, Social Security numbers
- Medical record numbers, insurance policy IDs
- Vehicle identifiers, URLs, IP addresses, biometric identifiers

Redacted values are replaced with consistent placeholders (e.g., `[NAME REDACTED]`).

### 6.3 Log Sanitization

Application logs pass through a PHI filter processor that replaces fields such as `patient_id`, `patient_name`, `ssn`, `question`, `mrn`, and `date_of_birth` with `[PHI_REDACTED]` before any log output. This ensures sensitive data never appears in CloudWatch, stdout, or log aggregation systems.

### 6.4 Consent Management

A `consent_records` table tracks per-patient, per-purpose consent grants and revocations with fields for consent type, scope, method (verbal, written, electronic, implied), granting authority, and document references. Consent status is enforced via `is_active` computed column.

**HIPAA references:** 45 CFR 164.514(b) — Safe Harbor de-identification; 45 CFR 164.508 — Uses and disclosures requiring authorization.

---

## 7. Network Security

### 7.1 VPC Architecture

- **Private subnets** for all compute (ECS Fargate) and data (RDS) resources
- **Public subnets** only for NAT Gateways and the API Gateway VPC Link endpoint
- **VPC Flow Logs** capture all accepted and rejected traffic; delivered to CloudWatch with KMS encryption

### 7.2 Security Groups (Firewall Rules)

Security groups follow the principle of least privilege:

| Resource | Inbound Allowed From | Port |
|---|---|---|
| ALB | VPC CIDR + peered VPCs only | 80 |
| ECS Tasks | ALB security group only | 8000 |
| RDS | ECS task security group + jumpbox only | 5432 |

No security group permits unrestricted inbound access (0.0.0.0/0) on any port.

### 7.3 VPC Endpoints

Gateway VPC Endpoints for S3 and DynamoDB route traffic over the AWS private backbone, avoiding the public internet and NAT Gateway data-processing charges.

### 7.4 Jumpbox

Administrative database access is available only through a jumpbox instance in a private subnet, accessible via AWS Systems Manager Session Manager (no SSH keys, no open ports). All session activity is logged to CloudWatch.

### 7.5 CORS Policy

Cross-Origin Resource Sharing is restricted to explicitly configured origins. The default configuration allows only `localhost` origins for development. Production deployments must set the `CORS_ORIGINS` environment variable to the specific frontend domains.

**HIPAA reference:** 45 CFR 164.312(e)(1) — Transmission security.

---

## 8. Data Retention and Disposal

MedIntelligent maintains a formal Data Retention and Disposal Policy (see `hipaa/data_retention_policy.md`). Key provisions:

| Data Category | Retention Period | Disposal Method |
|---|---|---|
| Clinical records | 6 years minimum (HIPAA) | Soft-delete, then hard-delete after retention |
| Imaging metadata | 7 years (ACR/CMS) | Soft-delete |
| Audit logs | 10 years (OCR guidance) | Archive to S3 (encrypted NDJSON), then delete from live DB |
| Consent records | 6 years from last activity | Soft-delete |

### Soft-Delete

Clinical records are soft-deleted (`is_deleted = TRUE, deleted_at = NOW()`) rather than permanently removed. Soft-deleted records remain encrypted, subject to RLS, and available for regulatory inspection — but are excluded from all API responses and FHIR search results.

### Automated Enforcement

A retention enforcement tool runs daily (as an AWS Lambda or ECS scheduled task) and processes each table according to its configured policy. It defaults to dry-run mode (report only) and requires explicit opt-in for live enforcement.

### Tenant Termination

Upon subscription termination, tenants may request data return (FHIR R4 Bundle export) or destruction (with written certification) within 30 days, per the BAA.

**HIPAA reference:** 45 CFR 164.530(j) — Documentation and retention.

---

## 9. Incident Response

MedIntelligent maintains a formal Incident Response Plan (see `hipaa/incident_response_plan.md`) with seven phases:

1. **Detection and Triage** — alerts from SLA monitoring, audit log anomalies, AWS GuardDuty, CloudTrail, and external reports
2. **Containment** — API key revocation, tenant suspension, credential rotation, VPC isolation
3. **Investigation** — audit log review, CloudTrail analysis, PHI scope determination
4. **Breach Risk Assessment** — four-factor test per 45 CFR 164.402
5. **Notification** — CE notification within 72 hours; HHS notification within 60 days; individual and media notification as required
6. **Remediation and Recovery** — patching, credential rotation, compliance re-verification
7. **Post-Incident Review** — blameless review within 14 days; findings retained for 6 years

### Severity Classification

| Severity | Response Time | Example |
|---|---|---|
| Sev 1 (Critical) | 15-minute acknowledgment | Confirmed multi-tenant PHI breach |
| Sev 2 (High) | 30-minute acknowledgment | Single-tenant unauthorized PHI access |
| Sev 3 (Medium) | 2-hour acknowledgment | Failed brute-force attack; SLA breach |
| Sev 4 (Low) | Next business day | Port scan; disclosed vulnerability |

### Annual Testing

The incident response plan is tested annually through tabletop exercises, technical drills (key revocation, tenant suspension), and notification process dry-runs.

**HIPAA references:** 45 CFR 164.308(a)(6) — Security incident procedures; 45 CFR 164.402–414 — Breach notification rule.

---

## 10. Business Continuity

### 10.1 Database Backups

- **Automated backups:** RDS point-in-time recovery with a configurable retention period (minimum 7 days)
- **Snapshot encryption:** All backups encrypted with the same KMS key as the source instance
- **Deletion protection:** Enabled on production RDS instances to prevent accidental termination

### 10.2 High Availability

- **Multi-AZ deployment:** RDS supports Multi-AZ for automatic failover (configurable per environment)
- **ECS Fargate:** Tasks distributed across multiple availability zones
- **NAT Gateway:** Supports single or multi-AZ deployment

### 10.3 Health Monitoring

The `/health` endpoint reports real-time status of all critical components (vector store, deep thinking RAG, RAG+CAG system, knowledge graph, MCP client, database). The `/alerts` endpoint evaluates live metrics against SLA thresholds and returns actionable runbooks for on-call engineers.

Prometheus-compatible gauges enable integration with existing monitoring infrastructure (Grafana, Alertmanager, Datadog).

**HIPAA reference:** 45 CFR 164.308(a)(7) — Contingency plan.

---

## 11. Vulnerability Management

### 11.1 Container Security

- **Base images:** Official Python slim images, rebuilt on every deployment
- **Image scanning:** Trivy vulnerability scanner runs in CI/CD on every push to main; builds fail on critical/high vulnerabilities
- **ECR image scanning:** AWS ECR native scanning enabled on push

### 11.2 Dependency Management

- **Python dependencies:** Pinned versions in `requirements.txt`; reviewed on update
- **Terraform providers:** Version-locked in `terraform.lock.hcl`

### 11.3 CI/CD Pipeline Security

- **OIDC authentication:** GitHub Actions authenticate to AWS via OpenID Connect (no long-lived credentials)
- **Least-privilege IAM:** Deploy role scoped to specific ECR, ECS, and Terraform resources
- **No secrets in code:** All sensitive values stored in AWS Secrets Manager or GitHub Secrets

### 11.4 SQL Injection Prevention

- The `/sql/validate` endpoint blocks dangerous SQL keywords (DROP, DELETE, TRUNCATE, ALTER, CREATE) before execution
- All database queries use parameterized statements (psycopg2 `%s` placeholders)
- LLM-generated SQL is validated and sanitized before execution

---

## 12. Subprocessors

MedIntelligent uses the following subprocessors that may process or transmit data:

| Subprocessor | Purpose | PHI Access | Location | BAA/DPA |
|---|---|---|---|---|
| Amazon Web Services | Cloud infrastructure (ECS, RDS, S3, KMS, CloudWatch) | Yes — stores and processes ePHI | US-East-1 | AWS BAA available |
| Stripe, Inc. | Payment processing and subscription management | No — receives only tenant ID and token counts | US | Stripe DPA available |
| LLM Providers (configurable) | Natural language processing for clinical queries | Limited — receives redacted clinical context | Configurable | Per-provider agreement required |

Customers are notified of subprocessor changes per the BAA. LLM providers are configurable — Enterprise customers may use self-hosted models to eliminate external PHI transmission entirely.

---

## 13. Compliance Validation

### 13.1 Automated Compliance Tools

MedIntelligent includes built-in compliance validation tools:

| Tool | Purpose | Command |
|---|---|---|
| HIPAA Gap Assessment | Evaluates all Security Rule safeguards; reports IMPLEMENTED/PARTIAL/MISSING with remediation steps | `python generate_hipaa_gap_report.py` |
| RDS Security Checker | Validates 12 technical controls on the production database (encryption, TLS, backups, audit, network isolation) | `python check_rds_security.py` |
| SLA Alert Evaluator | Real-time evaluation of error rate, latency, hallucination rate, and health status against SLA thresholds | `GET /alerts` |

### 13.2 HIPAA Security Rule Coverage

| Safeguard Category | Citation | Status |
|---|---|---|
| Risk Analysis | 164.308(a)(1)(ii)(A) | Implemented (gap assessment tool) |
| Workforce Security | 164.308(a)(3) | Implemented (RBAC, RLS) |
| Information Access Management | 164.308(a)(4) | Implemented (role-based authorization) |
| Security Awareness Training | 164.308(a)(5) | Customer responsibility |
| Security Incident Procedures | 164.308(a)(6) | Implemented (incident response plan) |
| Contingency Plan | 164.308(a)(7) | Implemented (RDS backups, Multi-AZ) |
| Facility Access Controls | 164.310(a) | Inherited from AWS (SOC 2, ISO 27001) |
| Workstation Security | 164.310(b)-(c) | Customer responsibility |
| Access Control | 164.312(a) | Implemented (JWT, API keys, RBAC, RLS) |
| Audit Controls | 164.312(b) | Implemented (3-layer audit logging) |
| Integrity Controls | 164.312(c) | Implemented (HMAC authentication, checksums) |
| Person Authentication | 164.312(d) | Implemented (bcrypt, lockout, MFA-ready) |
| Transmission Security | 164.312(e) | Implemented (TLS 1.2+, forced SSL) |

---

## 14. Customer Responsibilities

Under the shared responsibility model, customers (Covered Entities) are responsible for:

| Area | Customer Responsibility |
|---|---|
| **BAA execution** | Sign the Business Associate Agreement before transmitting PHI |
| **User management** | Create and deactivate user accounts; assign appropriate roles |
| **Credential security** | Store API keys and JWT secrets securely; do not embed in client-side code |
| **Workforce training** | Security awareness training for staff using the platform |
| **Workstation security** | Endpoint protection on devices accessing the platform |
| **Minimum necessary** | Submit only the PHI necessary for the intended query purpose |
| **Breach notification** | Notify affected individuals (MedIntelligent provides data and templates) |
| **Notice of Privacy Practices** | Maintain and communicate NPP to patients |
| **State-specific requirements** | Identify state retention or notification laws that exceed HIPAA minimums |

---

## 15. Contact

For security questions, vulnerability reports, or BAA requests:

| Purpose | Contact |
|---|---|
| Security inquiries | security@medintelligent.ai |
| Privacy Officer | privacy@medintelligent.ai |
| BAA requests | legal@medintelligent.ai |
| Vulnerability disclosure | security@medintelligent.ai (or responsible disclosure program) |
| General support | support@medintelligent.ai |

---

## Document History

| Version | Date | Change |
|---|---|---|
| 1.0 | May 2026 | Initial publication |

---

*This document is provided for informational purposes and describes MedIntelligent's current security practices. It does not constitute a legal guarantee or warranty. Security practices are subject to continuous improvement. Consult legal counsel for compliance questions specific to your organization. Last reviewed May 2026.*

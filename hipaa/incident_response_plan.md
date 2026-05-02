# SECURITY INCIDENT RESPONSE PLAN

**Policy Owner:** MedIntelligent, Inc.
**Effective Date:** _______________
**Last Reviewed:** May 2026
**Review Cadence:** Annual (and after every Severity 1 or 2 incident)
**Regulatory Basis:** 45 CFR 164.308(a)(6), 45 CFR 164.402–164.414

---

## 1. PURPOSE

This plan establishes procedures for identifying, containing, investigating, and reporting security incidents — including breaches of Protected Health Information (PHI) — in compliance with the HIPAA Security Rule and Breach Notification Rule. It ensures MedIntelligent meets its obligations under its Business Associate Agreement (BAA), including the 72-hour security incident notification and 60-day breach notification commitments.

---

## 2. SCOPE

This plan applies to all systems, data, and personnel involved in the MedIntelligent platform, including:

- Production infrastructure (AWS ECS, RDS, S3, CloudWatch)
- Application code and APIs
- Third-party integrations (Stripe, LLM providers, telemedicine partners)
- All employees, contractors, and subprocessors with access to ePHI

---

## 3. DEFINITIONS

| Term | Definition |
|---|---|
| **Security Incident** | The attempted or successful unauthorized access, use, disclosure, modification, or destruction of information or interference with system operations (45 CFR 164.304) |
| **Breach** | The acquisition, access, use, or disclosure of unsecured PHI in a manner not permitted by the Privacy Rule that compromises the security or privacy of the PHI (45 CFR 164.402) |
| **Unsecured PHI** | PHI that is not rendered unusable, unreadable, or indecipherable to unauthorized persons through encryption or destruction per HHS guidance |
| **Covered Entity (CE)** | The healthcare organization (clinic/hospital) whose PHI is affected |

---

## 4. ROLES AND RESPONSIBILITIES

### 4.1 Incident Response Team

| Role | Responsibility | Contact |
|---|---|---|
| **Security Officer** | Owns this plan; leads incident response; final severity classification; coordinates breach notification | *(name and contact on file)* |
| **Privacy Officer** | Advises on PHI scope; drafts individual notifications; liaises with legal counsel | *(name and contact on file)* |
| **Engineering Lead** | Technical investigation and containment; forensic evidence preservation; root-cause analysis | *(name and contact on file)* |
| **On-Call Engineer** | First responder for production alerts; initial triage and severity assessment | PagerDuty / on-call rotation |
| **Legal Counsel** | Breach risk assessment; regulatory notification guidance; litigation hold | *(external counsel on file)* |
| **Executive Sponsor** | Authorizes public communications; budget for remediation; HHS notification sign-off | *(name on file)* |

### 4.2 Designation

The Security Officer is designated as the Incident Commander for all Severity 1 and 2 incidents. For Severity 3 and 4, the On-Call Engineer serves as Incident Commander with escalation authority.

---

## 5. SEVERITY CLASSIFICATION

| Severity | Definition | Examples | Response Time |
|---|---|---|---|
| **Sev 1 — Critical** | Confirmed breach of unsecured PHI affecting multiple individuals or tenants | Unauthorized data exfiltration; database compromise; leaked credentials with PHI access | 15 minutes to acknowledge; immediate containment |
| **Sev 2 — High** | Confirmed or probable security incident involving PHI; single tenant affected | Unauthorized API access to patient data; compromised tenant API key; insider access violation | 30 minutes to acknowledge; contain within 2 hours |
| **Sev 3 — Medium** | Security incident with no confirmed PHI exposure | Failed brute-force login attempts; SLA threshold breach; suspicious API traffic pattern; misconfigured access controls discovered | 2 hours to acknowledge; investigate within 24 hours |
| **Sev 4 — Low** | Minor security event; no PHI risk | Port scan detected; spam login attempts blocked; dependency vulnerability disclosed (no exploit) | Next business day |

---

## 6. INCIDENT RESPONSE PHASES

### Phase 1: Detection and Triage (0–30 minutes)

**Detection sources:**

| Source | Mechanism | Alert Channel |
|---|---|---|
| SLA alerts | `GET /alerts` endpoint; Prometheus Alertmanager rules (`observability/alert_rules.yml`) | PagerDuty / Slack |
| Health check | `GET /health` component monitoring | PagerDuty |
| Audit log anomaly | Unusual access patterns in `audit_log` table (`GET /admin/audit`) | CloudWatch Alarm |
| Login failures | `LoginAttemptTracker` lockout events (`auth.py`) | Application logs |
| AWS GuardDuty | Threat detection on AWS account | SNS → PagerDuty |
| CloudTrail | API-level activity on AWS resources | CloudWatch Logs |
| External report | Customer, researcher, or partner reports a vulnerability | Security email / support |

**Triage steps:**

1. On-call engineer receives alert and acknowledges within the target response time
2. Assess severity using the classification table above
3. If Sev 1 or 2: immediately page the Security Officer
4. Create an incident record (see Section 10) with:
   - Incident ID (format: `INC-YYYY-NNN`)
   - Detection time (UTC)
   - Detection source
   - Initial severity
   - Affected systems and tenants (if known)
5. Open a dedicated incident channel (Slack `#inc-YYYY-NNN` or equivalent)

---

### Phase 2: Containment (30 minutes – 4 hours)

**Immediate containment (Sev 1/2):**

| Scenario | Containment Action |
|---|---|
| Compromised API key | Revoke key immediately: `DELETE /tenants/{id}/keys/{key_id}` |
| Compromised user credentials | Lock account via `LoginAttemptTracker`; revoke all active JWTs by rotating `JWT_SECRET_KEY` |
| Compromised tenant | Suspend tenant: `POST /admin/tenants/{id}/suspend` (flushes caches, blocks queries) |
| Unauthorized database access | Rotate RDS credentials via Secrets Manager; revoke `app_tenant` role sessions |
| Infrastructure compromise | Isolate affected ECS tasks (modify security group to deny all inbound); snapshot ECS task for forensics |
| Data exfiltration in progress | Block egress at VPC level; snapshot RDS for forensic preservation |

**Evidence preservation:**

- Snapshot the affected RDS instance before any remediation
- Export CloudWatch logs for the incident window to S3 (dedicated forensics bucket)
- Export audit log records: `GET /admin/tenants/{tenant_id}/audit?start_date=<incident_start>`
- Preserve CloudTrail events for the affected time window
- Do NOT restart or redeploy services until forensic data is captured

---

### Phase 3: Investigation (4–72 hours)

**Investigation checklist:**

- [ ] Identify the attack vector (how did unauthorized access occur?)
- [ ] Determine the scope of PHI affected:
  - Which tenants?
  - Which patients (by `patient_id_hash`)?
  - Which data elements (diagnoses, medications, demographics)?
  - Date range of exposure
- [ ] Review audit logs (`audit_log` table) for the affected period
- [ ] Review database query audit logs (`query_audit_log` table)
- [ ] Review CloudTrail for AWS API calls (IAM, RDS, S3, Secrets Manager)
- [ ] Review application logs in CloudWatch for error patterns
- [ ] Check if PHI was encrypted at the time of the incident (determines if this is a Breach of *unsecured* PHI)
- [ ] Determine whether the HIPAA Breach Exception applies:
  - (a) Unintentional acquisition by authorized person, acting in good faith
  - (b) Inadvertent disclosure between authorized persons at the same organization
  - (c) Good-faith belief that unauthorized person could not retain the information

**PHI encryption assessment:**

MedIntelligent stores PHI with the following protections. If the compromised data was subject to ALL of the following, it may qualify as *secured* PHI (not subject to breach notification):

| Layer | Control | Reference |
|---|---|---|
| Database storage | AES-256 via RDS KMS encryption | `infra/terraform/modules/rds_postgres/main.tf` |
| Patient identifiers | HMAC-SHA256 hashed before storage | `phi_utils.py:hash_patient_id()` |
| Sensitive fields | Fernet (AES-128-CBC + HMAC) encryption | `phi_utils.py:FieldEncryptor` |
| Transmission | TLS 1.2+ enforced | RDS `rds.force_ssl=1` |
| Backups | Encrypted with source KMS key | AWS RDS default behavior |

---

### Phase 4: Breach Risk Assessment (24–72 hours)

If the investigation reveals potential PHI exposure, the Security Officer and Privacy Officer must conduct a four-factor risk assessment per 45 CFR 164.402:

| Factor | Assessment Question |
|---|---|
| **1. Nature and extent of PHI** | What types of identifiers and clinical data were involved? (Names, diagnoses, SSNs, etc.) |
| **2. Unauthorized person** | Who accessed the PHI? Was it an authorized workforce member acting outside scope, or an external attacker? |
| **3. Whether PHI was actually acquired or viewed** | Is there evidence the data was accessed, downloaded, or merely exposed? (Check audit logs, network logs) |
| **4. Mitigation** | What steps were taken to reduce harm? (Key revocation, account suspension, data not retained by recipient) |

**Decision:** If the risk assessment concludes there is greater than a *low probability* that PHI was compromised, it is a **reportable Breach** and Phase 5 applies.

---

### Phase 5: Notification (within regulatory timelines)

#### 5.1 Covered Entity Notification

**Timeline:** Within **72 hours** of discovery (per BAA Section 2.4(a))

**Method:** Encrypted email to the CE's Privacy Officer on file, followed by a written incident report containing:

- Date of discovery
- Date(s) the incident occurred
- Description of the incident
- Types of PHI involved
- Number of individuals affected (if known)
- Containment steps taken
- Recommended protective actions for affected individuals

#### 5.2 Individual Notification (CE responsibility, BA supports)

**Timeline:** Within **60 calendar days** of discovery (45 CFR 164.404)

MedIntelligent will assist the Covered Entity with:

- Identifying affected individuals (by `patient_id_hash` lookup with the CE's identifier mapping)
- Drafting notification letters
- Providing technical details for the notification

#### 5.3 HHS Notification

| Breach Size | Notification Timeline | Method |
|---|---|---|
| 500+ individuals | Within 60 days of discovery | HHS Breach Reporting Tool: https://ocrportal.hhs.gov/ocr/breach/wizard_breach.jsf |
| Under 500 individuals | Within 60 days of the end of the calendar year in which the breach was discovered | Annual HHS breach log submission |

#### 5.4 State Attorney General Notification

If the breach affects 500+ residents of a single state, notify the state Attorney General per applicable state breach notification laws. Legal counsel coordinates this notification.

#### 5.5 Media Notification

If the breach affects 500+ individuals in a single state or jurisdiction, prominent media notification is required per 45 CFR 164.406. The Executive Sponsor and legal counsel coordinate media communications.

---

### Phase 6: Remediation and Recovery (1–30 days)

**Remediation steps:**

- [ ] Patch the vulnerability or close the attack vector
- [ ] Rotate all potentially compromised credentials (JWT secret, API keys, DB passwords, KMS keys)
- [ ] Restore affected systems from verified clean backups if necessary
- [ ] Reactivate suspended tenants after verification
- [ ] Deploy additional monitoring for the affected attack vector
- [ ] Update security controls (firewall rules, IAM policies, RLS policies) as needed

**Recovery verification:**

- [ ] Run `check_rds_security.py` — all checks must PASS
- [ ] Run `generate_hipaa_gap_report.py` — no new MISSING or FAIL findings
- [ ] Verify `GET /health` returns all components `available`
- [ ] Verify `GET /alerts` returns `status: green`
- [ ] Confirm audit logging is operational (submit test request, verify audit entry)

---

### Phase 7: Post-Incident Review (within 14 days)

Conduct a blameless post-incident review (PIR) within 14 days of incident closure. Document:

1. **Timeline** — minute-by-minute reconstruction of events
2. **Root cause** — technical and process failures that allowed the incident
3. **Detection gap** — how long between the incident occurring and detection; how to shorten this
4. **Response effectiveness** — what went well, what slowed the response
5. **Action items** — specific, assigned, time-bound remediation tasks
6. **Plan updates** — changes to this Incident Response Plan based on lessons learned

The PIR document is retained for 6 years per HIPAA documentation requirements.

---

## 7. COMMUNICATION TEMPLATES

### 7.1 Internal Escalation (Sev 1/2)

```
SECURITY INCIDENT — [Severity Level]
Incident ID: INC-YYYY-NNN
Detected: [UTC timestamp]
Source: [detection source]

Summary: [1-2 sentence description]

Affected: [tenants/systems/data types]

Current status: [Triage / Containing / Investigating]

Incident Commander: [name]
Channel: #inc-YYYY-NNN

Next update in [30/60] minutes.
```

### 7.2 Covered Entity Notification

```
Subject: Security Incident Notification — MedIntelligent [INC-YYYY-NNN]

Dear [CE Privacy Officer],

We are writing to notify you of a security incident that may affect
Protected Health Information (PHI) processed by MedIntelligent on
behalf of [CE Name].

Date of Discovery: [date]
Date(s) of Incident: [date range]

Description: [what happened, how it was detected, what data types
were involved]

Individuals Potentially Affected: [count or "under investigation"]

Actions Taken:
- [containment steps]
- [remediation steps]

Recommended Actions for Your Organization:
- [protective steps for affected individuals]

We will provide an updated report within [timeframe]. For questions,
contact our Security Officer at [contact].

This notification is provided pursuant to Section 2.4 of our
Business Associate Agreement.

Sincerely,
[Security Officer Name]
MedIntelligent, Inc.
```

---

## 8. ANNUAL TESTING

This plan must be tested at least annually through one of the following:

| Exercise Type | Frequency | Description |
|---|---|---|
| **Tabletop exercise** | Annual (minimum) | Walk through a simulated Sev 1 breach scenario with the full IRT; validate communication chains, decision points, and notification timelines |
| **Technical drill** | Annual | Simulate a compromised API key or tenant; verify containment steps (key revocation, tenant suspension) work within target timelines |
| **Notification drill** | Annual | Dry-run the CE notification process end-to-end; verify contact information is current |

**Test results** are documented and retained for 6 years. Findings are incorporated into the next plan revision.

---

## 9. REGULATORY REFERENCE

| Citation | Requirement | How This Plan Addresses It |
|---|---|---|
| 45 CFR 164.308(a)(6)(i) | Security incident procedures | Sections 5 and 6 (detection through remediation) |
| 45 CFR 164.308(a)(6)(ii) | Response and reporting | Section 6 Phase 5 (notification procedures) |
| 45 CFR 164.402 | Breach definition and exceptions | Section 6 Phase 3 (encryption assessment) and Phase 4 (risk assessment) |
| 45 CFR 164.404 | Notification to individuals | Section 6 Phase 5.2 (60-day timeline) |
| 45 CFR 164.406 | Notification to media | Section 6 Phase 5.5 (500+ threshold) |
| 45 CFR 164.408 | Notification to HHS | Section 6 Phase 5.3 (HHS breach portal) |
| 45 CFR 164.410 | BA notification to CE | Section 6 Phase 5.1 (72-hour timeline) |
| 45 CFR 164.414 | Administrative requirements for breach notifications | Section 7 (templates and documentation) |
| 45 CFR 164.530(j) | Documentation retention | Section 6 Phase 7 (6-year PIR retention) |

---

## 10. INCIDENT LOG

All incidents are tracked in a register with the following fields:

| Field | Description |
|---|---|
| Incident ID | `INC-YYYY-NNN` |
| Detection time | UTC timestamp |
| Detection source | Alert, report, audit, etc. |
| Severity | 1–4 |
| Description | Brief summary |
| Affected tenants | Tenant IDs or "platform-wide" |
| PHI involved | Yes / No / Under investigation |
| Breach determination | Breach / Not a breach / Low probability (with rationale) |
| Containment time | Time from detection to containment |
| Resolution time | Time from detection to full resolution |
| CE notified | Date or N/A |
| HHS notified | Date or N/A |
| PIR completed | Date |
| Action items | Link to tracking system |

The incident register is retained for 6 years and is available for HHS inspection per 45 CFR 164.530(j).

---

## 11. RELATED DOCUMENTS

| Document | Location |
|---|---|
| Business Associate Agreement | `hipaa/baa_template.md` |
| Data Retention Policy | `hipaa/data_retention_policy.md` |
| HIPAA Gap Assessment Tool | `generate_hipaa_gap_report.py` |
| RDS Security Checker | `check_rds_security.py` |
| SLA Alert Rules | `observability/alert_rules.yml` |
| SLA Alert Evaluator | `observability/alerts.py` |
| Audit Logging Schema | `hipaa_db_migration.py` |
| PHI Utilities | `phi_utils.py` |

---

## 12. APPROVAL

| Role | Name | Signature | Date |
|---|---|---|---|
| Security Officer | _________________ | _________________ | _________ |
| Privacy Officer | _________________ | _________________ | _________ |
| Executive Sponsor | _________________ | _________________ | _________ |

---

*This plan is provided for informational purposes and does not constitute legal advice. Consult legal counsel and qualified security professionals to ensure compliance with all applicable regulations. Last reviewed May 2026.*

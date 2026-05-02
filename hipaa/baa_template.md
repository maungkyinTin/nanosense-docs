# BUSINESS ASSOCIATE AGREEMENT

**Effective Date:** _______________

**By and Between:**

**Covered Entity:** _______________________________________________
Address: _______________________________________________________
("CE")

**Business Associate:** MedIntelligent, Inc.
Address: _______________________________________________________
("BA")

---

## RECITALS

WHEREAS, BA provides Clinical Decision Support (CDS) and Medical AI services to CE under a separate Order Form or Master Services Agreement (the "Underlying Agreement"); and

WHEREAS, in the course of providing those services, BA may create, receive, maintain, or transmit Protected Health Information ("PHI") on behalf of CE; and

WHEREAS, the Health Insurance Portability and Accountability Act of 1996 ("HIPAA"), the Health Information Technology for Economic and Clinical Health Act ("HITECH"), and the regulations promulgated thereunder (collectively, "HIPAA Rules") require CE and BA to enter into this Agreement;

NOW, THEREFORE, in consideration of the mutual covenants and agreements herein, and for other good and valuable consideration, the parties agree as follows:

---

## 1. DEFINITIONS

Terms used but not defined herein have the meaning ascribed to them under the HIPAA Rules (45 C.F.R. Parts 160 and 164).

1.1 **"Breach"** — as defined at 45 C.F.R. § 164.402.

1.2 **"Business Associate"** — BA as defined at 45 C.F.R. § 160.103.

1.3 **"Designated Record Set"** — as defined at 45 C.F.R. § 164.501.

1.4 **"Electronic Protected Health Information (ePHI)"** — PHI transmitted or maintained in electronic media.

1.5 **"Protected Health Information (PHI)"** — as defined at 45 C.F.R. § 160.103, limited to information BA creates, receives, maintains, or transmits on behalf of CE.

1.6 **"Required by Law"** — as defined at 45 C.F.R. § 164.103.

1.7 **"Security Incident"** — as defined at 45 C.F.R. § 164.304.

1.8 **"Unsecured PHI"** — as defined at 45 C.F.R. § 164.402.

---

## 2. OBLIGATIONS OF BUSINESS ASSOCIATE

### 2.1 Permitted Uses and Disclosures

BA may use or disclose PHI only to:

(a) Perform the services described in the Underlying Agreement;

(b) As Required by Law;

(c) For BA's proper management and administration, provided that disclosures are either Required by Law or BA obtains reasonable assurances from the recipient that PHI will remain confidential and used or further disclosed only as Required by Law or for the purpose for which it was disclosed.

### 2.2 Safeguards

BA shall implement administrative, physical, and technical safeguards that reasonably and appropriately protect the confidentiality, integrity, and availability of ePHI as required by the HIPAA Security Rule (45 C.F.R. Part 164, Subpart C), including:

(a) **Encryption at Rest** — All ePHI stored by BA (databases, object storage, backups) shall be encrypted using AES-256 or equivalent. MedIntelligent uses AWS RDS with storage encryption (AES-256) and KMS-managed keys.

(b) **Encryption in Transit** — All ePHI transmitted between BA systems and CE or third parties shall use TLS 1.2 or higher.

(c) **Access Controls** — BA shall maintain Role-Level Access Control and Row-Level Security (RLS) policies ensuring each tenant's ePHI is logically isolated.

(d) **Audit Logging** — BA shall maintain audit logs of all access to and disclosures of ePHI for a minimum of six (6) years.

(e) **Minimum Necessary** — BA shall limit its use and disclosure of PHI to the minimum necessary to accomplish the intended purpose.

### 2.3 Subcontractors

BA shall ensure that any subcontractor or agent that creates, receives, maintains, or transmits PHI on behalf of BA agrees to the same restrictions and conditions that apply to BA under this Agreement, in accordance with 45 C.F.R. § 164.308(b)(2).

Key subprocessors used by MedIntelligent:
| Subprocessor | Purpose | Location |
|---|---|---|
| Amazon Web Services (AWS) | Cloud infrastructure, RDS, ECS | US East 1 |
| Stripe, Inc. | Payment processing (no PHI) | US |

### 2.4 Reporting

(a) **Security Incidents** — BA shall report to CE any Security Incident of which BA becomes aware without unreasonable delay, and in no event later than **72 hours** after discovery.

(b) **Breach of Unsecured PHI** — BA shall notify CE of a Breach of Unsecured PHI without unreasonable delay, and in no event later than **60 calendar days** of discovery, as required by 45 C.F.R. § 164.410.

Notification shall be delivered to CE's Privacy Officer at the email address on file.

### 2.5 Individual Rights

BA shall, upon CE's written request and within **30 days**:

(a) Make PHI available in a Designated Record Set to CE for access by the individual as required by 45 C.F.R. § 164.524;

(b) Make PHI available for amendment and incorporate any amendments as directed by CE per 45 C.F.R. § 164.526;

(c) Provide an accounting of disclosures per 45 C.F.R. § 164.528.

### 2.6 Return or Destruction of PHI

Upon termination of the Underlying Agreement, BA shall, at CE's election: (a) return all PHI to CE; or (b) destroy all PHI and certify such destruction in writing, within **30 days** of termination. If return or destruction is infeasible, BA shall extend the protections of this Agreement to such PHI and limit further use to those purposes that make return or destruction infeasible.

### 2.7 HHS Inspection

BA shall make its internal practices, books, and records relating to the use and disclosure of PHI available to the Secretary of HHS for the purpose of determining compliance with the HIPAA Rules, in accordance with 45 C.F.R. § 164.504(e)(2)(ii)(I).

---

## 3. OBLIGATIONS OF COVERED ENTITY

CE shall:

(a) Notify BA of any limitation(s) in CE's Notice of Privacy Practices that may affect BA's use or disclosure of PHI;

(b) Notify BA of any changes in, or revocation of, an individual's authorization;

(c) Not request BA to use or disclose PHI in any manner that would violate the HIPAA Rules.

---

## 4. ON-PREMISES VPC DEPLOYMENT OPTION

CE may elect to deploy the MedIntelligent platform within CE's own AWS Virtual Private Cloud ("CE VPC") under a separate Order Form. Under this option:

(a) **Data Residency** — All ePHI remains within CE's AWS account and region of choice;

(b) **Network Isolation** — The deployment uses a private subnet with no public internet egress for database traffic;

(c) **Credential Management** — CE controls all AWS IAM roles, KMS keys, and RDS credentials;

(d) **BA Access** — BA's access to CE's environment is limited to read-only CloudWatch Logs for support purposes, and only with CE's prior written authorization per incident.

On-prem VPC deployment artifacts are provided as Docker Compose and Terraform modules in the `deploy/onprem/` directory of the licensed software package.

---

## 5. TERM AND TERMINATION

5.1 This Agreement is effective as of the Effective Date and remains in effect until the Underlying Agreement is terminated.

5.2 Either party may immediately terminate this Agreement if it determines the other party has materially breached any provision and fails to cure within **30 days** of written notice.

5.3 CE may terminate this Agreement if BA violates a material term of this Agreement and has not cured the violation within the **30-day** notice period.

---

## 6. GENERAL PROVISIONS

6.1 **Entire Agreement.** This Agreement, together with the Underlying Agreement, constitutes the entire agreement between the parties regarding the subject matter hereof.

6.2 **Amendment.** This Agreement may be amended only by a written instrument signed by authorized representatives of both parties. BA may amend this Agreement unilaterally, upon 60 days' written notice, to the extent necessary to comply with changes in the HIPAA Rules.

6.3 **Severability.** Any provision of this Agreement that is determined to be invalid or unenforceable shall be deemed deleted, and the remaining provisions shall continue in full force and effect.

6.4 **Governing Law.** This Agreement shall be governed by and construed under the laws of the State of Delaware, without regard to conflict of laws principles.

6.5 **No Third-Party Beneficiaries.** This Agreement is for the sole benefit of the parties and their permitted successors and assigns. Nothing herein shall create or be deemed to create any rights in any third party.

6.6 **Order of Precedence.** In the event of a conflict between this Agreement and the Underlying Agreement with respect to PHI, this Agreement shall control.

---

## 7. SIGNATURES

**COVERED ENTITY**

Signature: _________________________________ Date: ___________

Name: _____________________________________

Title: ______________________________________

Organization: ______________________________

**BUSINESS ASSOCIATE — MedIntelligent, Inc.**

Signature: _________________________________ Date: ___________

Name: _____________________________________

Title: ______________________________________

---

*This template is provided for informational purposes. Consult legal counsel before executing. Last reviewed against HIPAA Rules as amended through January 2025.*

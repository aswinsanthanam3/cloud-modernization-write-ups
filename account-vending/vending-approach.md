# Account Vending Automation — Project Plan

**Document Type:** Project Plan  
**Author:** Platform Engineering  
**Created:** 2026-03-15  
**Status:** Draft  
**Timeline:** 8 Sprints (16 weeks)  
**Team Size:** 4–5 Engineers  

---

## 1. Executive Summary

This project plan defines a sprint-by-sprint approach to building end-to-end account vending automation using AWS Control Tower Account Factory for Terraform (AFT). The target state replaces a manual, 2–4 week process with a self-service pipeline that provisions fully compliant AWS accounts within hours.

### 1.1 Current State Pain Points

| Problem | Impact |
|---------|--------|
| 2–4 week account delivery time | Business needs accounts in < 1 day |
| Tribal knowledge, no documentation | Bus factor risk, onboarding friction |
| 75+ StackSets with no IaC, inconsistent deployment (46–75 per account) | Drift, compliance gaps, audit liability |
| Manual SSO group and IAM role provisioning | Bottleneck on identity team availability |
| Manual security validation per account | Blocks workload deployment, non-repeatable |
| No policy-as-code or compliance scorecard | No real-time visibility into account posture |

### 1.2 Target State

- Self-service account request → fully provisioned account in < 4 hours
- 2–3 account archetypes (prod, non-prod, sandbox) with standardized configurations
- AFT-driven provisioning with automated networking, identity, and compliance
- Dual policy-as-code: OPA/Conftest (pre-deploy) + AWS Config Rules (runtime)
- Compliance scorecard per account with automated security certification
- All infrastructure codified in Terraform with no unmanaged StackSets

---

## 2. Architectural Decisions (Pre-Sprint)

The following decisions were made during project scoping and drive the plan structure.

### ADR-AV-001: StackSet Rationalization — Hybrid Approach

**Decision:** Port critical StackSets to Terraform (deployed via AFT customizations), sunset non-critical StackSets over time.

**Rationale:** Reverse-engineering all 75+ StackSets is infeasible within the project timeline. A hybrid approach focuses effort on high-value controls while establishing a deprecation path for the remainder.

**Consequences:**

- Sprint 2 includes a StackSet audit and classification exercise
- Critical StackSets are converted to AFT global/account customizations
- Non-critical StackSets remain operational with a documented sunset plan
- Security architecture team must validate the critical vs. non-critical classification

### ADR-AV-002: Account Archetypes

**Decision:** Support 2–3 account archetypes: production, non-production, and sandbox.

**Rationale:** Standard VPC topology per archetype simplifies network templating. Each archetype maps to a distinct AFT account customization template with appropriate guardrails.

**Consequences:**

- Each archetype gets its own AFT account customization repo/directory
- SCP policies are tiered by archetype (sandbox is more restrictive, prod has break-glass provisions)
- Network topology is fully parameterized per archetype (CIDR allocation, subnet structure, TGW attachment)

### ADR-AV-003: Policy-as-Code — Dual Layer

**Decision:** Implement both OPA/Conftest for pre-deploy Terraform plan validation and AWS Config Rules for runtime compliance.

**Rationale:** Shift-left validation catches misconfigurations before they reach AWS. Runtime Config Rules provide continuous compliance monitoring and feed the scorecard. Neither alone is sufficient.

**Consequences:**

- OPA policies are maintained alongside Terraform modules and executed in CI/CD
- AWS Config conformance packs are deployed via AFT global customizations
- Compliance scorecard aggregates Config rule evaluations per account
- Security architecture team must codify their manual checklist into both OPA policies and Config rules

### ADR-AV-004: Pipeline Target State (PENDING — Sprint 1 Decision)

**Decision:** To be finalized in Sprint 1.

**Options under consideration:**

| Option | Pros | Cons |
|--------|------|------|
| AFT native CodePipeline | Fully integrated, AWS-supported, no external dependencies | Less flexible, team must learn CodePipeline |
| AFT triggers GitLab CI | Team familiarity, existing pipeline investment | Additional integration complexity, dual pipeline maintenance |
| AFT triggers Harness | Advanced deployment strategies, existing investment | Heaviest integration lift, vendor lock-in |

**Recommendation:** Use AFT native CodePipeline for account customizations (networking, identity, baseline controls). Use GitLab/Harness for workload-layer deployments that happen after account vending is complete. This creates a clean separation: AFT owns the account, application teams own the workloads.

**Action:** Team to evaluate and finalize by end of Sprint 1, Week 2.

---

## 3. Sprint Plan

### Sprint 0 — Foundation & Discovery (Weeks 1–2)

**Objective:** Establish project infrastructure, audit current state, and produce documentation that eliminates tribal knowledge.

#### Deliverables

- [ ] **Current-state documentation:** Document every step of the existing account vending process as a runbook (interviews with current operators)
- [ ] **StackSet inventory:** Catalog all 75+ StackSets with metadata — name, purpose, which OUs/accounts they target, last update date, owner (if known)
- [ ] **StackSet classification:** Classify each StackSet as Critical (must port), Redundant (duplicate or overlapping), Obsolete (no longer needed), or Deferred (port later)
- [ ] **AFT health assessment:** Evaluate current AFT deployment — version, configuration, pipeline state, known issues
- [ ] **Repository structure:** Create Git repositories for AFT configuration, account customizations, OPA policies, and compliance scorecard
- [ ] **IPAM integration spike:** Document the current IPAM tool (Infoblox/NetBox) API and determine the integration pattern for automated CIDR allocation
- [ ] **Okta SSO mapping:** Document current SSO group structure, permission sets, and the manual provisioning workflow

#### Acceptance Criteria

- Written runbook of current process with no undocumented steps
- StackSet inventory spreadsheet with classification reviewed by security architecture
- AFT upgrade/remediation plan if current state requires it
- Repository structure created and team access provisioned

#### Risks

| Risk | Mitigation |
|------|------------|
| Tribal knowledge holders unavailable for interviews | Schedule interviews in first 3 days, record sessions |
| StackSet classification disputes with security team | Timebox classification review to 2 hours, escalate unresolved items |

---

### Sprint 1 — AFT Stabilization & Account Archetypes (Weeks 3–4)

**Objective:** Get AFT into a healthy, version-current state. Define and implement account archetype templates.

#### Deliverables

- [ ] **AFT upgrade/cleanup:** Bring AFT to current version, resolve any configuration drift or pipeline failures
- [ ] **Account archetype definitions:** Formalize prod, non-prod, and sandbox archetypes with documented differences in network topology, SCP scope, SSO access, and compliance requirements
- [ ] **AFT account request templates:** Create `account-request` Terraform configurations for each archetype
- [ ] **Global customization baseline:** Implement AFT global customizations for controls that apply to all accounts (CloudTrail, GuardDuty enrollment, default SCPs, etc.)
- [ ] **Pipeline decision (ADR-AV-004):** Finalize pipeline target state with team evaluation and sign-off
- [ ] **CIDR allocation automation:** Build IPAM integration module — Terraform calls IPAM API to reserve next available CIDR block for the archetype's VPC pattern

#### Acceptance Criteria

- AFT successfully provisions a test account with no manual intervention
- All three archetype templates produce valid `account-request` configurations
- Global customizations deploy CloudTrail, GuardDuty, and baseline SCPs to test account
- IPAM integration returns valid CIDR blocks for each archetype's subnet layout
- ADR-AV-004 signed off with documented decision rationale

#### Dependencies

- Security architecture team review of archetype SCP differences
- IPAM tool API access and service account credentials
- AFT management account access for upgrade

---

### Sprint 2 — Networking Automation (Weeks 5–6)

**Objective:** Automate the full network provisioning stack that currently requires separate TF jobs.

#### Deliverables

- [ ] **VPC module per archetype:** Terraform module that creates VPC, subnets (public/private/isolated as defined per archetype), route tables, NACLs, and flow logs
- [ ] **Security group baseline:** Default security groups per archetype with documented ingress/egress rules
- [ ] **VPC endpoints:** Terraform module for standard VPCEs (S3, DynamoDB, SSM, CloudWatch, ECR, STS, etc.) per archetype
- [ ] **Transit Gateway attachment:** Automated TGW attachment with route propagation to shared services and on-premises networks
- [ ] **AFT account customization integration:** Wire networking modules into AFT account customizations so they execute automatically post-vend
- [ ] **DNS configuration:** Route 53 resolver rules, PHZ associations for shared services

#### Acceptance Criteria

- End-to-end test: AFT provisions account → networking customization runs → VPC, subnets, SGs, VPCEs, TGW attachment, and DNS all created with no manual steps
- CIDR blocks allocated from IPAM automatically (no manual reservation)
- TGW route table updated, connectivity to shared services validated
- Network topology matches archetype specification exactly

#### Dependencies

- Network team review of TGW route table changes
- Shared services account owners for PHZ association approval
- Completed IPAM integration from Sprint 1

---

### Sprint 3 — Identity & Access Automation (Weeks 7–8)

**Objective:** Automate SSO group creation, permission set assignment, and baseline IAM roles.

#### Deliverables

- [ ] **Okta integration module:** Terraform (or API-driven automation) to create Okta groups per account archetype and assign them to AWS IAM Identity Center
- [ ] **Permission set templates:** Define and deploy permission sets per archetype — admin, developer, read-only, break-glass — as Terraform resources
- [ ] **SSO group-to-permission-set mapping:** Automate the assignment of Okta groups to permission sets for the vended account
- [ ] **Baseline IAM roles:** Terraform module for cross-account roles required by third-party integrations (monitoring, security scanning, backup, etc.)
- [ ] **Service-linked roles:** Automate creation of AWS service-linked roles needed by baseline services (Config, GuardDuty, SecurityHub, etc.)
- [ ] **AFT integration:** Wire identity modules into AFT account customizations, sequenced after networking

#### Acceptance Criteria

- Vended account automatically gets correct Okta groups and permission sets with no manual tickets
- Break-glass access pattern documented and tested
- Third-party integration roles are functional (validated by integration owners)
- No manual IAM provisioning steps remain

#### Dependencies

- Okta admin API access and service account
- Identity team review of permission set definitions
- Third-party integration owners to validate role trust policies

---

### Sprint 4 — StackSet Migration (Critical Controls) (Weeks 9–10)

**Objective:** Convert critical-classified StackSets to Terraform and deploy via AFT customizations.

#### Deliverables

- [ ] **Critical StackSet conversion:** Reverse-engineer each critical StackSet's CloudFormation into equivalent Terraform — expected 15–25 StackSets based on hybrid classification
- [ ] **State import:** For existing accounts, import current resources into Terraform state to avoid recreating infrastructure
- [ ] **AFT global vs. account customization placement:** Determine which converted controls are global (all accounts) vs. account-specific and place accordingly
- [ ] **StackSet sunset plan:** Document the deprecation timeline and process for each non-critical StackSet, including validation that no active account depends solely on a non-critical StackSet for a compliance control
- [ ] **Drift detection:** Implement Terraform drift detection for converted controls

#### Acceptance Criteria

- All critical StackSets have equivalent Terraform deployed via AFT customizations
- Existing accounts: Terraform state imported, `terraform plan` shows no diff
- New accounts: critical controls deploy automatically via AFT
- StackSet sunset plan reviewed and approved by security architecture

#### Risks

| Risk | Mitigation |
|------|------------|
| CloudFormation-to-Terraform conversion errors | Test each conversion in a sandbox account before production |
| State import conflicts | Run imports in a dedicated maintenance window per account batch |
| Undocumented StackSet dependencies | Cross-reference with AWS Config resource inventory |

---

### Sprint 5 — Policy-as-Code: Shift-Left (Weeks 11–12)

**Objective:** Implement OPA/Conftest pre-deploy validation for all Terraform in the vending pipeline.

#### Deliverables

- [ ] **OPA policy library:** Codify security architecture's manual checklist into Rego policies covering — encryption at rest (EBS, S3, RDS), public access blocks, logging requirements, tagging standards, network security (no 0.0.0.0/0 ingress), IAM constraints
- [ ] **Conftest integration:** Integrate Conftest into the AFT pipeline so every `terraform plan` is validated against OPA policies before `apply`
- [ ] **Policy testing framework:** Unit tests for each OPA policy using Conftest's test runner with positive and negative test cases
- [ ] **Policy violation reporting:** Pipeline output clearly identifies which policies failed, which resources are non-compliant, and the remediation action
- [ ] **Policy exception process:** Define and document the exception workflow — how a team requests a policy waiver, who approves, and how it's tracked

#### Acceptance Criteria

- AFT pipeline rejects a `terraform apply` when OPA policies are violated
- All policies have passing unit tests (positive and negative cases)
- Policy violation output is human-readable with actionable remediation guidance
- Exception process documented and approved by security architecture

#### Dependencies

- Security architecture team's manual checklist (must be complete and prioritized)
- Agreement on policy severity levels (block vs. warn)

---

### Sprint 6 — Policy-as-Code: Runtime & Compliance Scorecard (Weeks 13–14)

**Objective:** Deploy AWS Config conformance packs for runtime compliance and build the account compliance scorecard.

#### Deliverables

- [ ] **AWS Config conformance packs:** Terraform modules deploying Config rules that mirror the OPA policy library — encryption, public access, logging, tagging, network, IAM
- [ ] **Custom Config rules:** Lambda-backed Config rules for organization-specific controls not covered by AWS managed rules
- [ ] **Config aggregator:** Central Config aggregator in the audit/security account for cross-account visibility
- [ ] **Compliance scorecard:** Dashboard (CloudWatch dashboard, QuickSight, or custom) showing per-account compliance status — percentage of Config rules passing, critical violations, trend over time
- [ ] **AFT integration:** Config conformance packs deployed via AFT global customizations to every vended account
- [ ] **Automated certification signal:** Define the Config rule pass rate threshold (e.g., 100% critical, 95% overall) that constitutes automated security certification — replacing the manual validation step

#### Acceptance Criteria

- Every vended account gets Config conformance packs automatically
- Compliance scorecard shows real-time per-account compliance percentage
- Security architecture team agrees on the automated certification threshold
- Accounts meeting the threshold are auto-certified for workload deployment

#### Dependencies

- Security architecture team sign-off on certification threshold
- Central Config aggregator permissions in audit account
- QuickSight/dashboard tooling access if using QuickSight

---

### Sprint 7 — Integration Testing & Self-Service Interface (Weeks 15–16)

**Objective:** End-to-end integration testing across all archetypes. Build the self-service request interface.

#### Deliverables

- [ ] **End-to-end test suite:** Automated tests that provision an account of each archetype and validate every component — networking, identity, controls, compliance, logging
- [ ] **Self-service intake:** Account request interface — this could be a Git-based workflow (PR to `account-request` repo), a ServiceNow catalog item, or a lightweight internal UI — that captures archetype, business justification, cost center, and owner
- [ ] **Approval workflow:** Automated routing of account requests to appropriate approvers (finance for cost center, security for compliance, platform for capacity)
- [ ] **Notification and observability:** SNS/Slack notifications for vending pipeline status — request received, provisioning in progress, completed, failed
- [ ] **Runbook and documentation:** Complete operational runbook covering troubleshooting, rollback procedures, and day-2 operations
- [ ] **Existing account remediation plan:** Strategy for bringing existing accounts (with inconsistent StackSet deployments) into compliance with the new baseline

#### Acceptance Criteria

- All three archetypes pass end-to-end automated tests
- Self-service request → approval → provisioned account completes in < 4 hours
- Pipeline failure triggers alert with actionable error context
- Runbook reviewed and validated by on-call engineers
- Existing account remediation plan scoped and estimated

#### Risks

| Risk | Mitigation |
|------|------------|
| Integration test environment costs | Use sandbox archetype with aggressive auto-cleanup |
| Self-service interface scope creep | MVP: Git-based PR workflow. Iterate to UI later |

---

## 4. Sprint Capacity & Velocity Assumptions

| Parameter | Value |
|-----------|-------|
| Sprint duration | 2 weeks |
| Team size | 4–5 engineers |
| Capacity per sprint | ~160–200 engineer-hours (accounting for meetings, reviews, overhead) |
| Parallel workstreams | 2–3 concurrent tracks where dependencies allow |
| Code review policy | All Terraform requires peer review before merge |

### Recommended Workstream Assignments

| Track | Engineers | Sprints Active |
|-------|-----------|----------------|
| AFT core and account customizations | 1–2 | 0–7 |
| Networking and IPAM integration | 1 | 1–2, 7 |
| Identity and SSO automation | 1 | 3, 7 |
| StackSet audit and migration | 1–2 | 0, 4 |
| Policy-as-code and compliance | 1–2 | 5–6 |
| Integration testing and self-service | 2 | 7 |

---

## 5. Key Milestones

| Milestone | Sprint | Date (Projected) |
|-----------|--------|-------------------|
| Current-state fully documented, StackSet inventory complete | Sprint 0 | Week 2 |
| AFT stable, first automated account vend (no networking/identity) | Sprint 1 | Week 4 |
| Full networking automation integrated into AFT | Sprint 2 | Week 6 |
| Full identity automation, no manual tickets | Sprint 3 | Week 8 |
| Critical StackSets replaced with Terraform | Sprint 4 | Week 10 |
| Pre-deploy policy validation live in pipeline | Sprint 5 | Week 12 |
| Runtime compliance scorecard operational | Sprint 6 | Week 14 |
| Self-service account vending in production | Sprint 7 | Week 16 |

---

## 6. Risk Register

| # | Risk | Likelihood | Impact | Mitigation | Owner |
|---|------|-----------|--------|------------|-------|
| R1 | Tribal knowledge holders leave or are unavailable | Medium | High | Prioritize documentation in Sprint 0; record all interviews | Project Lead |
| R2 | StackSet conversion takes longer than estimated | High | Medium | Hybrid approach limits scope; defer non-critical StackSets | StackSet Track Lead |
| R3 | Okta API limitations block SSO automation | Medium | High | Early spike in Sprint 0; identify fallback (SCIM, manual-with-API) | Identity Track Lead |
| R4 | IPAM integration unreliable or API undocumented | Medium | Medium | Sprint 1 spike with fallback to Terraform IPAM provider | Networking Track Lead |
| R5 | Security team delays in validating classifications/thresholds | High | High | Embed security architect in sprint ceremonies; timebox reviews | Project Lead |
| R6 | AFT upgrade introduces breaking changes | Low | High | Test upgrade in non-prod management account first | AFT Track Lead |
| R7 | Existing account remediation scope overwhelming | Medium | Medium | Defer to post-project phase; focus on new accounts first | Project Lead |

---

## 7. Definition of Done — Account Vending

An account is considered "fully vended" when all of the following are true:

1. AWS account created via AFT with correct OU placement
2. SCPs applied per archetype
3. VPC, subnets, security groups, VPCEs, and TGW attachment provisioned per archetype topology
4. CIDR block allocated from IPAM and recorded
5. DNS resolver rules and PHZ associations configured
6. Okta SSO groups created and mapped to permission sets
7. Baseline IAM roles for third-party integrations provisioned
8. CloudTrail, GuardDuty, SecurityHub, and Config enrolled
9. Logs forwarding to central Splunk architecture validated
10. OPA policies passed during provisioning (shift-left)
11. AWS Config conformance pack deployed and evaluating
12. Compliance scorecard shows account at or above certification threshold
13. Account owner notified with access instructions and runbook links

---

## 8. Out of Scope (This Project)

- Workload-layer deployment automation (application CI/CD pipelines)
- Existing account remediation (planned as follow-on project, scoped in Sprint 7)
- Non-critical StackSet conversion (tracked via sunset plan, executed incrementally post-project)
- Multi-region account vending (single-region baseline; multi-region is a future enhancement)
- Cost optimization automation (tagging standards are in scope; budget alerts and optimization tooling are not)

---

## 9. Appendix

### A. References

- AWS Control Tower Account Factory for Terraform (AFT): https://docs.aws.amazon.com/controltower/latest/userguide/aft-overview.html
- AFT Architecture and Design: https://docs.aws.amazon.com/controltower/latest/userguide/aft-architecture.html
- OPA/Conftest for Terraform: https://www.conftest.dev/
- AWS Config Conformance Packs: https://docs.aws.amazon.com/config/latest/developerguide/conformance-packs.html
- AWS VPC IPAM: https://docs.aws.amazon.com/vpc/latest/ipam/what-is-it-ipam.html
- AWS IAM Identity Center (SSO) with Okta: https://docs.aws.amazon.com/singlesignon/latest/userguide/okta-idp.html
- Terraform AWS Provider: https://registry.terraform.io/providers/hashicorp/aws/latest/docs

### B. Glossary

| Term | Definition |
|------|------------|
| AFT | Account Factory for Terraform — AWS-provided solution for automating account provisioning via Terraform |
| Account Archetype | A standardized account configuration template (e.g., prod, non-prod, sandbox) |
| Global Customization | AFT Terraform that runs against every vended account |
| Account Customization | AFT Terraform that runs for specific accounts or account types |
| SCP | Service Control Policy — organization-level guardrail in AWS Organizations |
| OPA | Open Policy Agent — policy engine for infrastructure-as-code validation |
| Conftest | CLI tool that runs OPA policies against structured data (e.g., Terraform plan JSON) |
| IPAM | IP Address Management — tool for automated CIDR block allocation |
| TGW | Transit Gateway — AWS network hub for inter-VPC and hybrid connectivity |
| VPCE | VPC Endpoint — private connectivity to AWS services without internet transit |
| PHZ | Private Hosted Zone — Route 53 DNS zone resolvable only within associated VPCs |

### C. Decision Log

| ID | Decision | Date | Status |
|----|----------|------|--------|
| ADR-AV-001 | Hybrid StackSet rationalization | 2026-03-15 | Approved |
| ADR-AV-002 | 2–3 account archetypes (prod, non-prod, sandbox) | 2026-03-15 | Approved |
| ADR-AV-003 | Dual policy-as-code (OPA + Config Rules) | 2026-03-15 | Approved |
| ADR-AV-004 | Pipeline target state | TBD (Sprint 1) | Pending |
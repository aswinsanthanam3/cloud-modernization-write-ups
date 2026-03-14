# AWS Landing Zone Re-Architecture: StackSet Migration to AFT

## Project Overview

Migrate 75+ AWS CloudFormation StackSets—currently deployed as raw templates directly from the billing account with no IaC governance—into a fully managed AFT (Account Factory for Terraform) implementation. This migration must preserve stateful resources (IAM roles, third-party integrations) across <50 accounts while introducing account-level deployment flexibility that StackSets' OU-scoped model cannot provide.

---

## Current State

| Dimension | Current State |
|---|---|
| **StackSet Management** | 75+ raw CloudFormation templates deployed directly on the billing account, no IaC |
| **Account Vending** | AFT is operational but limited to account provisioning only |
| **AFT Customizations** | `aft-global-customizations` and `aft-account-customizations` are not built out |
| **Deployment Targeting** | OU-level StackSets; different OUs receive different subsets |
| **Account-Level Variation** | Not supported today; all accounts in an OU get the same resources |
| **Source of Truth** | Tribal knowledge held by team members |
| **StackSet Categories** | Security tooling (GuardDuty, SecurityHub, Config), third-party integrations (Datadog, Splunk, etc.), IAM roles for cross-account access |
| **Network Resources** | Deployed via Harness pipelines (out of scope for this project) |

## Target State

| Dimension | Target State |
|---|---|
| **StackSet Management** | Eliminated or wrapped under Terraform governance |
| **Account Vending** | AFT handles full account lifecycle: provisioning + customization |
| **AFT Customizations** | Tag-driven global and account customizations deploying all baseline resources |
| **Deployment Targeting** | Account-level via tags; OU structure reflects governance boundaries only |
| **Account-Level Variation** | Supported via tagging taxonomy and conditional Terraform modules |
| **Source of Truth** | Codified in tagging taxonomy, Terraform modules, and documentation |
| **OU Structure** | Restructured to reflect governance/SCP boundaries, decoupled from resource deployment |

---

## Problem Statements

### P1: No IaC Governance
75+ StackSets exist as unversioned CloudFormation templates deployed manually from the billing account. There is no version control, no PR review process, no audit trail, and no rollback capability. Any change is a manual, untracked operation.

### P2: Tribal Knowledge Dependency
The mapping of which StackSets deploy to which OUs, why they exist, and what account-level exceptions apply lives entirely in team members' heads. This creates single points of failure and makes onboarding, auditing, and incident response fragile.

### P3: OU-Level Granularity Is Insufficient
StackSets target OUs, meaning every account in an OU receives identical resources. In practice, accounts within the same OU have different integration, security, and access requirements. The current model forces either over-provisioning (deploy everything everywhere) or manual exceptions that circumvent the StackSet model.

### P4: OU Structure Conflates Governance with Resource Deployment
OUs are currently designed around "which StackSets to deploy" rather than "which governance policies (SCPs, billing, compliance) apply." This entangles two concerns that should be independent, making both harder to manage.

### P5: Stateful Resource Migration Risk
The majority of StackSets deploy stateful resources—IAM roles actively assumed by third-party tools, cross-account access roles in use by internal systems. Migration must preserve these resources without disrupting live integrations.

---

## Success Criteria

### Must Have (Quarter Exit Criteria)
- [ ] 100% of StackSets are either converted to native Terraform or wrapped under Terraform-managed StackSet resources—zero unmanaged StackSets remain
- [ ] All IaC is version-controlled with PR-based review and pipeline deployment
- [ ] A codified tagging taxonomy exists and is applied to all accounts
- [ ] `aft-global-customizations` deploys security baselines to all accounts
- [ ] `aft-account-customizations` deploys third-party integrations based on account tags
- [ ] Zero downtime or disruption to existing third-party integrations and cross-account access during migration
- [ ] New account provisioning via AFT automatically receives correct baseline + integrations based on tags

### Should Have
- [ ] OU restructuring completed and SCPs realigned
- [ ] Drift detection pipeline operational
- [ ] Runbook documentation for common operations (new account, add integration, modify baseline)

### Nice to Have (Phase 2 Backlog)
- [ ] Cross-account IAM role StackSets (Tier 2) fully converted to native Terraform
- [ ] Automated compliance validation pipeline

---

## Tiered Migration Strategy

The 75+ StackSets are classified into three tiers based on risk, complexity, and value of native Terraform conversion.

### Global Customizations (Convert to Native Terraform)
**What:** Security tooling—GuardDuty, SecurityHub, Config rules, CloudTrail, and similar resources that apply uniformly to every account with minimal variation.

**Why convert:** These are the simplest to convert (well-documented Terraform providers, low variation), and deploying them via `aft-global-customizations` ensures every account—existing and future—gets the baseline automatically.

**Estimated count:** ~15–20 StackSets

### Tier 1 Account Customizations (Convert to Native Terraform)
**What:** Third-party integrations—Datadog, Splunk, CrowdStrike, and similar tools that require per-account IAM roles and configuration, but where not every account needs every integration.

**Why convert:** These benefit most from the tag-driven account customization model. Native Terraform gives full state management and the ability to vary deployments per account. The blast radius per resource is contained (typically one IAM role + external ID per integration per account).

**Estimated count:** ~20–30 StackSets

### Tier 2 Account Customizations (Wrap Under Terraform, Convert Later)
**What:** Cross-account IAM roles—roles assumed by internal systems, CI/CD pipelines, and other AWS accounts for operational access.

**Why defer full conversion:** These are the highest-risk resources. Something is actively assuming each role, and the dependency mapping requires careful analysis. Wrapping them under Terraform-managed `aws_cloudformation_stack_set` resources gives IaC governance now without the risk of a state import breaking live access.

**Estimated count:** ~20–30 StackSets

---

## Sprint Plan

### Sprint 1 (Weeks 1–2): Inventory & Knowledge Capture

**Goal:** Replace tribal knowledge with a codified, version-controlled inventory of all StackSets and their deployment mapping.

**Deliverables:**
- Complete StackSet inventory spreadsheet/document: name, target OUs, parameters, resources created, category (security/integration/IAM), tier classification
- Account-to-StackSet mapping: which accounts receive which StackSets, and documented exceptions
- Dependency map for stateful resources: for each IAM role, document what assumes it (third-party service, internal system, CI/CD pipeline)
- Draft tagging taxonomy design

**Key Activities:**
- Run extraction scripts against the billing account to pull all StackSet metadata (`list-stack-sets`, `describe-stack-set`, `list-stack-instances`)
- Conduct knowledge transfer sessions with team members who hold tribal knowledge
- Classify each StackSet into Global / Tier 1 / Tier 2
- Begin drafting the target OU structure

**Risks & Mitigations:**
- Risk: Knowledge holders have gaps or conflicting understanding → Mitigation: Cross-validate against actual deployed resources using `list-stack-instances` and resource-level queries in target accounts
- Risk: Some StackSets are orphaned or redundant → Mitigation: Flag any StackSet with zero active instances or no clear owner for decommissioning review

---

### Sprint 2 (Week 3): Architecture & Repo Setup

**Goal:** Establish the AFT customization infrastructure and finalize the tagging taxonomy.

**Deliverables:**
- `aft-global-customizations` repo initialized with directory structure and CI/CD pipeline
- `aft-account-customizations` repo initialized with tag-driven conditional module pattern
- Finalized tagging taxonomy document
- Target OU restructuring plan documented and reviewed

**Key Activities:**
- Set up repo structures following AFT conventions
- Implement the tag-driven conditional pattern in `aft-account-customizations`:
  ```hcl
  locals {
    account_tags    = var.account_tags
    needs_datadog   = contains(split(",", lookup(local.account_tags, "aft:integrations", "")), "datadog")
    needs_splunk    = contains(split(",", lookup(local.account_tags, "aft:integrations", "")), "splunk")
    is_production   = lookup(local.account_tags, "aft:security-profile", "") == "production"
  }

  module "datadog_integration" {
    count  = local.needs_datadog ? 1 : 0
    source = "./modules/datadog"
  }
  ```
- Define and document the tagging taxonomy (e.g., `aft:integration-tier`, `aft:security-profile`, `aft:integrations`)
- Write initial Terraform modules for the 2–3 simplest global security resources as a proof of concept
- Validate AFT pipeline triggers customizations correctly in a sandbox account

**Risks & Mitigations:**
- Risk: AFT pipeline behavior for customizations is unfamiliar → Mitigation: Test with a single sandbox account before expanding; allocate time for debugging pipeline mechanics
- Risk: Tagging taxonomy is over-engineered → Mitigation: Start minimal (3–5 tags), expand only when a real need arises

---

### Sprint 3 (Weeks 4–5): Global Customizations Go Live

**Goal:** Convert all security tooling StackSets to native Terraform and deploy via `aft-global-customizations`. First StackSets decommissioned.

**Deliverables:**
- Terraform modules for all security baseline resources (GuardDuty, SecurityHub, Config, CloudTrail, etc.)
- Terraform state imported for all existing security resources across all accounts
- Old security tooling StackSets deleted (using `RetainStacks=true` to preserve resources)
- Validation report confirming all security tooling operational post-migration

**Key Activities:**
- Write Terraform modules for each security baseline resource
- Build and run the import automation script:
  ```bash
  for account_id in $(get_all_account_ids); do
    creds=$(assume_role $account_id "AFTExecutionRole")
    terraform import "module.guardduty.aws_guardduty_detector.main" "$detector_id"
    terraform import "module.securityhub.aws_securityhub_account.main" "$account_id"
    # ... repeat for each resource
  done
  ```
- Run `terraform plan` per account to confirm zero diff after import (state matches reality)
- Delete StackSet instances with `RetainStacks=true`, then delete the StackSet itself
- Trigger AFT pipeline for a few accounts to validate end-to-end flow

**Risks & Mitigations:**
- Risk: `terraform plan` shows unexpected diffs after import due to CFN vs Terraform resource attribute differences → Mitigation: Budget time for module adjustments; use `ignore_changes` lifecycle blocks sparingly for cosmetic diffs
- Risk: StackSet deletion fails due to instances still in progress → Mitigation: Verify all instances are in `SUCCEEDED` or `CURRENT` state before deletion

---

### Sprint 4 (Weeks 6–8): Tier 1 — Third-Party Integration Conversion

**Goal:** Convert all third-party integration StackSets to native Terraform modules deployed via tag-driven `aft-account-customizations`.

**Deliverables:**
- Terraform modules for each third-party integration (Datadog, Splunk, CrowdStrike, etc.)
- Account tags applied to all accounts based on the tagging taxonomy
- Terraform state imported for all integration resources in target accounts
- Integration validation confirming each third party can still assume their IAM role and operate correctly
- Old integration StackSets deleted

**Key Activities:**
- Apply tags to all accounts using the finalized taxonomy
- Convert integration StackSets to Terraform modules, one vendor at a time
- Per-vendor migration pattern:
  1. Write the Terraform module
  2. Import existing IAM role + resources in each target account
  3. `terraform plan` to confirm zero diff
  4. Validate third-party integration still works (test `sts:AssumeRole` for the external ID)
  5. Delete StackSet instances (`RetainStacks=true`) and StackSet
- Start with the most widely deployed integration (e.g., Datadog) to establish the pattern, then parallelize remaining vendors across the team

**Risks & Mitigations:**
- Risk: Third-party integration breaks after import if role ARN or trust policy changes → Mitigation: Import only (no modify); validate assume-role before deleting StackSet; keep StackSet as rollback until validation passes
- Risk: Some integrations have resources beyond a single IAM role (Lambda functions, CloudWatch log groups, etc.) → Mitigation: Sprint 1 inventory should capture all resources per StackSet; verify completeness before starting conversion

---

### Sprint 5 (Weeks 9–10): Tier 2 — Wrap Cross-Account IAM StackSets

**Goal:** Bring all remaining cross-account IAM role StackSets under Terraform governance without converting the underlying CloudFormation.

**Deliverables:**
- All Tier 2 StackSets managed via `aws_cloudformation_stack_set` and `aws_cloudformation_stack_set_instance` Terraform resources
- CFN templates stored in version control
- Deployment and parameter changes flow through Terraform plan/apply pipeline
- Detailed documentation of every cross-account IAM role: who/what assumes it, from which account, for what purpose

**Key Activities:**
- For each Tier 2 StackSet:
  1. Export the CFN template and commit to repo
  2. Write Terraform resource blocks referencing the template
  3. Import the existing StackSet and instances into Terraform state
  4. `terraform plan` to confirm zero diff
- Document each IAM role's consumers and purpose (feeds Phase 2 conversion planning)

**Risks & Mitigations:**
- Risk: Terraform import of StackSet instances is tedious for many-account deployments → Mitigation: Script the import; <50 accounts keeps this manageable
- Risk: Some StackSets have parameters that vary by account within the same OU → Mitigation: Use `parameter_overrides` on `aws_cloudformation_stack_set_instance` resources to capture account-level variation

---

### Sprint 6 (Week 11): OU Restructuring

**Goal:** Execute the planned OU restructuring now that resource deployment is decoupled from OU membership.

**Deliverables:**
- Accounts moved to new OU structure
- SCPs updated and validated for new OU boundaries
- AFT pipeline validated: moved accounts still receive correct customizations based on tags (not OU)
- No resource disruption during OU moves

**Key Activities:**
- Execute account moves per the restructuring plan from Sprint 2
- Update SCPs to align with new OU governance boundaries
- Validate that tag-driven AFT customizations are OU-independent
- Run `terraform plan` across all accounts to confirm no unintended changes
- Communicate OU changes to stakeholders (billing, compliance, security)

**Risks & Mitigations:**
- Risk: SCP changes inadvertently block AFT pipeline execution role → Mitigation: Ensure AFT execution role is explicitly allowed in all SCPs; test in sandbox OU first
- Risk: Some legacy automation depends on OU membership → Mitigation: Audit for any scripts, Lambda functions, or pipelines that reference OU IDs; update before moving accounts

---

### Sprint 7 (Week 12): Hardening & Handoff

**Goal:** Production-harden the new system and ensure the team can operate it independently.

**Deliverables:**
- Drift detection pipeline operational (scheduled `terraform plan` with alerting on drift)
- Operational runbooks:
  - How to onboard a new account via AFT (including tag assignment)
  - How to add or remove an integration for an existing account
  - How to modify the security baseline
  - How to troubleshoot AFT pipeline failures
- Phase 2 roadmap documented: plan for converting Tier 2 cross-account IAM StackSets to native Terraform
- Project retrospective completed

**Key Activities:**
- Set up scheduled drift detection (e.g., nightly `terraform plan` in a CI pipeline that alerts on non-zero diffs)
- Write and review operational runbooks
- Conduct knowledge transfer to the broader team
- Document Phase 2 scope, estimated effort, and prioritization for Tier 2 conversion
- Run a full validation sweep: every account has the expected resources based on its tags

**Risks & Mitigations:**
- Risk: Drift detection generates noisy false positives → Mitigation: Tune `ignore_changes` lifecycle blocks during Sprint 3–5 as diffs are discovered; establish a process for triaging drift alerts
- Risk: Runbooks are incomplete for edge cases → Mitigation: Have a team member who was not involved in the migration attempt each runbook procedure in a sandbox account

---

## Key Architecture Decisions

### Decision 1: Tag-Driven Over OU-Driven Customization
Account tags determine which Terraform modules are applied, not OU membership. This decouples governance (SCPs, billing) from resource deployment and enables account-level variation within the same OU.

### Decision 2: Tiered Migration Over Big-Bang Conversion
Converting all 75+ StackSets to native Terraform in one quarter with stateful resources is too risky. Wrapping Tier 2 (cross-account IAM) under Terraform-managed StackSets provides IaC governance immediately while deferring the riskiest conversions to Phase 2.

### Decision 3: Import-First, No Recreate
All stateful resources are imported into Terraform state rather than destroyed and recreated. This preserves IAM role ARNs, trust policies, and external configurations that third-party tools depend on.

### Decision 4: StackSet Deletion Uses RetainStacks
When deleting migrated StackSets, always use `RetainStacks=true` to preserve the underlying resources. Terraform already owns them via state import.

---

## Team & Resourcing

| Role | Count | Focus |
|---|---|---|
| Lead Engineer (you) | 1 | Architecture, taxonomy design, OU restructuring, stakeholder communication |
| Platform Engineers | 1–2 | Module development, import automation, pipeline setup, testing |

---

## Phase 2 Roadmap (Post-Quarter)

Items deferred to Phase 2 for future planning:

- Convert Tier 2 cross-account IAM StackSets from Terraform-wrapped CloudFormation to native Terraform modules
- Implement automated compliance validation pipeline
- Evaluate Harness pipeline migration for network resources (VPC, peering, TGW) into AFT or standalone Terraform
- Expand drift detection to include automated remediation for low-risk drifts
- Implement policy-as-code (e.g., OPA/Sentinel) for Terraform plan validation

---

## Appendix: References & Documentation

### AWS Control Tower & AFT

| # | Resource | URL |
|---|----------|-----|
| 1 | AFT Overview | https://docs.aws.amazon.com/controltower/latest/userguide/aft-overview.html |
| 2 | AFT Account Customization Options | https://docs.aws.amazon.com/controltower/latest/userguide/aft-account-customization-options.html |
| 3 | AFT Account Provisioning Pipeline | https://docs.aws.amazon.com/controltower/latest/userguide/aft-provisioning-framework.html |
| 4 | AFT Architecture & Order of Operations | https://docs.aws.amazon.com/controltower/latest/userguide/aft-architecture.html |
| 5 | Provision a New Account with AFT | https://docs.aws.amazon.com/controltower/latest/userguide/aft-provision-account.html |
| 6 | Update an Existing Account with AFT | https://docs.aws.amazon.com/controltower/latest/userguide/aft-update-account.html |
| 7 | Provision Accounts with AFT (Getting Started) | https://docs.aws.amazon.com/controltower/latest/userguide/taf-account-provisioning.html |
| 8 | AFT Blueprints (AWS Labs) | https://awslabs.github.io/aft-blueprints/getting-started/ |
| 9 | AWS Blog: Deploy & Customize Accounts Using AFT | https://aws.amazon.com/blogs/mt/deploy-and-customize-aws-accounts-using-account-factory-for-terraform-in-aws-control-tower/ |

### AWS Landing Zone & OU Structure

| # | Resource | URL |
|---|----------|-----|
| 10 | Plan Your AWS Control Tower Landing Zone | https://docs.aws.amazon.com/controltower/latest/userguide/planning-your-deployment.html |
| 11 | AWS Multi-Account Strategy for Control Tower | https://docs.aws.amazon.com/controltower/latest/userguide/aws-multi-account-landing-zone.html |
| 12 | Designing an AWS Control Tower Landing Zone (Prescriptive Guidance) | https://docs.aws.amazon.com/prescriptive-guidance/latest/designing-control-tower-landing-zone/introduction.html |
| 13 | Configuring Account Structure and OUs (Prescriptive Guidance) | https://docs.aws.amazon.com/prescriptive-guidance/latest/designing-control-tower-landing-zone/account-structure.html |
| 14 | OU Structure Best Practices & Lessons Learned (Prescriptive Guidance) | https://docs.aws.amazon.com/prescriptive-guidance/latest/ou-structure-landing-zone/best-practices.html |
| 15 | AWS Blog: Organizing Your Landing Zone with Nested OUs | https://aws.amazon.com/blogs/mt/organizing-your-aws-control-tower-landing-zone-with-nested-ous/ |

### CloudFormation StackSets

| # | Resource | URL |
|---|----------|-----|
| 16 | StackSets Concepts | https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-concepts.html |
| 17 | Delete Stacks from StackSets (`--retain-stacks`) | https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stackinstances-delete.html |
| 18 | Delete StackSets | https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-delete.html |
| 19 | DeleteStackInstances API Reference | https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_DeleteStackInstances.html |
| 20 | Troubleshooting StackSets | https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-troubleshooting.html |
| 21 | Delete StackSets and Stacks (Control Tower) | https://docs.aws.amazon.com/controltower/latest/userguide/controltower-walkthrough-delete-stacksets.html |
| 22 | DeletionPolicy Attribute | https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-attribute-deletionpolicy.html |

### Terraform — AWS Provider Resources

| # | Resource | URL |
|---|----------|-----|
| 23 | `aws_cloudformation_stack_set` | https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudformation_stack_set |
| 24 | `aws_cloudformation_stack_set_instance` | https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudformation_stack_set_instance |
| 25 | `aws_cloudformation_stack` | https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudformation_stack |

### Terraform — Import & State Management

| # | Resource | URL |
|---|----------|-----|
| 26 | `terraform import` Command Reference | https://developer.hashicorp.com/terraform/cli/commands/import |
| 27 | Import Existing Infrastructure (Overview) | https://developer.hashicorp.com/terraform/cli/import |
| 28 | `terraform import` Usage Guide | https://developer.hashicorp.com/terraform/cli/import/usage |
| 29 | Import Block Reference (Declarative Import) | https://developer.hashicorp.com/terraform/language/import |
| 30 | Tutorial: Import Terraform Configuration | https://developer.hashicorp.com/terraform/tutorials/state/state-import |

### Terraform — AFT Tutorial

| # | Resource | URL |
|---|----------|-----|
| 31 | HashiCorp Tutorial: Manage AWS Accounts Using Control Tower AFT | https://developer.hashicorp.com/terraform/tutorials/aws/aws-control-tower-aft |
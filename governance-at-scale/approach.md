# Account Security Scorecard — Architecture and Implementation

**Reference:** ADR-2025-002, Implementation Plan Phase 2 + Phase 4
**Date:** 2026-03-15
**Author:** Aswin (Platform Engineering)
**Scope:** New accounts vended via AFT only

---

## 1. Overview

Every account vended through the AFT pipeline receives an automated security scorecard that continuously measures two metrics against tier-appropriate thresholds. When scores drop below defined thresholds, the system automatically notifies account owners and escalates to the platform team.

The scorecard only measures controls that are **applicable to the account's tier**. A Sandbox account with 24 applicable CIS controls is scored against those 24 — not against the full 40. This ensures teams receive fair, actionable scores relative to the expectations for their environment.

```
                    ┌─────────────────────────┐
                    │   AFT Vends Account      │
                    │   (assigns tier tag)      │
                    └────────────┬──────────────┘
                                 │
                    ┌────────────▼──────────────┐
                    │  Security Hub Enabled      │
                    │  (tier-specific config     │
                    │   policy applied)          │
                    └────────────┬──────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                   │
    ┌─────────▼────────┐ ┌──────▼───────┐ ┌────────▼────────┐
    │ Security Hub      │ │ Config Rule  │ │ Tag Policy      │
    │ CIS v5.0 Checks   │ │ required-tags│ │ (Organizations) │
    └─────────┬────────┘ └──────┬───────┘ └────────┬────────┘
              │                  │                   │
              └──────────────────┼───────────────────┘
                                 │
                    ┌────────────▼──────────────┐
                    │  Scorecard Lambda          │
                    │  (every 6 hours)           │
                    │                            │
                    │  1. Query Security Hub API │
                    │     per-account compliance │
                    │  2. Query Config API       │
                    │     tag compliance         │
                    │  3. Look up tier from tag  │
                    │  4. Calculate score using  │
                    │     tier-specific denominator│
                    │  5. Compare to thresholds  │
                    │  6. Write to DynamoDB      │
                    │  7. Trigger notifications  │
                    └────────────┬──────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                   │
    ┌─────────▼────────┐ ┌──────▼───────┐ ┌────────▼────────┐
    │ DynamoDB          │ │ SNS / Slack  │ │ QuickSight      │
    │ (score history)   │ │ (alerts)     │ │ (dashboard)     │
    └──────────────────┘ └──────────────┘ └─────────────────┘
```

---

## 2. Scorecard Metrics

### 2.1 CIS Compliance Score

**Source:** Security Hub `GetFindingAggregator` and `BatchGetSecurityControls` APIs

**Calculation:**

```
cis_score = (passed_controls / applicable_controls_for_tier) × 100
```

Where `applicable_controls_for_tier` is determined by the tier applicability matrix in the compliance control mapping YAML:

| Tier             | Applicable Controls | Scorecard Denominator |
|------------------|--------------------|-----------------------|
| Production       | 40                 | 40                    |
| Development      | 34                 | 34                    |
| Sandbox          | 24                 | 24                    |
| Shared Services  | 40                 | 40                    |

Controls with status `NOT_AVAILABLE` (resource doesn't exist in the account) are excluded from both numerator and denominator. For example, if a Production account has no EFS file systems, EFS.1 and EFS.8 are excluded — the denominator drops to 38.

### 2.2 Tag Compliance Score

**Source:** AWS Config rule `required-tags` evaluated against all taggable resources

**Calculation:**

```
tag_score = (resources_with_all_mandatory_tags / total_taggable_resources) × 100
```

**Mandatory tags:**

| Tag Key              | Validation                                        |
|----------------------|---------------------------------------------------|
| `Environment`        | Must match: `production`, `development`, `sandbox`, `shared-services` |
| `CostCenter`         | Regex: `^CC-\d{6}$`                              |
| `Owner`              | Valid email format                                 |
| `DataClassification` | Must match: `public`, `internal`, `confidential`, `restricted` |

A resource is "compliant" only if ALL four tags are present and valid. Missing or malformed tags = non-compliant.

---

## 3. Threshold Model

Tiered thresholds with progressive escalation:

```
100% ─────────────────────────────────────── Target
 95% ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  Prod/Shared target
 90% ════════════════════════════════════════ WARNING
       → Notify account owner
       → Yellow flag on dashboard

 85% ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  Dev/Sandbox target
 80% ════════════════════════════════════════ CRITICAL
       → Notify account owner
       → Escalate to platform team
       → Orange flag on dashboard

 70% ════════════════════════════════════════ ESCALATION
       → Notify account owner
       → Escalate to platform team
       → Flag for leadership review
       → Red flag on dashboard
       → Repeated: manual quarantine SCP review

  0% ───────────────────────────────────────
```

Thresholds apply uniformly to both CIS and tag compliance scores. An account enters the worst applicable state — if CIS is 92% (warning) but tags are 75% (critical), the account state is CRITICAL.

---

## 4. Notification Pipeline

### 4.1 Architecture

```
Scorecard Lambda
      │
      ├── Score ≥ 90% (both metrics)
      │       → No action
      │       → Green status in DynamoDB + dashboard
      │
      ├── Score < 90% AND ≥ 80% (WARNING)
      │       → SNS → account owner email
      │       → Slack → #cloud-governance (warning template)
      │       → DynamoDB → status = WARNING, timestamp
      │
      ├── Score < 80% AND ≥ 70% (CRITICAL)
      │       → SNS → account owner email + platform team
      │       → Slack → #cloud-governance + #platform-alerts (critical template)
      │       → DynamoDB → status = CRITICAL, timestamp
      │
      └── Score < 70% (ESCALATION)
              → SNS → account owner + platform team + eng leadership
              → Slack → #cloud-governance + #platform-alerts + #leadership (escalation template)
              → DynamoDB → status = ESCALATION, timestamp, escalation_count++
```

### 4.2 Notification Content

Each notification includes:

- Account ID, account name, tier, and account owner
- Current CIS score and tag score (with delta from previous evaluation)
- Top 5 failing controls with Security Hub remediation links
- Top 5 resources with missing tags
- Link to the real-time QuickSight dashboard filtered to this account
- Days in current threshold state (to track persistent non-compliance)

### 4.3 De-duplication

Notifications are de-duplicated to avoid alert fatigue:

- **Same threshold state:** Notify once on entry, then suppress for 24 hours. Re-notify if score worsens within the same state.
- **State change (e.g., WARNING → CRITICAL):** Always notify immediately.
- **Recovery (e.g., CRITICAL → WARNING or → GREEN):** Send recovery notification once.

---

## 5. Dashboard Design

### 5.1 QuickSight Dashboard — Data Flow

```
Security Hub (per-account scores)
        │
        ▼
Scorecard Lambda (every 6 hrs)
        │
        ▼
DynamoDB Table: account-scorecards
  ├── PK: account_id
  ├── SK: evaluation_timestamp
  ├── tier: "production" | "development" | "sandbox" | "shared_services"
  ├── cis_score: 94.7
  ├── tag_score: 88.2
  ├── status: "WARNING" | "CRITICAL" | "ESCALATION" | "GREEN"
  ├── applicable_controls: 40
  ├── passed_controls: 38
  ├── failing_controls: ["EC2.8", "IAM.3"]
  ├── taggable_resources: 127
  ├── tag_compliant_resources: 112
  ├── account_name: "prod-payments-001"
  ├── account_owner: "team-payments@company.com"
  └── days_in_state: 3
        │
        ▼
QuickSight (connected to DynamoDB via Athena or direct)
```

### 5.2 Dashboard Views

**Organization Overview (Platform Team)**

- Heatmap grid: all accounts × score (color-coded green/yellow/orange/red)
- Aggregated score distribution by tier
- Trend chart: organization-wide average CIS and tag scores over 90 days
- Top 10 worst-scoring accounts
- Top 10 most-failed controls across all accounts

**Account Detail (Account Owner)**

- Current CIS score with gauge visualization
- Current tag score with gauge visualization
- Score trend over 30 days
- Failing controls list with severity and remediation link
- Non-compliant resources list with missing tags
- Comparison to tier average ("Your score: 87% | Tier average: 93%")

**Tier Summary (Engineering Leadership)**

- Average score per tier with trend
- Count of accounts per threshold state (GREEN / WARNING / CRITICAL / ESCALATION)
- Accounts in ESCALATION state with days-in-state counter

---

## 6. AFT Integration — New Accounts Only

### 6.1 How Tier Assignment Works

When an account is vended via AFT, the account request includes the tier:

```hcl
# aft-account-request/terraform/prod-payments-001/main.tf
module "prod_payments_001" {
  source = "./modules/aft-account-request"

  control_tower_parameters = {
    AccountEmail = "aws+prod-payments-001@company.com"
    AccountName  = "prod-payments-001"
    ManagedOrganizationalUnit = "Production"
    SSOUserEmail     = "team-payments@company.com"
    SSOUserFirstName = "Payments"
    SSOUserLastName  = "Team"
  }

  account_tags = {
    Environment        = "production"
    CostCenter         = "CC-004521"
    Owner              = "team-payments@company.com"
    DataClassification = "confidential"
  }

  account_customizations_name = "production"   # ← This selects the tier baseline
}
```

The `account_customizations_name` maps to the AFT account customizations directory, which triggers:

1. Security Hub central configuration policy assignment (tier-specific)
2. Config rule deployment (tier-specific conformance pack)
3. ASR member stack deployment with tier-appropriate remediation toggles
4. Scorecard Lambda registration (adds account to DynamoDB tracking table)
5. Tag policy enforcement verification

### 6.2 Scorecard Lambda — Account Discovery

The scorecard Lambda discovers which accounts to evaluate by querying accounts tagged with `ManagedBy: AFT`. Brownfield accounts without this tag are excluded.

```python
# Pseudocode — account discovery
def get_scored_accounts():
    """Return only AFT-vended accounts for scorecard evaluation."""
    orgs_client = boto3.client('organizations')
    accounts = []
    for account in orgs_client.list_accounts():
        tags = orgs_client.list_tags_for_resource(ResourceId=account['Id'])
        tag_map = {t['Key']: t['Value'] for t in tags['Tags']}
        if tag_map.get('ManagedBy') == 'AFT':
            accounts.append({
                'account_id': account['Id'],
                'account_name': account['Name'],
                'tier': tag_map.get('Environment', 'unknown'),
                'owner': tag_map.get('Owner', 'unknown'),
            })
    return accounts
```

### 6.3 Brownfield Account Exclusion

Existing accounts are explicitly excluded from:

- Scorecard evaluation (no `ManagedBy: AFT` tag)
- Automated remediation (ASR member stack not deployed)
- Tier-specific Security Hub configuration policies (retain existing Security Hub config)
- Compliance reporting (Audit Manager assessments can include them for evidence collection only)

A future migration phase can enroll brownfield accounts by tagging them with `ManagedBy: AFT` and deploying the tier baseline. This is a deliberate opt-in, not automatic.

---

## 7. Implementation Tasks

These tasks integrate into the existing implementation plan phasing:

### Phase 2 Additions (Weeks 7–12)

| # | Task | Owner | Phase Reference |
|---|------|-------|-----------------|
| SC-1 | Create DynamoDB table `account-scorecards` with PK=account_id, SK=timestamp | Platform Eng | Phase 2 |
| SC-2 | Build scorecard Lambda function (Security Hub + Config API queries, tier lookup, threshold comparison) | Platform Eng | Phase 2 |
| SC-3 | Create EventBridge scheduled rule (every 6 hours) | Platform Eng | Phase 2 |
| SC-4 | Create SNS topics for warning/critical/escalation | Platform Eng | Phase 2 |
| SC-5 | Configure Slack webhook integration for #cloud-governance | Platform Eng | Phase 2 |
| SC-6 | Build QuickSight dataset connected to DynamoDB | Platform Eng | Phase 2 |
| SC-7 | Build QuickSight dashboard (org overview + account detail + tier summary) | Platform Eng | Phase 2 |
| SC-8 | Create Security Hub central configuration policies (one per tier) | Platform Eng | Phase 2 |
| SC-9 | Test scorecard end-to-end with a test account per tier | Platform Eng | Phase 2 |

### Phase 4 Additions (Weeks 19–24)

| # | Task | Owner | Phase Reference |
|---|------|-------|-----------------|
| SC-10 | Add `ManagedBy: AFT` tag to AFT global customizations | Platform Eng | Phase 4 |
| SC-11 | Add scorecard Lambda registration to AFT global customizations | Platform Eng | Phase 4 |
| SC-12 | Add tier-specific Security Hub policy assignment to AFT account customizations | Platform Eng | Phase 4 |
| SC-13 | Validate first 5 AFT-vended accounts appear in scorecard dashboard | Platform Eng + Security | Phase 4 |
| SC-14 | Build weekly summary report Lambda (Monday 08:00 UTC email) | Platform Eng | Phase 4 |

---

## 8. Cost Impact

The scorecard adds minimal cost to the overall architecture:

| Component | Pricing | Estimated Monthly Cost |
|-----------|---------|----------------------|
| DynamoDB table | On-demand: $1.25/million writes, $0.25/million reads | $5–$15 (at 100 accounts × 4 evals/day) |
| Scorecard Lambda | $0.20/million invocations + duration | $2–$5 |
| EventBridge scheduled rule | Free (included) | $0 |
| SNS notifications | $0.50/million notifications | <$1 |
| QuickSight | $18/author/month; $5/reader/month (session-based) | $36–$72 (2 authors + 10 readers) |
| **Total scorecard cost** | | **$44–$93/month** |

---

## Appendix: Sources and References

1. **AWS Security Hub — GetFindings API.** AWS. https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-api.html

2. **Security Hub Central Configuration.** AWS. https://docs.aws.amazon.com/securityhub/latest/userguide/central-configuration-intro.html

3. **AWS Config — required-tags Managed Rule.** AWS. https://docs.aws.amazon.com/config/latest/developerguide/required-tags.html

4. **Amazon QuickSight Pricing.** AWS. https://aws.amazon.com/quicksight/pricing/

5. **CIS AWS Foundations Benchmark v5.0.0 Controls in Security Hub.** AWS. https://docs.aws.amazon.com/securityhub/latest/userguide/cis-aws-foundations-benchmark.html
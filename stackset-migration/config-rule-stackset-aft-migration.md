# Migrating AWS Config Rules: StackSet → AFT Global Customizations

**Migration Type:** Zero-downtime state import  
**Source:** CloudFormation StackSet (unmanaged)  
**Target:** AFT `aft-global-customizations` repo (Terraform)  
**Scope:** Two examples — managed rule + custom (Lambda-backed) rule

---

## Understanding the Migration Path

Here's what's happening conceptually:

```
BEFORE (Current State)
══════════════════════

CloudFormation StackSet (in management account)
    │
    │ deploys stack instances to each member account
    │
    ├── Account A: CF Stack → aws_config_config_rule "s3-encryption-check"
    ├── Account B: CF Stack → aws_config_config_rule "s3-encryption-check"
    └── Account C: CF Stack → aws_config_config_rule "s3-encryption-check"

    Problem: No IaC, no state management, no version control.
    The StackSet is managed via console/CLI. Changes are manual.


AFTER (Target State)
════════════════════

aft-global-customizations repo (in version control)
    │
    │ AFT pipeline runs per-account CodePipeline
    │
    ├── Account A: Terraform state → aws_config_config_rule "s3-encryption-check"
    ├── Account B: Terraform state → aws_config_config_rule "s3-encryption-check"
    └── Account C: Terraform state → aws_config_config_rule "s3-encryption-check"

    Benefit: IaC, state tracked, versioned, tested, applied via GitOps.
```

**The critical challenge:** The Config Rules already exist in every account. If you
just write Terraform and let AFT apply it, Terraform will try to CREATE new rules.
AWS will either error (duplicate name) or create duplicates. You need to IMPORT the
existing rules into Terraform state first, then delete the StackSet — in that order.

---

## Phase 1: Inventory the Existing StackSet

Before writing any Terraform, document exactly what the StackSet deploys. This
information lives in tribal knowledge today — externalize it now.

### Step 1.1: Export the StackSet Template

```bash
# Get the StackSet template from the management account
aws cloudformation describe-stack-set \
  --stack-set-name "OrgConfigRules" \
  --query "StackSet.TemplateBody" \
  --output text > stackset-template.yaml
```

### Step 1.2: Document What the StackSet Deploys

For this walkthrough, assume the StackSet template contains two Config Rules:

**Rule 1 — Managed Rule (AWS-provided):**

```yaml
# Inside stackset-template.yaml
Resources:
  S3EncryptionCheck:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: s3-bucket-encryption-check
      Description: "Checks that S3 buckets have server-side encryption enabled"
      Source:
        Owner: AWS
        SourceIdentifier: S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED
      Scope:
        ComplianceResourceTypes:
          - "AWS::S3::Bucket"
      MaximumExecutionFrequency: TwentyFour_Hours
```

**Rule 2 — Custom Rule (Lambda-backed):**

```yaml
  CustomTagValidation:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: custom-mandatory-tag-check
      Description: "Validates all resources have mandatory tags"
      Source:
        Owner: CUSTOM_LAMBDA
        SourceIdentifier: !GetAtt TagValidationLambda.Arn
        SourceDetails:
          - EventSource: aws.config
            MessageType: ConfigurationItemChangeNotification
      Scope:
        ComplianceResourceTypes:
          - "AWS::EC2::Instance"
          - "AWS::S3::Bucket"
          - "AWS::RDS::DBInstance"
      InputParameters: '{"requiredTags": "Team,Environment,CostCenter,ManagedBy"}'

  TagValidationLambda:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: config-rule-tag-validation
      Runtime: python3.12
      Handler: index.handler
      Role: !GetAtt TagValidationRole.Arn
      Code:
        ZipFile: |
          import json
          import boto3

          REQUIRED_TAGS = ["Team", "Environment", "CostCenter", "ManagedBy"]

          def handler(event, context):
              config = boto3.client("config")
              invoking_event = json.loads(event["invokingEvent"])
              configuration_item = invoking_event["configurationItem"]

              tags = configuration_item.get("tags", {})
              missing = [t for t in REQUIRED_TAGS if t not in tags]

              evaluation = {
                  "ComplianceResourceType": configuration_item["resourceType"],
                  "ComplianceResourceId": configuration_item["resourceId"],
                  "ComplianceType": "NON_COMPLIANT" if missing else "COMPLIANT",
                  "Annotation": f"Missing tags: {', '.join(missing)}" if missing else "All required tags present",
                  "OrderingTimestamp": configuration_item["configurationItemCaptureTime"],
              }

              config.put_evaluations(
                  Evaluations=[evaluation],
                  ResultToken=event["resultToken"],
              )

  TagValidationRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: config-rule-tag-validation-role
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWS_ConfigRole
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

### Step 1.3: List All Accounts Where the StackSet Is Deployed

```bash
# List all stack instances
aws cloudformation list-stack-instances \
  --stack-set-name "OrgConfigRules" \
  --query "Summaries[].{Account:Account,Region:Region,Status:Status}" \
  --output table
```

Save this output — you'll need to import state in every one of these accounts.

---

## Phase 2: Write the Terraform in AFT Global Customizations

### Step 2.1: AFT Repository Structure

```
aft-global-customizations/
├── terraform/
│   ├── config_rules.tf          # Config Rule resources
│   ├── config_rules_lambda.tf   # Lambda function for custom rule
│   ├── config_rules_iam.tf      # IAM role for Lambda
│   ├── config_rules_import.tf   # Import blocks (temporary, removed after migration)
│   ├── variables.tf             # Shared variables
│   └── data.tf                  # Data sources
├── api_helpers/
│   └── python/
│       └── source/
│           └── tag_validation/
│               └── index.py     # Lambda source code (extracted from CF inline)
└── README.md
```

### Step 2.2: Managed Rule — Terraform Configuration

```hcl
# terraform/config_rules.tf

# ─────────────────────────────────────────────────
# Rule 1: Managed Rule (S3 Encryption Check)
#
# This is a direct equivalent of the StackSet's
# AWS::Config::ConfigRule with Source.Owner = AWS.
#
# AWS provides the evaluation logic — we just
# configure the rule.
# ─────────────────────────────────────────────────

resource "aws_config_config_rule" "s3_encryption_check" {
  name        = "s3-bucket-encryption-check"
  description = "Checks that S3 buckets have server-side encryption enabled"

  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED"
  }

  scope {
    compliance_resource_types = ["AWS::S3::Bucket"]
  }

  maximum_execution_frequency = "TwentyFour_Hours"

  tags = {
    ManagedBy   = "terraform"
    Source      = "aft-global-customizations"
    MigratedFrom = "StackSet:OrgConfigRules"
  }

  # Config Rule requires an existing Configuration Recorder.
  # In Control Tower accounts, this already exists — but the
  # depends_on is a safety net in case of race conditions.
  depends_on = [data.aws_config_configuration_recorder.existing]
}

# ─────────────────────────────────────────────────
# IMPORTANT: Config Rules need a Configuration Recorder
# to already be running. In Control Tower accounts,
# AWS Config is enabled by default. We use a data source
# to confirm it exists rather than creating one.
# ─────────────────────────────────────────────────

data "aws_config_configuration_recorder" "existing" {}
```

**Key differences from the CloudFormation version:**

The Terraform resource maps almost 1:1 to the CloudFormation resource, but with
a few naming differences:

| CloudFormation Property | Terraform Argument | Notes |
|---|---|---|
| `ConfigRuleName` | `name` | Same value |
| `Source.Owner` | `source.owner` | Same value (`AWS`) |
| `Source.SourceIdentifier` | `source.source_identifier` | Same value |
| `Scope.ComplianceResourceTypes` | `scope.compliance_resource_types` | Same value |
| `MaximumExecutionFrequency` | `maximum_execution_frequency` | Same value |

### Step 2.3: Custom Rule (Lambda-Backed) — Terraform Configuration

The custom rule is more complex because it includes a Lambda function and IAM role.

First, extract the Lambda code from the CloudFormation inline `ZipFile` into a
proper file:

```python
# api_helpers/python/source/tag_validation/index.py
"""
AWS Config custom rule: validates mandatory tags on resources.

Migrated from StackSet OrgConfigRules → AFT global customizations.
"""
import json
import boto3

REQUIRED_TAGS = ["Team", "Environment", "CostCenter", "ManagedBy"]


def handler(event, context):
    """Evaluate resource compliance based on mandatory tags."""
    config = boto3.client("config")
    invoking_event = json.loads(event["invokingEvent"])
    configuration_item = invoking_event["configurationItem"]

    tags = configuration_item.get("tags", {})
    missing = [t for t in REQUIRED_TAGS if t not in tags]

    evaluation = {
        "ComplianceResourceType": configuration_item["resourceType"],
        "ComplianceResourceId": configuration_item["resourceId"],
        "ComplianceType": "NON_COMPLIANT" if missing else "COMPLIANT",
        "Annotation": (
            f"Missing tags: {', '.join(missing)}" if missing else "All required tags present"
        ),
        "OrderingTimestamp": configuration_item["configurationItemCaptureTime"],
    }

    config.put_evaluations(
        Evaluations=[evaluation],
        ResultToken=event["resultToken"],
    )

    return evaluation
```

Now the Terraform for the Lambda function:

```hcl
# terraform/config_rules_lambda.tf

# ─────────────────────────────────────────────────
# Lambda function for the custom Config Rule.
#
# IMPORTANT: In the StackSet, the Lambda code was
# inline (ZipFile). In Terraform, we package it
# as a zip and deploy from a local file.
#
# AFT runs Terraform from the aft-global-customizations
# repo, so the file path is relative to the terraform/
# directory.
# ─────────────────────────────────────────────────

data "archive_file" "tag_validation" {
  type        = "zip"
  source_dir  = "${path.module}/../api_helpers/python/source/tag_validation"
  output_path = "${path.module}/lambda_packages/tag_validation.zip"
}

resource "aws_lambda_function" "tag_validation" {
  function_name = "config-rule-tag-validation"
  description   = "Custom Config Rule: validates mandatory tags on resources"

  filename         = data.archive_file.tag_validation.output_path
  source_code_hash = data.archive_file.tag_validation.output_base64sha256

  runtime = "python3.12"
  handler = "index.handler"
  timeout = 60

  role = aws_iam_role.config_rule_tag_validation.arn

  tags = {
    ManagedBy    = "terraform"
    Source       = "aft-global-customizations"
    MigratedFrom = "StackSet:OrgConfigRules"
  }
}

resource "aws_lambda_permission" "allow_config" {
  statement_id  = "AllowConfigInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.tag_validation.function_name
  principal     = "config.amazonaws.com"
}
```

The IAM role:

```hcl
# terraform/config_rules_iam.tf

resource "aws_iam_role" "config_rule_tag_validation" {
  name = "config-rule-tag-validation-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
      Action = "sts:AssumeRole"
    }]
  })

  tags = {
    ManagedBy    = "terraform"
    Source       = "aft-global-customizations"
    MigratedFrom = "StackSet:OrgConfigRules"
  }
}

resource "aws_iam_role_policy_attachment" "config_rule_policy" {
  role       = aws_iam_role.config_rule_tag_validation.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWS_ConfigRole"
}

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.config_rule_tag_validation.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}
```

The custom Config Rule itself:

```hcl
# terraform/config_rules.tf (continued — add below the managed rule)

# ─────────────────────────────────────────────────
# Rule 2: Custom Rule (Lambda-backed Tag Validation)
#
# This is the equivalent of the StackSet's
# AWS::Config::ConfigRule with Source.Owner = CUSTOM_LAMBDA.
#
# The Lambda function evaluates compliance — we provide
# the function ARN as the source identifier.
# ─────────────────────────────────────────────────

resource "aws_config_config_rule" "custom_tag_validation" {
  name        = "custom-mandatory-tag-check"
  description = "Validates all resources have mandatory tags"

  source {
    owner = "CUSTOM_LAMBDA"

    # IMPORTANT: This must be the Lambda ARN, not the function name.
    source_identifier = aws_lambda_function.tag_validation.arn

    source_detail {
      event_source = "aws.config"
      message_type = "ConfigurationItemChangeNotification"
    }
  }

  scope {
    compliance_resource_types = [
      "AWS::EC2::Instance",
      "AWS::S3::Bucket",
      "AWS::RDS::DBInstance",
    ]
  }

  input_parameters = jsonencode({
    requiredTags = "Team,Environment,CostCenter,ManagedBy"
  })

  tags = {
    ManagedBy    = "terraform"
    Source       = "aft-global-customizations"
    MigratedFrom = "StackSet:OrgConfigRules"
  }

  depends_on = [
    aws_lambda_permission.allow_config,
    data.aws_config_configuration_recorder.existing,
  ]
}
```

---

## Phase 3: The State Import (The Hard Part)

This is where most migrations fail. The resources already exist in AWS (created by
the StackSet). Terraform doesn't know about them. If you run `terraform apply` now,
it will try to create duplicates and fail.

You have two options for importing. Use the one that fits your Terraform version.

### Option A: Declarative Import Blocks (Terraform >= 1.5 — Recommended)

Add a temporary import file:

```hcl
# terraform/config_rules_import.tf
#
# TEMPORARY FILE — remove after successful migration.
#
# These import blocks tell Terraform to adopt existing AWS resources
# into state instead of creating new ones. The 'id' is the Config
# Rule name (for Config Rules) or the function name (for Lambda).
#
# After import succeeds across all accounts, delete this file and
# commit. Subsequent plans will show no changes.

# ── Managed Config Rule ──
import {
  to = aws_config_config_rule.s3_encryption_check
  id = "s3-bucket-encryption-check"
}

# ── Custom Config Rule ──
import {
  to = aws_config_config_rule.custom_tag_validation
  id = "custom-mandatory-tag-check"
}

# ── Lambda Function ──
import {
  to = aws_lambda_function.tag_validation
  id = "config-rule-tag-validation"
}

# ── IAM Role ──
import {
  to = aws_iam_role.config_rule_tag_validation
  id = "config-rule-tag-validation-role"
}

# ── IAM Policy Attachments ──
# Format: role-name/policy-arn
import {
  to = aws_iam_role_policy_attachment.config_rule_policy
  id = "config-rule-tag-validation-role/arn:aws:iam::aws:policy/service-role/AWS_ConfigRole"
}

import {
  to = aws_iam_role_policy_attachment.lambda_basic
  id = "config-rule-tag-validation-role/arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# ── Lambda Permission ──
# Format: function-name/statement-id
import {
  to = aws_lambda_permission.allow_config
  id = "config-rule-tag-validation/AllowConfigInvoke"
}
```

### Option B: CLI Import (Terraform < 1.5 — Manual)

If you're running an older Terraform version in AFT, you'd run these commands
manually per account. This is much harder to automate across all accounts.

```bash
# Assume into the target account first
# (AFT uses AWSAFTExecution role)

terraform import aws_config_config_rule.s3_encryption_check \
  "s3-bucket-encryption-check"

terraform import aws_config_config_rule.custom_tag_validation \
  "custom-mandatory-tag-check"

terraform import aws_lambda_function.tag_validation \
  "config-rule-tag-validation"

terraform import aws_iam_role.config_rule_tag_validation \
  "config-rule-tag-validation-role"

terraform import aws_iam_role_policy_attachment.config_rule_policy \
  "config-rule-tag-validation-role/arn:aws:iam::aws:policy/service-role/AWS_ConfigRole"

terraform import aws_iam_role_policy_attachment.lambda_basic \
  "config-rule-tag-validation-role/arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"

terraform import aws_lambda_permission.allow_config \
  "config-rule-tag-validation/AllowConfigInvoke"
```

---

## Phase 4: The Migration Sequence (Step by Step)

This is the exact order of operations. Getting this wrong causes downtime or
duplicate resources.

### Step 4.1: Dry Run on ONE Account First

Do NOT push to the AFT repo yet. Run the import manually against one sandbox/dev
account to validate.

```bash
# 1. Clone your aft-global-customizations repo
git clone <your-repo-url>
cd aft-global-customizations/terraform

# 2. Assume the AWSAFTExecution role in the target account
# (or use your sandbox account credentials)
export AWS_PROFILE=sandbox-account

# 3. Initialize Terraform with a LOCAL backend (not AFT's S3 backend)
#    This is a test run — we don't want to pollute AFT's state yet.
cat > backend_override.tf << 'EOF'
terraform {
  backend "local" {
    path = "test-import.tfstate"
  }
}
EOF

terraform init

# 4. Run plan — this should show the imports + minimal changes
terraform plan

# EXPECTED OUTPUT:
# aws_config_config_rule.s3_encryption_check: Importing...
# aws_config_config_rule.custom_tag_validation: Importing...
# aws_lambda_function.tag_validation: Importing...
# aws_iam_role.config_rule_tag_validation: Importing...
# (etc.)
#
# Plan: 7 to import, 0 to add, N to change, 0 to destroy.
#
# The "N to change" is expected — these are usually tag additions
# (ManagedBy, Source, MigratedFrom) and minor attribute differences
# between how CloudFormation and Terraform represent the same resource.

# 5. Review EVERY change carefully.
#    If you see "destroy" on any existing resource, STOP.
#    Something in your Terraform config doesn't match the existing resource.
#    Adjust your .tf files until plan shows only imports + tag changes.

# 6. Apply (still against local state, not AFT)
terraform apply

# 7. Verify the Config Rules are still working
aws configservice describe-config-rules \
  --config-rule-names "s3-bucket-encryption-check" "custom-mandatory-tag-check" \
  --query "ConfigRules[].{Name:ConfigRuleName,State:ConfigRuleState}"

# Expected: both rules show "ACTIVE"

# 8. Clean up the test
rm test-import.tfstate test-import.tfstate.backup backend_override.tf
```

### Step 4.2: Commit to the AFT Repo (All Accounts)

Once the dry run passes cleanly on one account:

```bash
# 1. Remove the backend override (AFT provides its own backend via Jinja)
rm -f backend_override.tf

# 2. Commit the Terraform code WITH import blocks
git add terraform/
git add api_helpers/
git commit -m "feat: migrate Config Rules from StackSet to AFT global customizations

Migrates s3-bucket-encryption-check (managed rule) and
custom-mandatory-tag-check (Lambda-backed custom rule) from
CloudFormation StackSet 'OrgConfigRules' to Terraform managed
by AFT global customizations.

Import blocks included to adopt existing resources into state.
These blocks will be removed after migration is confirmed across
all accounts.

Refs: TP-2025-002"

# 3. Push to trigger AFT pipelines
git push origin main
```

### Step 4.3: Monitor the AFT Pipeline Execution

AFT will run the CodePipeline for each account. For each account:

1. AFT assumes the `AWSAFTExecution` role in the target account
2. Terraform init runs with AFT's S3 backend (per-account state)
3. Terraform plan detects the import blocks and shows the import + changes
4. Terraform apply imports the resources into state and applies tag changes

**Monitor via:**

```bash
# Watch the CodePipeline status for each account
aws codepipeline list-pipelines \
  --query "pipelines[?contains(name, 'global-customizations')].name" \
  --output text

# Check a specific pipeline's status
aws codepipeline get-pipeline-state \
  --name "aft-global-customizations-<account-id>" \
  --query "stageStates[].{Stage:stageName,Status:latestExecution.status}"
```

### Step 4.4: Validate Config Rules Are Still Active

After AFT pipelines complete for all accounts:

```bash
# Run this against each account (script it)
for account_id in 111111111111 222222222222 333333333333; do
  echo "=== Account: $account_id ==="

  # Assume role into the account
  CREDS=$(aws sts assume-role \
    --role-arn "arn:aws:iam::${account_id}:role/AWSAFTExecution" \
    --role-session-name "config-rule-check")

  export AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r '.Credentials.AccessKeyId')
  export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r '.Credentials.SecretAccessKey')
  export AWS_SESSION_TOKEN=$(echo $CREDS | jq -r '.Credentials.SessionToken')

  # Check Config Rules
  aws configservice describe-config-rules \
    --config-rule-names "s3-bucket-encryption-check" "custom-mandatory-tag-check" \
    --query "ConfigRules[].{Name:ConfigRuleName,State:ConfigRuleState}" \
    --output table

  # Check compliance status
  aws configservice describe-compliance-by-config-rule \
    --config-rule-names "s3-bucket-encryption-check" "custom-mandatory-tag-check" \
    --query "ComplianceByConfigRules[].{Rule:ConfigRuleName,Compliance:Compliance.ComplianceType}" \
    --output table

  # Unset credentials
  unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
done
```

### Step 4.5: Remove Import Blocks

After all accounts are confirmed healthy:

```bash
# Remove the temporary import file
rm terraform/config_rules_import.tf

git add -A
git commit -m "chore: remove import blocks after successful migration

All accounts confirmed: Config Rules are ACTIVE and managed by Terraform.
StackSet 'OrgConfigRules' can now be deleted.

Refs: TP-2025-002"

git push origin main
```

AFT will run again — this time the plan should show **zero changes** for every
account (imports are already in state, and the import blocks are gone).

### Step 4.6: Delete the StackSet (Point of No Return)

Only do this after step 4.5 produces zero-change plans across all accounts.

```bash
# 1. First, delete all stack instances (but RETAIN resources)
#    RETAIN is critical — it removes CF's management of the resources
#    WITHOUT deleting the actual AWS resources.
aws cloudformation delete-stack-instances \
  --stack-set-name "OrgConfigRules" \
  --regions us-west-2 \
  --no-retain-stacks \
  --deployment-targets OrganizationalUnitIds=ou-xxxx-xxxxxxxx \
  --operation-preferences FailureToleranceCount=0,MaxConcurrentCount=5

# WAIT — this takes time. Monitor:
aws cloudformation describe-stack-set-operation \
  --stack-set-name "OrgConfigRules" \
  --operation-id <operation-id-from-above>

# ─────────────────────────────────────────────────
# CRITICAL DECISION POINT:
#
# --no-retain-stacks: Deletes the CF stack AND the resources.
#   DO NOT USE THIS — Terraform now manages these resources.
#
# --retain-stacks: Deletes the CF stack but KEEPS the resources.
#   USE THIS — it removes CF's ownership while leaving the
#   resources intact for Terraform to manage.
#
# CORRECTION: Use --retain-stacks (not --no-retain-stacks)
# ─────────────────────────────────────────────────

# CORRECT COMMAND:
aws cloudformation delete-stack-instances \
  --stack-set-name "OrgConfigRules" \
  --regions us-west-2 \
  --retain-stacks \
  --deployment-targets OrganizationalUnitIds=ou-xxxx-xxxxxxxx \
  --operation-preferences FailureToleranceCount=0,MaxConcurrentCount=5

# 2. After all stack instances are deleted, delete the StackSet itself
aws cloudformation delete-stack-set \
  --stack-set-name "OrgConfigRules"
```

---

## Phase 5: Post-Migration Verification Checklist

Run this checklist for every account after the full migration:

```
□ Config Rule "s3-bucket-encryption-check" is ACTIVE
□ Config Rule "custom-mandatory-tag-check" is ACTIVE
□ Lambda function "config-rule-tag-validation" is running
□ Compliance evaluations are current (check last evaluation timestamp)
□ Terraform state shows all 7 resources with no pending changes
□ CloudFormation stack instance is deleted (retained resources)
□ Tags include "ManagedBy=terraform" and "Source=aft-global-customizations"
□ No duplicate Config Rules exist in the account
```

---

## Common Pitfalls and How to Avoid Them

### Pitfall 1: Attribute Drift Between CF and TF

When you import, `terraform plan` will often show changes even if you think
your Terraform matches the StackSet template exactly. Common differences:

| What Drifts | Why | Fix |
|---|---|---|
| `maximum_execution_frequency` casing | CF uses `TwentyFour_Hours`, TF uses `TwentyFour_Hours` (same, but check) | Match exact API value |
| `input_parameters` ordering | CF and TF may serialize JSON keys in different order | Use `jsonencode()` with sorted keys |
| Tags | CF may not have `ManagedBy` tag; TF adds it | Expected — this is a safe change |
| Lambda `source_code_hash` | Inline ZipFile vs. packaged zip produce different hashes | Expected on first apply; Lambda will update in-place |
| IAM role path | CF defaults to `/`, TF might not set it | Explicitly set `path = "/"` in TF |

### Pitfall 2: Lambda Code Hash Mismatch

The biggest risk with the custom rule migration. The Lambda code was inline in
CloudFormation (ZipFile). When you package it as a zip in Terraform, the hash
will be different. On the first apply after import, Terraform will UPDATE the
Lambda function in-place. This is safe (no downtime) but triggers a brief
redeployment.

To minimize risk: ensure the Python code in `index.py` is byte-for-byte
identical to what was in the CloudFormation template. No extra whitespace,
no reformatting.

### Pitfall 3: AFT State Isolation

AFT maintains separate Terraform state per account. When you push to
`aft-global-customizations`, each account's pipeline runs independently.
This means:

- Account A's import might succeed while Account B's fails
- You need to monitor EVERY account's pipeline, not just one
- A failure in one account does NOT roll back other accounts

Build the validation script from Step 4.4 to check all accounts
programmatically.

### Pitfall 4: --no-retain-stacks vs --retain-stacks

This is the most dangerous command in the entire migration. Using the wrong
flag will DELETE your Config Rules in every account.

**`--retain-stacks`** = "Delete the CloudFormation stack but keep the AWS resources"
→ THIS IS WHAT YOU WANT

**`--no-retain-stacks`** = "Delete the CloudFormation stack AND destroy all resources"
→ NEVER USE THIS during migration

---

## Appendix: References

1. **AFT Global Customizations structure** — Terraform configs go under the
   `terraform/` directory; AFT provides backend and provider configs via Jinja
   templates at apply time. See
   [AWS docs on account customizations](https://docs.aws.amazon.com/controltower/latest/userguide/aft-account-customization-options.html)
   and [HashiCorp AFT tutorial](https://developer.hashicorp.com/terraform/tutorials/aws/aws-control-tower-aft).

2. **`aws_config_config_rule` Terraform resource** — Config Rules are imported
   using the rule name as the ID. See
   [Terraform Registry: aws_config_config_rule](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/config_config_rule.html).

3. **Terraform import blocks (v1.5+)** — Declarative import blocks allow
   importing existing resources as part of the normal plan/apply workflow
   without manual CLI commands. See
   [HashiCorp import documentation](https://developer.hashicorp.com/terraform/cli/import/usage).

4. **Terraform state import best practices** — The import command only updates
   state, not configuration. Starting in Terraform 1.5, import blocks combined
   with `terraform plan -generate-config-out` can auto-generate configuration.
   See [Spacelift import guide](https://spacelift.io/blog/importing-exisiting-infrastructure-into-terraform).

5. **AFT architecture and pipeline flow** — Account requests flow through
   CodePipeline with global customizations applied before account-specific
   customizations. See
   [AWS AFT architecture docs](https://docs.aws.amazon.com/controltower/latest/userguide/aft-architecture.html).

6. **CloudFormation StackSet deletion with retain** — `delete-stack-instances`
   with `--retain-stacks` removes CloudFormation management while preserving
   the underlying AWS resources. This is essential for handoff to Terraform.
   See AWS CloudFormation CLI reference.

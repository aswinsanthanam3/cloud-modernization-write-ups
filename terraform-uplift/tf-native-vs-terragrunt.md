# Terragrunt vs Native Terraform: Lambda Function — Enterprise Example

This document compares two approaches to managing a Lambda function across
dev/staging/prod environments with real-world enterprise dependencies.

---

## What We're Deploying

A Lambda function that:
- Processes SQS messages from a payment queue
- Runs inside a VPC (private subnets, needs NAT to reach external APIs)
- Has an IAM execution role with least-privilege policies
- Writes to a DynamoDB table
- Sends metrics to CloudWatch with custom alarms
- Has environment-specific config (memory, timeout, concurrency, VPC CIDR)
- Uses a Lambda Layer for shared libraries
- Has KMS encryption on environment variables
- Has mandatory org tags from CI metadata

**Dependency chain:**

```
VPC (subnets, NAT, security groups)
    └── Lambda Function
            ├── IAM Role + Policies
            ├── SQS Queue (event source mapping)
            ├── DynamoDB Table (write access)
            ├── KMS Key (env var encryption)
            ├── CloudWatch Log Group
            ├── CloudWatch Alarms
            └── Lambda Layer
```

---

## Approach 1: Terragrunt

### Directory Structure

```
infra/
├── terragrunt.hcl                          # Root config
├── _env/
│   └── lambda-payment-processor.hcl        # Common inputs template
│
├── dev/
│   ├── env.hcl                             # Environment-level vars
│   ├── vpc/
│   │   └── terragrunt.hcl
│   ├── kms/
│   │   └── terragrunt.hcl
│   ├── dynamodb-payments/
│   │   └── terragrunt.hcl
│   ├── sqs-payments/
│   │   └── terragrunt.hcl
│   ├── lambda-layer-shared/
│   │   └── terragrunt.hcl
│   └── lambda-payment-processor/
│       └── terragrunt.hcl                  # <-- This is what developers touch
│
├── staging/
│   ├── env.hcl
│   ├── vpc/
│   │   └── terragrunt.hcl
│   ├── kms/
│   │   └── terragrunt.hcl
│   ├── dynamodb-payments/
│   │   └── terragrunt.hcl
│   ├── sqs-payments/
│   │   └── terragrunt.hcl
│   ├── lambda-layer-shared/
│   │   └── terragrunt.hcl
│   └── lambda-payment-processor/
│       └── terragrunt.hcl
│
└── prod/
    ├── env.hcl
    ├── vpc/
    │   └── terragrunt.hcl
    ├── kms/
    │   └── terragrunt.hcl
    ├── dynamodb-payments/
    │   └── terragrunt.hcl
    ├── sqs-payments/
    │   └── terragrunt.hcl
    ├── lambda-layer-shared/
    │   └── terragrunt.hcl
    └── lambda-payment-processor/
        └── terragrunt.hcl
```

That's **21 terragrunt.hcl files** for one feature across three environments.

### Root Config

```hcl
# infra/terragrunt.hcl

locals {
  env_vars     = read_terragrunt_config(find_in_parent_folders("env.hcl"))
  env          = local.env_vars.locals.environment
  account_id   = local.env_vars.locals.aws_account_id
  region       = local.env_vars.locals.aws_region
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "aws" {
  region = "${local.region}"

  assume_role {
    role_arn = "arn:aws:iam::${local.account_id}:role/terraform-deploy"
  }

  default_tags {
    tags = {
      Environment = "${local.env}"
      ManagedBy   = "terraform"
      Repository  = "${get_env("CI_PROJECT_PATH", "local")}"
      Pipeline    = "${get_env("CI_PIPELINE_ID", "manual")}"
      CostCenter  = "${local.env_vars.locals.cost_center}"
    }
  }
}
EOF
}

remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket         = "tfstate-${local.env}-${local.account_id}"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = local.region
    dynamodb_table = "tfstate-lock-${local.env}"
    encrypt        = true
  }
}
```

### Environment Config

```hcl
# infra/dev/env.hcl

locals {
  environment    = "dev"
  aws_account_id = "111111111111"
  aws_region     = "us-west-2"
  cost_center    = "CC-1234"
  vpc_cidr       = "10.0.0.0/16"
}
```

```hcl
# infra/prod/env.hcl

locals {
  environment    = "prod"
  aws_account_id = "333333333333"
  aws_region     = "us-west-2"
  cost_center    = "CC-1234"
  vpc_cidr       = "10.2.0.0/16"
}
```

### The Lambda Module (Where the Dependency Pain Starts)

```hcl
# infra/dev/lambda-payment-processor/terragrunt.hcl

include "root" {
  path = find_in_parent_folders()
}

include "env" {
  path   = "${get_terragrunt_dir()}/../../_env/lambda-payment-processor.hcl"
  expose = true
}

terraform {
  source = "git::https://gitlab.internal/infra/modules/lambda.git?ref=v3.2.1"
}

# ──────────────────────────────────────────────────────────
# HERE IS THE DEPENDENCY PROBLEM
#
# Each 'dependency' block requires that module's state to be
# accessible at plan time. If the VPC hasn't been applied yet,
# or its state is locked, or the state bucket permissions are
# wrong, this module CANNOT plan at all.
#
# You cannot test this Lambda module in isolation.
# ──────────────────────────────────────────────────────────

dependency "vpc" {
  config_path = "../vpc"

  # mock_outputs are used when the dependency hasn't been applied yet
  # but these are FAKE values that may not match real output shapes
  mock_outputs = {
    private_subnet_ids = ["subnet-mock1", "subnet-mock2"]
    vpc_id             = "vpc-mock"
    lambda_sg_id       = "sg-mock"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

dependency "kms" {
  config_path = "../kms"
  mock_outputs = {
    key_arn = "arn:aws:kms:us-west-2:111111111111:key/mock-key-id"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

dependency "dynamodb" {
  config_path = "../dynamodb-payments"
  mock_outputs = {
    table_arn  = "arn:aws:dynamodb:us-west-2:111111111111:table/mock"
    table_name = "mock-table"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

dependency "sqs" {
  config_path = "../sqs-payments"
  mock_outputs = {
    queue_arn = "arn:aws:sqs:us-west-2:111111111111:mock-queue"
    queue_url = "https://sqs.us-west-2.amazonaws.com/111111111111/mock-queue"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

dependency "lambda_layer" {
  config_path = "../lambda-layer-shared"
  mock_outputs = {
    layer_arn = "arn:aws:lambda:us-west-2:111111111111:layer:mock:1"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

inputs = {
  function_name = "payment-processor"
  runtime       = "python3.12"
  handler       = "app.handler"
  timeout       = 30
  memory_size   = 256

  # ── Inputs wired from dependencies ──
  vpc_subnet_ids         = dependency.vpc.outputs.private_subnet_ids
  vpc_security_group_ids = [dependency.vpc.outputs.lambda_sg_id]
  kms_key_arn            = dependency.kms.outputs.key_arn
  dynamodb_table_arn     = dependency.dynamodb.outputs.table_arn
  dynamodb_table_name    = dependency.dynamodb.outputs.table_name
  sqs_queue_arn          = dependency.sqs.outputs.queue_arn
  sqs_queue_url          = dependency.sqs.outputs.queue_url
  lambda_layer_arns      = [dependency.lambda_layer.outputs.layer_arn]

  # ── Environment-specific overrides ──
  reserved_concurrent_executions = 10    # dev: low concurrency
  environment_variables = {
    STAGE           = "dev"
    LOG_LEVEL       = "DEBUG"
    TABLE_NAME      = dependency.dynamodb.outputs.table_name
    QUEUE_URL       = dependency.sqs.outputs.queue_url
    ENABLE_TRACING  = "true"
  }

  # ── Alarm thresholds (dev: relaxed) ──
  error_alarm_threshold          = 10
  duration_alarm_threshold_ms    = 5000
  throttle_alarm_threshold       = 5
}
```

### What the Prod Version Looks Like

```hcl
# infra/prod/lambda-payment-processor/terragrunt.hcl

include "root" {
  path = find_in_parent_folders()
}

include "env" {
  path   = "${get_terragrunt_dir()}/../../_env/lambda-payment-processor.hcl"
  expose = true
}

terraform {
  source = "git::https://gitlab.internal/infra/modules/lambda.git?ref=v3.2.1"
}

# Same five dependency blocks, pointing to ../vpc, ../kms, etc.
# Exact same structure. Only the inputs below change.

dependency "vpc" {
  config_path = "../vpc"
  mock_outputs = {
    private_subnet_ids = ["subnet-mock1", "subnet-mock2"]
    vpc_id             = "vpc-mock"
    lambda_sg_id       = "sg-mock"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

dependency "kms" {
  config_path = "../kms"
  mock_outputs = {
    key_arn = "arn:aws:kms:us-west-2:333333333333:key/mock-key-id"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

dependency "dynamodb" {
  config_path = "../dynamodb-payments"
  mock_outputs = {
    table_arn  = "arn:aws:dynamodb:us-west-2:333333333333:table/mock"
    table_name = "mock-table"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

dependency "sqs" {
  config_path = "../sqs-payments"
  mock_outputs = {
    queue_arn = "arn:aws:sqs:us-west-2:333333333333:mock-queue"
    queue_url = "https://sqs.us-west-2.amazonaws.com/333333333333/mock-queue"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

dependency "lambda_layer" {
  config_path = "../lambda-layer-shared"
  mock_outputs = {
    layer_arn = "arn:aws:lambda:us-west-2:333333333333:layer:mock:1"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

inputs = {
  function_name = "payment-processor"
  runtime       = "python3.12"
  handler       = "app.handler"
  timeout       = 60               # prod: higher timeout
  memory_size   = 1024             # prod: 4x memory

  vpc_subnet_ids         = dependency.vpc.outputs.private_subnet_ids
  vpc_security_group_ids = [dependency.vpc.outputs.lambda_sg_id]
  kms_key_arn            = dependency.kms.outputs.key_arn
  dynamodb_table_arn     = dependency.dynamodb.outputs.table_arn
  dynamodb_table_name    = dependency.dynamodb.outputs.table_name
  sqs_queue_arn          = dependency.sqs.outputs.queue_arn
  sqs_queue_url          = dependency.sqs.outputs.queue_url
  lambda_layer_arns      = [dependency.lambda_layer.outputs.layer_arn]

  reserved_concurrent_executions = 500  # prod: high concurrency

  environment_variables = {
    STAGE           = "prod"
    LOG_LEVEL       = "WARN"            # prod: less noise
    TABLE_NAME      = dependency.dynamodb.outputs.table_name
    QUEUE_URL       = dependency.sqs.outputs.queue_url
    ENABLE_TRACING  = "true"
  }

  # prod: tight alarm thresholds
  error_alarm_threshold          = 3
  duration_alarm_threshold_ms    = 3000
  throttle_alarm_threshold       = 2
}
```

### Why Developers Hate This

Look at the two files above. Here is what goes wrong in practice:

**1. Dependency blocks are copy-pasted across every environment.**
The five `dependency` blocks (vpc, kms, dynamodb, sqs, lambda_layer) are identical in
structure across dev/staging/prod. Only the mock account IDs change. That is 5 blocks
× 3 environments = 15 dependency declarations for ONE Lambda function. Each one has
to be maintained. Each one can drift.

**2. Mock outputs are lies that hide real failures.**
The `mock_outputs` blocks let you run `terragrunt plan` even when the dependency
hasn't been applied yet. Sounds helpful — but the mock values don't validate against
the real output shape. If the VPC module changes its output name from
`private_subnet_ids` to `private_subnets`, the mock still passes but the real apply
explodes. You've traded a clear error for a delayed one.

**3. You cannot plan the Lambda without ALL dependencies being accessible.**
Try running `terragrunt plan` on the Lambda module. Terragrunt will:
  - Parse all 5 dependency blocks
  - Attempt to read the state file of each dependency
  - Fail if ANY of those states are locked, missing, or in a different account
    you don't have access to

In practice this means: a developer working on the Lambda function cannot test
their changes unless the VPC, KMS, DynamoDB, SQS, and Layer modules have all been
successfully applied AND their state files are readable. On a Monday morning when
someone else is mid-apply on the VPC, you are blocked.

**4. `terragrunt run-all apply` creates terrifying blast radius.**
Running from the `dev/` directory:
```
cd infra/dev
terragrunt run-all apply
```
This will apply VPC, KMS, DynamoDB, SQS, Layer, AND Lambda in dependency order.
One command touching six independent pieces of infrastructure. If it fails
mid-way (say, at DynamoDB), Lambda hasn't been applied yet but SQS has, and now
your state is partially applied. Rollback is manual.

**5. Error messages point to generated files, not your code.**
When `terraform plan` fails inside Terragrunt, the error says something like:
```
Error: Invalid value for variable "vpc_subnet_ids"

  on /home/dev/.terragrunt-cache/abc123def456/modules/lambda/variables.tf line 42
```
That path is a temp directory Terragrunt generated. The developer has to mentally
map this back to their `terragrunt.hcl` dependency wiring. There is no IDE support
for this mapping.

**6. Version upgrades require touching every environment.**
To bump the Lambda module from v3.2.1 to v3.3.0, you change the `source` line in
every environment's `terragrunt.hcl`:
```
dev/lambda-payment-processor/terragrunt.hcl
staging/lambda-payment-processor/terragrunt.hcl
prod/lambda-payment-processor/terragrunt.hcl
```
In theory you can use `include` blocks to share the source. In practice, most
teams end up with per-environment overrides that break the sharing.

---

## Approach 2: Native Terraform (What Good Looks Like)

### Directory Structure

```
infra/
├── modules/                            # Your org's composition modules (optional)
│   └── payment-processor/              # Groups Lambda + IAM + alarms as one unit
│       ├── main.tf
│       ├── iam.tf
│       ├── alarms.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── versions.tf
│
├── environments/
│   ├── dev/
│   │   ├── main.tf                     # <-- THIS is what developers touch
│   │   ├── data.tf                     # data sources for VPC, KMS, etc.
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── backend.tf
│   ├── staging/
│   │   ├── main.tf
│   │   ├── data.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── backend.tf
│   └── prod/
│       ├── main.tf
│       ├── data.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── backend.tf
│
└── policies/                           # OPA/Conftest policies
    ├── lambda/
    │   ├── vpc_required.rego
    │   ├── encryption_required.rego
    │   └── concurrency_limits.rego
    └── common/
        └── mandatory_tags.rego
```

Fewer files, flatter structure, and every file is standard Terraform.

### The Dev Environment

```hcl
# environments/dev/backend.tf

terraform {
  required_version = ">= 1.6.0"

  backend "s3" {
    bucket         = "tfstate-dev-111111111111"
    key            = "payment-processor/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "tfstate-lock-dev"
    encrypt        = true
  }
}
```

```hcl
# environments/dev/data.tf
#
# THIS IS THE KEY DIFFERENCE FROM TERRAGRUNT.
#
# Instead of 'dependency' blocks that require another module's STATE,
# we use 'data' sources that query AWS directly.
#
# Why this is better:
# - No state file coupling. If someone is mid-apply on the VPC,
#   you can still plan your Lambda — data sources read the LIVE state
#   of AWS, not another module's state file.
# - No mock outputs. You get real values or a clear error.
# - Standard Terraform. Every developer knows data sources.

data "aws_vpc" "main" {
  tags = {
    Environment = "dev"
    ManagedBy   = "terraform"
  }
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
  tags = {
    Tier = "private"
  }
}

data "aws_security_group" "lambda" {
  vpc_id = data.aws_vpc.main.id
  tags = {
    Purpose = "lambda-egress"
  }
}

data "aws_kms_alias" "lambda" {
  name = "alias/lambda-env-encryption"
}

data "aws_dynamodb_table" "payments" {
  name = "payments-dev"
}

data "aws_sqs_queue" "payments" {
  name = "payments-processor-dev"
}

data "aws_lambda_layer_version" "shared" {
  layer_name = "shared-libs"
}
```

```hcl
# environments/dev/main.tf

locals {
  environment = "dev"
  team        = "payments"
}

module "payment_processor" {
  source = "../../modules/payment-processor"

  # ── Function config ──
  function_name = "payment-processor"
  runtime       = "python3.12"
  handler       = "app.handler"
  timeout       = 30
  memory_size   = 256

  # ── VPC config (from data sources — no state coupling) ──
  vpc_subnet_ids         = data.aws_subnets.private.ids
  vpc_security_group_ids = [data.aws_security_group.lambda.id]

  # ── Dependencies (from data sources) ──
  kms_key_arn        = data.aws_kms_alias.lambda.target_key_arn
  dynamodb_table_arn = data.aws_dynamodb_table.payments.arn
  sqs_queue_arn      = data.aws_sqs_queue.payments.arn
  lambda_layer_arns  = [data.aws_lambda_layer_version.shared.arn]

  # ── Concurrency ──
  reserved_concurrent_executions = 10

  # ── Environment variables ──
  environment_variables = {
    STAGE          = local.environment
    LOG_LEVEL      = "DEBUG"
    TABLE_NAME     = data.aws_dynamodb_table.payments.name
    QUEUE_URL      = data.aws_sqs_queue.payments.url
    ENABLE_TRACING = "true"
  }

  # ── Alarm thresholds (relaxed for dev) ──
  error_alarm_threshold       = 10
  duration_alarm_threshold_ms = 5000
  throttle_alarm_threshold    = 5

  # ── Tags ──
  tags = {
    Team        = local.team
    Environment = local.environment
  }
}
```

### The Prod Environment

```hcl
# environments/prod/backend.tf

terraform {
  required_version = ">= 1.6.0"

  backend "s3" {
    bucket         = "tfstate-prod-333333333333"
    key            = "payment-processor/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "tfstate-lock-prod"
    encrypt        = true
  }
}
```

```hcl
# environments/prod/data.tf

data "aws_vpc" "main" {
  tags = {
    Environment = "prod"
    ManagedBy   = "terraform"
  }
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
  tags = {
    Tier = "private"
  }
}

data "aws_security_group" "lambda" {
  vpc_id = data.aws_vpc.main.id
  tags = {
    Purpose = "lambda-egress"
  }
}

data "aws_kms_alias" "lambda" {
  name = "alias/lambda-env-encryption"
}

data "aws_dynamodb_table" "payments" {
  name = "payments-prod"
}

data "aws_sqs_queue" "payments" {
  name = "payments-processor-prod"
}

data "aws_lambda_layer_version" "shared" {
  layer_name = "shared-libs"
}
```

```hcl
# environments/prod/main.tf

locals {
  environment = "prod"
  team        = "payments"
}

module "payment_processor" {
  source = "../../modules/payment-processor"

  function_name = "payment-processor"
  runtime       = "python3.12"
  handler       = "app.handler"
  timeout       = 60               # prod: higher timeout
  memory_size   = 1024             # prod: 4x memory

  vpc_subnet_ids         = data.aws_subnets.private.ids
  vpc_security_group_ids = [data.aws_security_group.lambda.id]

  kms_key_arn        = data.aws_kms_alias.lambda.target_key_arn
  dynamodb_table_arn = data.aws_dynamodb_table.payments.arn
  sqs_queue_arn      = data.aws_sqs_queue.payments.arn
  lambda_layer_arns  = [data.aws_lambda_layer_version.shared.arn]

  reserved_concurrent_executions = 500    # prod: high concurrency

  environment_variables = {
    STAGE          = local.environment
    LOG_LEVEL      = "WARN"               # prod: less noise
    TABLE_NAME     = data.aws_dynamodb_table.payments.name
    QUEUE_URL      = data.aws_sqs_queue.payments.url
    ENABLE_TRACING = "true"
  }

  # prod: tight thresholds
  error_alarm_threshold       = 3
  duration_alarm_threshold_ms = 3000
  throttle_alarm_threshold    = 2

  tags = {
    Team        = local.team
    Environment = local.environment
  }
}
```

### The Composition Module (What Your Wrapper SHOULD Have Been)

```hcl
# modules/payment-processor/main.tf
#
# This is a COMPOSITION module — it groups related resources
# into one logical unit. Unlike a wrapper, it adds real value
# by encoding the relationship between Lambda, IAM, SQS event
# source mapping, and CloudWatch alarms.

resource "aws_lambda_function" "this" {
  function_name = "${var.function_name}-${var.tags.Environment}"
  runtime       = var.runtime
  handler       = var.handler
  timeout       = var.timeout
  memory_size   = var.memory_size
  role          = aws_iam_role.lambda.arn

  filename         = var.deployment_package
  source_code_hash = filebase64sha256(var.deployment_package)

  reserved_concurrent_executions = var.reserved_concurrent_executions

  layers = var.lambda_layer_arns

  vpc_config {
    subnet_ids         = var.vpc_subnet_ids
    security_group_ids = var.vpc_security_group_ids
  }

  environment {
    variables = var.environment_variables
  }

  kms_key_arn = var.kms_key_arn

  tracing_config {
    mode = "Active"
  }

  tags = var.tags
}

# SQS event source mapping — this is the kind of wiring that
# belongs in a composition module because the batch size, max
# batching window, and error handling are tightly coupled to
# the Lambda's purpose.
resource "aws_lambda_event_source_mapping" "sqs" {
  event_source_arn                   = var.sqs_queue_arn
  function_name                      = aws_lambda_function.this.arn
  batch_size                         = var.sqs_batch_size
  maximum_batching_window_in_seconds = var.sqs_batching_window
  function_response_types            = ["ReportBatchItemFailures"]

  scaling_config {
    maximum_concurrency = var.reserved_concurrent_executions
  }
}

# CloudWatch log group with retention — automatically created
# so developers don't forget to set retention (which costs money)
resource "aws_cloudwatch_log_group" "this" {
  name              = "/aws/lambda/${aws_lambda_function.this.function_name}"
  retention_in_days = var.log_retention_days
  kms_key_id        = var.kms_key_arn
  tags              = var.tags
}
```

```hcl
# modules/payment-processor/iam.tf
#
# IAM is where composition modules earn their keep.
# Developers should never have to write IAM policies for Lambda.
# The module encodes least-privilege based on the resources passed in.

resource "aws_iam_role" "lambda" {
  name = "${var.function_name}-${var.tags.Environment}-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })

  tags = var.tags
}

# VPC access (ENI management)
resource "aws_iam_role_policy_attachment" "vpc" {
  role       = aws_iam_role.lambda.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}

# CloudWatch Logs
resource "aws_iam_role_policy" "logs" {
  name = "cloudwatch-logs"
  role = aws_iam_role.lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ]
      Resource = "${aws_cloudwatch_log_group.this.arn}:*"
    }]
  })
}

# DynamoDB — scoped to the specific table
resource "aws_iam_role_policy" "dynamodb" {
  name = "dynamodb-access"
  role = aws_iam_role.lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:GetItem",
        "dynamodb:Query"
      ]
      Resource = [
        var.dynamodb_table_arn,
        "${var.dynamodb_table_arn}/index/*"
      ]
    }]
  })
}

# SQS — scoped to the specific queue
resource "aws_iam_role_policy" "sqs" {
  name = "sqs-access"
  role = aws_iam_role.lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ]
      Resource = var.sqs_queue_arn
    }]
  })
}

# KMS — for decrypting environment variables
resource "aws_iam_role_policy" "kms" {
  name = "kms-decrypt"
  role = aws_iam_role.lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "kms:Decrypt"
      ]
      Resource = var.kms_key_arn
    }]
  })
}

# X-Ray tracing
resource "aws_iam_role_policy_attachment" "xray" {
  role       = aws_iam_role.lambda.name
  policy_arn = "arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess"
}
```

```hcl
# modules/payment-processor/alarms.tf

resource "aws_cloudwatch_metric_alarm" "errors" {
  alarm_name          = "${var.function_name}-${var.tags.Environment}-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  period              = 300
  statistic           = "Sum"
  threshold           = var.error_alarm_threshold
  treat_missing_data  = "notBreaching"

  dimensions = {
    FunctionName = aws_lambda_function.this.function_name
  }

  alarm_actions = var.alarm_sns_topic_arns
  ok_actions    = var.alarm_sns_topic_arns

  tags = var.tags
}

resource "aws_cloudwatch_metric_alarm" "duration" {
  alarm_name          = "${var.function_name}-${var.tags.Environment}-duration"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "Duration"
  namespace           = "AWS/Lambda"
  period              = 300
  statistic           = "p99"
  threshold           = var.duration_alarm_threshold_ms
  treat_missing_data  = "notBreaching"

  dimensions = {
    FunctionName = aws_lambda_function.this.function_name
  }

  alarm_actions = var.alarm_sns_topic_arns
  ok_actions    = var.alarm_sns_topic_arns

  tags = var.tags
}

resource "aws_cloudwatch_metric_alarm" "throttles" {
  alarm_name          = "${var.function_name}-${var.tags.Environment}-throttles"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "Throttles"
  namespace           = "AWS/Lambda"
  period              = 60
  statistic           = "Sum"
  threshold           = var.throttle_alarm_threshold
  treat_missing_data  = "notBreaching"

  dimensions = {
    FunctionName = aws_lambda_function.this.function_name
  }

  alarm_actions = var.alarm_sns_topic_arns
  ok_actions    = var.alarm_sns_topic_arns

  tags = var.tags
}
```

```hcl
# modules/payment-processor/variables.tf

# ── Function config ──
variable "function_name" {
  description = "Name of the Lambda function"
  type        = string
}

variable "runtime" {
  description = "Lambda runtime"
  type        = string
  default     = "python3.12"
}

variable "handler" {
  description = "Function entrypoint"
  type        = string
  default     = "app.handler"
}

variable "timeout" {
  description = "Function timeout in seconds"
  type        = number
  default     = 30

  validation {
    condition     = var.timeout >= 1 && var.timeout <= 900
    error_message = "Timeout must be between 1 and 900 seconds."
  }
}

variable "memory_size" {
  description = "Function memory in MB"
  type        = number
  default     = 256

  validation {
    condition     = var.memory_size >= 128 && var.memory_size <= 10240
    error_message = "Memory must be between 128 and 10240 MB."
  }
}

variable "deployment_package" {
  description = "Path to the Lambda deployment package (.zip)"
  type        = string
  default     = "lambda.zip"
}

variable "reserved_concurrent_executions" {
  description = "Reserved concurrent executions (-1 for unreserved)"
  type        = number
  default     = -1
}

# ── VPC config ──
variable "vpc_subnet_ids" {
  description = "List of VPC subnet IDs for Lambda"
  type        = list(string)
}

variable "vpc_security_group_ids" {
  description = "List of security group IDs for Lambda"
  type        = list(string)
}

# ── Dependencies ──
variable "kms_key_arn" {
  description = "KMS key ARN for environment variable encryption"
  type        = string
}

variable "dynamodb_table_arn" {
  description = "DynamoDB table ARN (for IAM policy scoping)"
  type        = string
}

variable "sqs_queue_arn" {
  description = "SQS queue ARN (for event source mapping and IAM)"
  type        = string
}

variable "lambda_layer_arns" {
  description = "List of Lambda layer ARNs to attach"
  type        = list(string)
  default     = []
}

# ── SQS event source config ──
variable "sqs_batch_size" {
  description = "SQS batch size for event source mapping"
  type        = number
  default     = 10
}

variable "sqs_batching_window" {
  description = "Max batching window in seconds"
  type        = number
  default     = 5
}

# ── Environment ──
variable "environment_variables" {
  description = "Environment variables for the Lambda function"
  type        = map(string)
  default     = {}
}

# ── Observability ──
variable "log_retention_days" {
  description = "CloudWatch log retention in days"
  type        = number
  default     = 30
}

variable "error_alarm_threshold" {
  description = "Error count threshold for CloudWatch alarm"
  type        = number
  default     = 5
}

variable "duration_alarm_threshold_ms" {
  description = "p99 duration threshold in ms for CloudWatch alarm"
  type        = number
  default     = 3000
}

variable "throttle_alarm_threshold" {
  description = "Throttle count threshold for CloudWatch alarm"
  type        = number
  default     = 3
}

variable "alarm_sns_topic_arns" {
  description = "SNS topic ARNs for alarm notifications"
  type        = list(string)
  default     = []
}

# ── Tags ──
variable "tags" {
  description = "Resource tags"
  type        = map(string)
}
```

---

## Side-by-Side Summary

| Dimension                       | Terragrunt                              | Native Terraform                          |
|---------------------------------|-----------------------------------------|-------------------------------------------|
| **Files for 3 environments**    | 21+ terragrunt.hcl files               | 9 .tf files + 1 shared module             |
| **Dependency resolution**       | `dependency` blocks read other states   | `data` sources query AWS directly         |
| **Can plan in isolation?**      | No — needs all dependency states        | Yes — data sources read live AWS state    |
| **Mock values needed?**         | Yes — for every dependency per env      | No — real values from AWS API             |
| **IDE support**                 | Minimal (no language server)            | Full (terraform-ls, autocomplete, go-to)  |
| **Error messages**              | Point to temp cache directory           | Point to your actual .tf files            |
| **Version upgrade scope**       | Touch every environment's .hcl          | Update module source in each env's main.tf (or use CLI to generate) |
| **Learning curve**              | Terraform + Terragrunt HCL dialect      | Terraform only                            |
| **Blast radius of run-all**     | Entire env directory applied at once    | N/A — you apply one directory at a time   |
| **State file coupling**         | Tight — modules depend on each other's state | Loose — each env is independent         |
| **Debugging**                   | Trace from cache path back to source    | Direct — errors reference your code       |
| **What the CLI factory replaces** | Root config, env.hcl, dependency wiring | backend.tf generation, data.tf generation, env-specific values |

---

## What the CLI Factory Does for Native Terraform

The CLI factory eliminates the remaining boilerplate in the native approach. Instead of
manually writing `backend.tf` and `data.tf` for each environment, the developer runs:

```bash
infra create lambda-payment-processor --team payments --env dev
```

And the CLI generates `backend.tf`, `data.tf`, and a scaffold `main.tf` with the right
module source, backend config, data sources, and tag defaults pre-filled. The developer
only needs to customize the inputs that are specific to their use case (memory, timeout,
concurrency, environment variables).

The DRY problem that Terragrunt solves is handled at generation time, not runtime.
The output is standard Terraform that any developer can read, debug, and modify.
# IaC Module Refactoring: Sprint Plan

**Duration:** 3 months (6 × 2-week sprints)  
**Team:** 1 developer  
**Pilot Module:** Lambda (pending usage data confirmation)  
**Sandbox Account:** Available for integration testing  

**Must-Have Deliverables:**
- CLI factory (`infra create`)
- Auto-generated module documentation
- Module testing (tftest + Terratest)
- Terragrunt removal for new modules

**Explicitly Deferred:**
- OPA/Conftest compliance gates (Prisma remains the safety net)
- Harness CD pipeline changes
- Existing module migration (post-3-month effort)

---

## Sprint 0: Pre-Work (Before Sprint 1 Starts)

**Owner:** Aswin (1–2 days of setup)

This isn't a full sprint — it's the prerequisite work to unblock the developer on day 1.

| Task | Detail |
|------|--------|
| Create `infra-cli` GitLab repo | Monorepo: CLI code, schemas, templates, policies, tests |
| Create `infra-modules-v2` GitLab repo | New module repo (clean break from existing 40+ wrapper repo) |
| Sandbox account access | Ensure the developer has IAM credentials for the sandbox account with permissions to create/destroy Lambda, IAM, VPC, SQS, DynamoDB, CloudWatch, KMS resources |
| Python environment | Python 3.11+, `pyproject.toml` with Typer, Jinja2, PyYAML, pytest, Ruff |
| Go environment | Go 1.21+ for Terratest (only needed Sprint 4) |
| Terraform version | Pin to >= 1.6.0 (needed for native `terraform test`) |
| Pull usage data | Query GitLab/Harness to confirm Lambda is the right pilot (or pick an alternative) |

**Exit criteria:** Developer can clone both repos, run `pytest` and `terraform validate` locally, and authenticate to the sandbox account.

---

## Sprint 1 (Weeks 1–2): Lambda Composition Module + Schema

**Goal:** Build the new Lambda composition module and its YAML schema from scratch. No CLI yet — just a clean, tested, documented module that works with native Terraform.

### Week 1: Module Development

| Day | Task | Output |
|-----|------|--------|
| 1 | Audit existing Lambda wrapper: catalog all inputs, hardcoded defaults, security policies, Prisma rules it satisfies | `audit/lambda-current-state.md` — list of every input, default, and hidden behavior |
| 2 | Design the new Lambda composition module's variable interface: what's exposed to developers vs. what's hardened by default | `modules/lambda-processor/variables.tf` (draft) |
| 3–4 | Implement the composition module: Lambda function + IAM role (least-privilege, scoped to inputs) + CloudWatch log group + CloudWatch alarms + SQS event source mapping | `modules/lambda-processor/main.tf`, `iam.tf`, `alarms.tf`, `outputs.tf` |
| 5 | Implement variable validations (timeout ranges, memory limits, naming patterns) + `versions.tf` with provider constraints | Complete `variables.tf` with validation blocks |

### Week 2: Schema + Documentation Generation

| Day | Task | Output |
|-----|------|--------|
| 1–2 | Write the YAML schema for the Lambda module: inputs (required/optional), hardened defaults (KMS, tags, tracing), backend reference, compliance policy references | `schemas/lambda-processor.yaml` |
| 3 | Build the doc generator: Python script that reads the YAML schema + `variables.tf` and produces a Markdown README with usage examples, input/output tables, and default values | `scripts/generate_docs.py` → `modules/lambda-processor/README.md` |
| 4 | Write a working example: `examples/lambda-processor/dev/` with `main.tf`, `data.tf`, `backend.tf` using native Terraform (no Terragrunt) against sandbox account | `examples/lambda-processor/dev/` directory |
| 5 | Manually deploy the example to sandbox: `terraform init && plan && apply`. Verify the Lambda, IAM role, log group, alarms, and event source mapping all create correctly. Document any issues. | Successful `terraform apply` in sandbox; `TESTING.md` with manual verification notes |

### Sprint 1 Deliverables

```
infra-modules-v2/
├── modules/
│   └── lambda-processor/
│       ├── main.tf              # Lambda + SQS event source mapping
│       ├── iam.tf               # Least-privilege IAM role + policies
│       ├── alarms.tf            # CloudWatch alarms (errors, duration, throttles)
│       ├── variables.tf         # Validated inputs with sane defaults
│       ├── outputs.tf           # Function ARN, role ARN, log group name
│       ├── versions.tf          # Provider + TF version constraints
│       └── README.md            # Auto-generated from schema
├── schemas/
│   └── lambda-processor.yaml    # YAML schema (the contract)
├── examples/
│   └── lambda-processor/
│       └── dev/
│           ├── main.tf          # Working example against sandbox
│           ├── data.tf          # VPC, KMS, SQS, DynamoDB data sources
│           └── backend.tf       # S3 backend for sandbox account
├── scripts/
│   └── generate_docs.py         # Schema → README generator
└── audit/
    └── lambda-current-state.md  # Audit of existing wrapper
```

**Sprint 1 Review Checkpoint:**
- [ ] Module deploys successfully in sandbox
- [ ] README is auto-generated and accurate
- [ ] Variable interface is clean (developer needs to set ~5–8 inputs, everything else has sane defaults)
- [ ] No Terragrunt dependency anywhere
- [ ] Aswin reviews and signs off on the variable interface design

---

## Sprint 2 (Weeks 3–4): CLI Factory Core

**Goal:** Build the `infra create` command that generates a complete, working Terraform configuration from the Lambda schema.

### Week 1: CLI Scaffolding + `infra create`

| Day | Task | Output |
|-----|------|--------|
| 1 | Set up CLI project structure: Typer app, subcommands, schema loader, config loader | `src/infra_cli/` package with `__main__.py`, `cli.py`, `schema.py`, `config.py` |
| 2 | Build schema loader: reads YAML schema, validates structure, exposes inputs/defaults/backend config | `src/infra_cli/schema.py` with `load_schema()`, `validate_inputs()` |
| 3 | Build environment config: reads `config/environments.yaml` (dev/staging/prod account IDs, state buckets, regions, assume roles) | `config/environments.yaml` + `src/infra_cli/config.py` |
| 4–5 | Build `infra create` command: accepts resource type + team + env, collects inputs interactively (with defaults from schema), validates, prepares context for rendering | `src/infra_cli/commands/create.py` — working command that collects all inputs |

### Week 2: Template Engine + `infra list` + `infra docs`

| Day | Task | Output |
|-----|------|--------|
| 1–2 | Build Jinja2 template engine: templates for `main.tf`, `data.tf`, `backend.tf`, `versions.tf`. Each template renders native Terraform (no Terragrunt). Security defaults injected from schema's `hardened_defaults`. | `templates/lambda-processor/` directory with `.tf.j2` templates |
| 3 | End-to-end test: run `infra create lambda-processor --team payments --env dev`, inspect generated files, run `terraform init && plan` against sandbox. Fix any rendering issues. | Generated Terraform that passes `terraform plan` |
| 4 | Build `infra list` (list available schemas) and `infra docs lambda-processor` (print auto-generated docs to terminal) | Two additional CLI commands |
| 5 | Write CLI tests: pytest suite covering schema loading, input validation, template rendering, file output. Test with valid inputs, missing required inputs, invalid patterns. | `tests/` directory with >= 80% coverage on CLI code |

### Sprint 2 Deliverables

```
infra-cli/
├── src/
│   └── infra_cli/
│       ├── __main__.py
│       ├── cli.py                    # Typer app entry point
│       ├── schema.py                 # YAML schema loader + validator
│       ├── config.py                 # Environment config loader
│       ├── renderer.py               # Jinja2 template engine
│       └── commands/
│           ├── create.py             # infra create <resource>
│           ├── list.py               # infra list
│           └── docs.py               # infra docs <resource>
├── config/
│   └── environments.yaml             # dev/staging/prod account config
├── templates/
│   └── lambda-processor/
│       ├── main.tf.j2               # Module invocation template
│       ├── data.tf.j2               # Data sources template
│       ├── backend.tf.j2            # S3 backend template
│       └── versions.tf.j2           # Provider constraints template
├── tests/
│   ├── test_schema.py
│   ├── test_renderer.py
│   ├── test_create_command.py
│   └── fixtures/                    # Sample schemas, expected outputs
├── pyproject.toml
└── README.md
```

**Sprint 2 Review Checkpoint:**
- [ ] `infra create lambda-processor --team payments --env dev` generates files that pass `terraform plan`
- [ ] `infra create lambda-processor --team payments --env prod` generates correct prod backend + account config
- [ ] `infra list` shows available schemas
- [ ] `infra docs lambda-processor` prints clean documentation
- [ ] CLI tests pass, >= 80% coverage
- [ ] Developer from an app team tries the CLI and provides feedback (hallway test)

---

## Sprint 3 (Weeks 5–6): Module Testing + CLI Polish

**Goal:** Add Terraform native tests and Terratest integration tests to the Lambda module. Polish CLI based on Sprint 2 feedback.

### Week 1: Terraform Native Tests (Plan-Based)

| Day | Task | Output |
|-----|------|--------|
| 1 | Write plan-based test assertions for the Lambda module: encryption enabled, IAM role created, log group has retention, alarms configured, VPC config present | `modules/lambda-processor/tests/lambda.tftest.hcl` |
| 2 | Write plan-based test for security hardening: KMS key on env vars, tracing enabled, public access blocked, mandatory tags present | `modules/lambda-processor/tests/security.tftest.hcl` |
| 3 | Write test helper module: generates random function names, provides mock VPC/subnet/SG/KMS/DynamoDB/SQS values for plan-time testing | `modules/lambda-processor/tests/setup/main.tf` |
| 4 | Run all tests locally: `terraform test`. Fix any failures. Ensure tests run in < 2 minutes (plan-only, no real resources). | All `.tftest.hcl` tests passing |
| 5 | Add test stage to GitLab CI for the module repo: `terraform init && terraform test` runs on every MR | `.gitlab-ci.yml` with `test` stage |

### Week 2: Terratest Integration Tests + CLI Feedback Iteration

| Day | Task | Output |
|-----|------|--------|
| 1–2 | Write Terratest integration test: deploys the Lambda example to sandbox, asserts function exists with correct config (runtime, memory, VPC, encryption), runs `defer terraform.Destroy()` | `tests/integration/lambda_test.go` |
| 3 | Set up nightly GitLab CI job for integration tests: runs Terratest against sandbox account on a schedule (not on every MR — too slow and expensive) | `.gitlab-ci.yml` with `integration-test` scheduled job |
| 4–5 | CLI polish based on Sprint 2 hallway test feedback: improve error messages, add `--dry-run` flag, add `--interactive` mode with guided prompts for optional inputs, add color output | Updated CLI code + tests |

### Sprint 3 Deliverables

```
# Added to infra-modules-v2:
modules/lambda-processor/
├── tests/
│   ├── lambda.tftest.hcl        # Plan-based: function config assertions
│   ├── security.tftest.hcl      # Plan-based: encryption, tags, tracing
│   └── setup/
│       └── main.tf              # Test helper (random names, mock values)

tests/
└── integration/
    ├── lambda_test.go           # Terratest: real deploy to sandbox
    ├── go.mod
    └── go.sum

.gitlab-ci.yml                   # Updated with test + nightly integration stages
```

**Sprint 3 Review Checkpoint:**
- [ ] `terraform test` passes in < 2 minutes (plan-only)
- [ ] Terratest deploys and destroys Lambda stack in sandbox successfully
- [ ] GitLab CI runs `terraform test` on every MR to the module repo
- [ ] Nightly integration test pipeline is green
- [ ] CLI `--dry-run` shows generated files without writing
- [ ] At least 1 app developer has successfully used the CLI to generate a Lambda config

---

## Sprint 4 (Weeks 7–8): Second Module + CLI Scaling

**Goal:** Prove the pattern scales by adding a second module. Based on usage data, this should be your second-highest-pain module (likely S3 or RDS — confirm with data).

### Week 1: Second Module (Assumed: S3)

| Day | Task | Output |
|-----|------|--------|
| 1 | Audit existing S3 wrapper: catalog inputs, defaults, Prisma rules | `audit/s3-current-state.md` |
| 2 | Determine module type: S3 is likely a **standalone module** (single resource + encryption + tags + public access block), not a composition module. Design accordingly — simpler variable interface than Lambda. | `modules/s3-bucket/variables.tf` (draft) |
| 3 | Implement S3 module: bucket + encryption + public access block + versioning + lifecycle rules + logging. Use upstream `aws_s3_bucket` resources directly (this is a standalone wrapper, not a composition). | `modules/s3-bucket/main.tf`, `variables.tf`, `outputs.tf` |
| 4 | Write YAML schema for S3 | `schemas/s3-bucket.yaml` |
| 5 | Write Jinja2 templates for S3, add S3 to CLI factory | `templates/s3-bucket/` directory |

### Week 2: Tests + Docs + Validation

| Day | Task | Output |
|-----|------|--------|
| 1 | Write `terraform test` plan-based assertions for S3: encryption, public access blocked, versioning, tags | `modules/s3-bucket/tests/s3.tftest.hcl` |
| 2 | Write Terratest integration test for S3 | `tests/integration/s3_test.go` |
| 3 | Auto-generate S3 README from schema | `modules/s3-bucket/README.md` |
| 4 | End-to-end validation: `infra create s3-bucket --team payments --env dev` → `terraform plan` → `terraform apply` in sandbox → verify → `terraform destroy` | Passing end-to-end flow |
| 5 | Refactor CLI internals: any patterns that emerged from adding the second module (template duplication, schema boilerplate, etc.) should be DRY-ed up now | Cleaner CLI codebase |

### Sprint 4 Review Checkpoint:
- [ ] S3 module deploys successfully in sandbox
- [ ] `infra create s3-bucket` works end-to-end
- [ ] `infra list` shows both `lambda-processor` and `s3-bucket`
- [ ] Both modules have plan-based tests + integration tests
- [ ] Both modules have auto-generated READMEs
- [ ] Pattern for adding new modules is clear and repeatable

---

## Sprint 5 (Weeks 9–10): Third Module + Developer Onboarding

**Goal:** Add a third module (confirm with data — likely RDS or VPC) and create the developer onboarding experience.

### Week 1: Third Module

| Day | Task | Output |
|-----|------|--------|
| 1 | Audit existing wrapper for the third module | `audit/{module}-current-state.md` |
| 2–3 | Implement module + schema + templates (pattern should be well-established by now — target 2 days, not 5) | `modules/{third-module}/`, `schemas/{third-module}.yaml`, `templates/{third-module}/` |
| 4 | Write tests (plan-based + Terratest) | `.tftest.hcl` + `_test.go` files |
| 5 | End-to-end validation in sandbox | Passing flow |

### Week 2: Developer Onboarding Package

| Day | Task | Output |
|-----|------|--------|
| 1–2 | Write the developer getting-started guide: install CLI, create first resource, understand generated files, run plan, iterate, commit, push to trigger CI | `docs/getting-started.md` |
| 3 | Write the module contributor guide: how to add a new module (schema format, template conventions, test requirements, doc generation, versioning) | `docs/contributing.md` |
| 4 | Build `infra init` command: sets up a new project directory with `.gitignore`, `README.md`, CI config template, and directory structure | `src/infra_cli/commands/init.py` |
| 5 | Run onboarding session: walk 2–3 app developers through the full flow. Collect feedback. Document friction points. | Feedback notes in `docs/onboarding-feedback.md` |

### Sprint 5 Review Checkpoint:
- [ ] Third module complete with tests and docs
- [ ] Developer getting-started guide is clear enough for self-service
- [ ] Contributor guide enables your team to add new modules without the pilot developer
- [ ] `infra init` scaffolds a working project
- [ ] 2–3 app developers have completed the onboarding flow successfully

---

## Sprint 6 (Weeks 11–12): Production Readiness + Handoff

**Goal:** Harden everything for production use, establish the cadence for adding remaining modules, and hand off to the team.

### Week 1: Production Hardening

| Day | Task | Output |
|-----|------|--------|
| 1 | CLI packaging: build distributable package (pip install from internal PyPI or GitLab package registry), add shell completion, version pinning | `pyproject.toml` with build config, `.gitlab-ci.yml` with publish stage |
| 2 | GitLab CI template: reusable `.gitlab-ci.yml` template that app teams include in their repos — runs `terraform validate`, `terraform fmt -check`, `terraform test`, `terraform plan` | `ci-templates/terraform-pipeline.yml` |
| 3 | Environment promotion test: generate Lambda config for dev → plan → apply → then generate for staging → plan → apply → then prod → plan only (no apply). Verify backend isolation, state separation, tag correctness across all three. | Documented promotion flow in `docs/environment-promotion.md` |
| 4 | Security review: verify all modules enforce KMS encryption, block public access where applicable, use least-privilege IAM, inject mandatory tags. Cross-reference against Prisma rule set. | `docs/security-compliance-matrix.md` |
| 5 | Edge case testing: test CLI with missing inputs, invalid team names, non-existent schemas, network errors. Verify error messages are helpful. | Updated tests + error handling |

### Week 2: Handoff + Roadmap

| Day | Task | Output |
|-----|------|--------|
| 1 | Write the module migration playbook: step-by-step process for converting any of the remaining 37+ old wrapper modules to the new pattern (audit → classify → implement → test → doc → deprecate old) | `docs/module-migration-playbook.md` |
| 2 | Estimate remaining work: based on Sprint 4/5 velocity, project how long it will take to migrate the remaining modules. Classify each as standalone/composition/eliminate. | `docs/remaining-modules-estimate.md` with timeline |
| 3 | Knowledge transfer session: the pilot developer walks the team through the codebase, CLI architecture, test patterns, schema format, and template conventions | Recorded session + notes |
| 4 | Write the ADR: formal decision record capturing what was decided, why, what alternatives were rejected, and what the ongoing maintenance model is | `docs/ADR-2025-002-IaC-Module-Refactoring.md` |
| 5 | Announce to engineering: Slack post + short demo of the CLI flow. Mark old GitLab template as deprecated for the three migrated resource types. | Communication sent |

### Sprint 6 Deliverables

```
# Final state of both repos:

infra-cli/
├── src/infra_cli/
│   ├── cli.py
│   ├── schema.py
│   ├── config.py
│   ├── renderer.py
│   └── commands/
│       ├── create.py              # infra create <resource>
│       ├── list.py                # infra list
│       ├── docs.py                # infra docs <resource>
│       └── init.py                # infra init
├── config/
│   └── environments.yaml
├── templates/
│   ├── lambda-processor/
│   ├── s3-bucket/
│   └── {third-module}/
├── tests/
├── pyproject.toml
└── README.md

infra-modules-v2/
├── modules/
│   ├── lambda-processor/
│   │   ├── main.tf, iam.tf, alarms.tf
│   │   ├── variables.tf, outputs.tf, versions.tf
│   │   ├── README.md (auto-generated)
│   │   └── tests/
│   │       ├── lambda.tftest.hcl
│   │       ├── security.tftest.hcl
│   │       └── setup/
│   ├── s3-bucket/
│   │   ├── main.tf, variables.tf, outputs.tf, versions.tf
│   │   ├── README.md (auto-generated)
│   │   └── tests/
│   └── {third-module}/
├── schemas/
│   ├── lambda-processor.yaml
│   ├── s3-bucket.yaml
│   └── {third-module}.yaml
├── examples/
│   ├── lambda-processor/dev/
│   ├── s3-bucket/dev/
│   └── {third-module}/dev/
├── tests/integration/
│   ├── lambda_test.go
│   ├── s3_test.go
│   └── {third-module}_test.go
├── scripts/
│   └── generate_docs.py
├── ci-templates/
│   └── terraform-pipeline.yml
├── docs/
│   ├── getting-started.md
│   ├── contributing.md
│   ├── environment-promotion.md
│   ├── security-compliance-matrix.md
│   ├── module-migration-playbook.md
│   ├── remaining-modules-estimate.md
│   └── ADR-2025-002-IaC-Module-Refactoring.md
├── audit/
│   ├── lambda-current-state.md
│   ├── s3-current-state.md
│   └── {third-module}-current-state.md
└── .gitlab-ci.yml
```

---

## Sprint Summary

| Sprint | Weeks | Focus | Key Output |
|--------|-------|-------|------------|
| 0 | Pre-work | Repo setup, access, tooling | Developer unblocked on day 1 |
| 1 | 1–2 | Lambda module + schema + docs | Working composition module, YAML schema, auto-generated README |
| 2 | 3–4 | CLI factory core | `infra create`, `infra list`, `infra docs` — generates deployable native TF |
| 3 | 5–6 | Module testing + CLI polish | `terraform test` + Terratest + GitLab CI integration |
| 4 | 7–8 | Second module (S3) | Pattern proven at scale; two modules fully operational |
| 5 | 9–10 | Third module + onboarding | Developer getting-started guide; 2–3 app teams onboarded |
| 6 | 11–12 | Production readiness + handoff | Packaged CLI, CI templates, migration playbook, ADR, announcement |

## Risk Mitigations

| Risk | Mitigation |
|------|-----------|
| Lambda pilot turns out to be too complex for Sprint 1 | Schema + module design in Sprint 1 is intentionally separated from CLI work in Sprint 2. If the module takes longer, CLI work shifts by a few days, not a full sprint. |
| Terratest sandbox permissions issues | Sprint 0 pre-work includes verifying IAM access. If blocked, defer Terratest to Sprint 4 and use plan-based tests only in Sprint 3. |
| App team feedback in Sprint 2 requires major CLI redesign | Sprint 3 has explicit buffer for "CLI polish based on feedback." If feedback is severe, cut the Terratest integration test from Sprint 3 and use the time for CLI rework. |
| Third module is a composition module as complex as Lambda | Sprint 5 includes 3 days for module implementation. If it's a simple standalone (like S3 was), this leaves buffer. If it's complex, cut the `infra init` command from Sprint 5 — it's nice-to-have. |
| Developer gets pulled to other work mid-quarter | Sprints 1–3 are the critical path. If the developer is pulled after Sprint 3, you still have a working CLI + one tested module — enough to prove the pattern and justify continued investment. |

## Success Criteria (End of 3 Months)

| Metric | Target |
|--------|--------|
| Modules fully migrated to new pattern | 3 |
| Developer lead time for those 3 resources | < 1 day (down from 2–3 weeks) |
| Test coverage (plan-based) | 100% for all 3 modules |
| Test coverage (integration) | 100% for all 3 modules |
| Documentation coverage | 100% auto-generated for all 3 modules |
| Terragrunt dependency | Eliminated for all new modules |
| App developers onboarded | 2–3 teams |
| Remaining module migration estimate | Documented with timeline |
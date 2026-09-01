# HCS DBaaS Platform: Layered Terraform Infrastructure as Code
 
## Table of Contents
* [Overview](#overview)
* [Real World Business Value](#real-world-business-value)
* [Skills Demonstrated](#skills-demonstrated)
* [Project Folder Structure](#project-folder-structure)
* [Tasks and Implementation Steps](#tasks-and-implementation-steps)
* [Core Implementation Breakdown](#core-implementation-breakdown)
* [IAM Role and Permissions](#iam-role-and-permissions)
* [Project Features (Detailed Breakdown)](#project-features-detailed-breakdown)
* [Design Decisions and Highlights](#design-decisions-and-highlights)
* [Local Testing and Validation](#local-testing-and-validation)
* [Errors Encountered and Resolved](#errors-encountered-and-resolved)
* [Conclusion](#conclusion)
## Overview
 
This repository contains the Infrastructure as Code for a private cloud Database as a Service (DBaaS) platform running on Huawei Cloud Stack (HCS), built for a regulated banking environment. It provisions and manages the full lifecycle of database infrastructure (network foundations, compute, data stores, identity and access, and observability) through a layered Terraform architecture with a reusable module library and Azure DevOps CI/CD pipelines.
 
The platform is decomposed into five active, numbered layers, each a logical concern deployed in dependency order:
 
| Layer | Name | Contains |
|---|---|---|
| 0 | Guardrails | Security groups, pipeline identity (RBAC), authorisation (policy as code) |
| 1 | Landing Zone | Virtual Data Centre project, VPC, subnets, route tables |
| 2 | Infrastructure | Elastic Cloud Server instances, object storage buckets |
| 3 | Data | RDS PostgreSQL instances, data replication |
| 5 | Shared | Prometheus, Grafana, Alertmanager, a custom metrics exporter |
 
Two repositories work together to deliver this: `dbaas-infra-live` (referred to as `infra-live/` throughout this document) holds environment specific configuration for every layer, and `dbaas-infra-modules` (referred to as `infra-modules/`) holds the single purpose, reusable Terraform modules that every layer's live configuration calls into. Both are represented as top level folders in this repository, see [Project Folder Structure](#project-folder-structure).
 
**Scope boundaries.** This repository owns Terraform manageable infrastructure only. It explicitly does not create, modify or delete objects in the organisation's external Active Directory; human identity is federated in, never managed by this codebase. It does not include Layer 4 (cloud native monitoring via the platform's built in trace and log services), which was scoped in the original design but deprioritised. Full technical depth on access governance, observability, and the CI/CD pipeline template lives in the three linked companion repositories rather than being duplicated here.
 
* `prometheus-grafana-managed-rds-observability`: the full Layer 5 observability stack, including the custom metrics exporter, alerting rules, and Grafana dashboards.
* `azure-devops-terraform-stages-pipeline`: the reusable, three mode Terraform stage template every layer in this platform deploys through.
* `hcs-iam-rbac-policy-as-code-design`: the full Layer 0 access governance architecture, covering the threat model, pipeline identity design, and the no clickops migration strategy.
## Real World Business Value
 
Before this platform existed, provisioning database infrastructure meant manual, inconsistent work across environments with no enforced quality gate and no single source of truth for what infrastructure actually existed. This codebase replaces that with:
 
* **Self service, repeatable provisioning.** A new RDS instance is a map entry in a `.tfvars` file, not a bespoke manual build. Every instance is provisioned through the same module with the same guardrails, every time.
* **Environment parity by construction.** Every environment (dev, test, UAT, non prod, prod) shares the same modules and the same folder shape, so onboarding a new environment means filling in tfvars rather than restructuring code.
* **Enforced quality and security gates on every change.** `terraform fmt`, `tflint`, and a HIGH or CRITICAL severity `tfsec` scan run ahead of every plan, for every layer, with no path to bypass them.
* **Auditability.** Every infrastructure change is a reviewed pull request and a Terraform plan artifact, not an undocumented console action, directly relevant in a regulated banking context where change evidence is an audit requirement, not a nice to have.
## Skills Demonstrated
 
* **Layered infrastructure architecture.** Decomposed a full platform into five numbered, dependency ordered layers, each independently deployable and independently versioned, with a sixth layer explicitly scoped out and documented as deliberately deprioritised rather than silently dropped.
* **Terraform module design.** Built a reusable module library consumed by every environment through parameterisation alone, using `for_each` over map variables so new instances are additive configuration changes rather than new resource blocks.
* **Multi backend state management.** Designed per stack Terraform state isolation on an S3 compatible object storage backend, including diagnosing and correctly configuring the compatibility flags a private cloud storage backend needs that public AWS S3 does not.
* **Cross stack data flow without tight coupling.** Used `terraform_remote_state` so downstream stacks consume upstream outputs (VPC IDs, security group IDs) without hardcoding values or relying on pipeline level variable passing.
* **CI/CD pipeline engineering.** Designed and consumed a reusable, three mode Azure DevOps pipeline template, and a second, independently triggered pipeline for non Terraform observability deployment with its own validation tooling.
* **Access governance and least privilege design.** Replaced a placeholder Active Directory coupled identity design with a two stack model (pipeline identity, authorisation) built around explicit least privilege scoping. Full depth in the linked companion repository.
* **Production observability engineering.** Built and operated a full metrics, alerting and dashboarding stack for a live PostgreSQL fleet. Full depth in the linked companion repository.
* **Incident and defect resolution on running infrastructure.** Diagnosed and resolved real operational issues including state backend compatibility failures, memory exhaustion from exporter fleet drift, and credential lockout risk on a shared service account. See [Errors Encountered and Resolved](#errors-encountered-and-resolved).
## Project Folder Structure
 
```
infra-live/                                   # Live infrastructure, environment specific configuration
├── 0-vdc-guardrails/                         # Layer 0: identity, RBAC, policy as code, network security
│   ├── NO-CLICKOPS-STRATEGY.md
│   ├── RBAC-PAC-IMPLEMENTATION-PLAN.md
│   ├── RBAC-PIPELINE-DESIGN.md
│   ├── 0-dev/
│   │   ├── policy-as-code/                   # Authorisation: custom roles, groups, users, assignments
│   │   ├── rbac/                             # Pipeline identity bootstrap (plan/apply/state accounts)
│   │   └── security-groups/                  # Network security group definitions
│   └── 1-test/ ... 4-prod/                   # Same sub stack shape, scaffolded, not yet populated
├── 1-vdc-landing-zone/                       # Layer 1: network foundations
│   └── 0-dev/
│       ├── route-tables/
│       ├── subnet-cidr/
│       ├── vdc/                              # Virtual Data Centre project
│       └── vpc/                              # VPC and subnet provisioning
├── 2-vdc-infra/                              # Layer 2: compute infrastructure
│   └── 0-dev/
│       ├── ecs/                              # Elastic Cloud Server instances (bastion, etc.)
│       └── obs-buckets/                      # Object storage (Terraform state and artefacts)
├── 3-vdc-data/                               # Layer 3: database layer
│   └── 0-dev/
│       ├── drs/                              # Data replication tooling
│       └── rds/                              # RDS PostgreSQL instances (for_each over a map)
├── 5-vdc-shared/                             # Layer 5: shared observability (full depth in companion repo)
│   ├── DECISIONS-dev.md
│   └── 0-dev/
│       ├── alertmanager/
│       ├── grafana/
│       ├── oc-exporter/
│       └── prometheus/
├── METRICS-AND-DASHBOARDS-DOCUMENTATION.md
├── MONITORING-DOCUMENTATION.md
├── PIPELINE-DOCUMENTATION.md
└── pipelines/                                 # Azure DevOps pipeline definitions (full depth in companion repo)
    ├── 0-dev.yml
    ├── 0-dev-monitoring.yml
    ├── 1-test.yml ... 4-prod.yml
    └── templates/
        ├── terraform-stages.yml
        └── env-pipeline.yml
 
infra-modules/                                 # Reusable Terraform modules, consumed by every layer above
├── ecs/
├── golden-image/
├── obs-bucket/
├── rbac/
├── rds/
├── security-groups/
├── vdc/
└── vpc/
```
 
## Tasks and Implementation Steps
 
1. **Decompose the platform into numbered, dependency ordered layers.** Rather than one flat Terraform configuration, infrastructure was split into five logical concerns (guardrails, landing zone, infrastructure, data, shared observability), each deployed in strict order, so a change to one concern has a contained blast radius.
2. **Extract reusable modules from live configuration.** Every resource definition (VPC, ECS, RDS, security groups, VDC project, golden image, RBAC bootstrap) was pulled out into a single purpose module in a separate repository, so live configuration only supplies environment specific parameters.
3. **Design per stack Terraform state isolation.** Each stack keeps its own state file on an S3 compatible object storage backend, keyed by environment and layer, rather than one shared state file for the whole platform. Artefact: the `5-backend.tf` file present in every stack folder.
4. **Adopt `for_each` over map variables for repeatable resources.** ECS and RDS instances are declared as map entries in `.auto.tfvars`, so adding an instance is a configuration change rather than a new resource block.
5. **Wire cross stack data flow through remote state, not pipeline variables.** Downstream stacks (RDS, security groups) read upstream outputs (VPC ID, subnet ID) via `terraform_remote_state`, so values are never hardcoded or passed as loosely typed pipeline variables. Artefact: the `data "terraform_remote_state"` blocks in the RDS stack.
6. **Replace the original single stack identity design with a two stack access governance model.** The original design coupled pipeline identity to Active Directory integration in one placeholder stack. This was replaced because pipeline identity and authorisation content are different concerns with different blast radii. Full detail in the companion access governance repository.
7. **Build a self hosted observability stack in place of the originally scoped cloud native monitoring layer.** Layer 4 (cloud native monitoring) was deprioritised in favour of Layer 5, a self hosted Prometheus, Grafana and custom exporter stack running on the Layer 2 bastion host. Full detail in the companion observability repository.
8. **Build one reusable CI/CD stage template rather than duplicating pipeline logic per stack.** Every stack in every environment deploys through the same three mode template. Full detail in the companion pipeline repository.
9. **Extract a shared environment pipeline template so the four gated environments cannot drift.** `1-test.yml` through `4-prod.yml` each call one shared template with five parameters, rather than maintaining four separate stage graphs.
10. **Separate the observability deployment pipeline from the Terraform pipeline.** The Layer 5 stack deploys over SSH with its own validation tooling (`promtool`, Python JSON checks), triggered independently from Terraform changes.
11. **Pre scaffold every non dev environment with the same folder shape as dev.** `1-test/` through `4-prod/` exist under every layer today, populated with placeholder or empty configuration, so onboarding a new environment is additive rather than a restructuring exercise.
12. **Track known operational gaps explicitly rather than silently.** A living issues log records what remains open (email alerting pending an SMTP relay, the ECS stack not yet on the remote state pattern RDS uses) rather than letting these gaps go undocumented. See [Errors Encountered and Resolved](#errors-encountered-and-resolved).
## Core Implementation Breakdown
 
### Layered dependency graph
 
Layers deploy in strict order, either through `terraform_remote_state` reads or pipeline level `dependsOn` ordering:
 
```
Layer 0: Guardrails       →  Security Groups, RBAC (pipeline identity), Policy as Code (roles/groups/users)
Layer 1: Landing Zone     →  VDC Project, VPC, Subnets (depends on Layer 0)
Layer 2: Infrastructure   →  ECS, OBS Buckets (depends on Layer 1)
Layer 3: Data             →  RDS Instances (depends on Layer 1 and Layer 0 security groups)
Layer 5: Shared           →  Prometheus, Grafana, Alertmanager (runs on the Layer 2 ECS bastion)
```
 
### Terraform module pattern: for_each plus parameter merge
 
The live RDS stack defines a baseline parameter set in `locals` and merges per instance overrides on top, so every instance gets consistent tuning without duplicating a full parameter list per instance:
 
```hcl
locals {
  pg_defaults = {
    max_slot_wal_keep_size = "10240"
    wal_keep_size          = "10240"
    max_connections        = "5001"
  }
}
 
module "rds" {
  source   = "../../../../infra-modules/rds"
  for_each = var.rds_instances
 
  db_parameters = merge(local.pg_defaults, each.value.db_parameters)
  # ...remaining arguments per instance
}
```
 
The same `for_each` over a map pattern is used for compute, so adding a new bastion or worker instance is a `.auto.tfvars` change, not a new resource block:
 
```hcl
module "ecs" {
  source   = "../../../../infra-modules/ecs"
  for_each = var.ecs_instances
 
  name                 = each.key
  vpc_az               = each.value.vpc_az
  ecs_image_id         = each.value.ecs_image_id
  ecs_flavor           = each.value.ecs_flavor
  ecs_keypair_name     = each.value.ecs_keypair_name
  subnet_id            = data.terraform_remote_state.vpc.outputs.subnet_id
  security_group_id    = var.security_group_id
  tags                 = each.value.tags
}
```
 
### State isolation and the OBS backend compatibility layer
 
Every stack keeps independent state on an S3 compatible object storage backend, which requires several compatibility flags that a true AWS S3 backend does not:
 
```hcl
terraform {
  backend "s3" {
    bucket = "<tfstate-bucket>"
    key    = "dev/vdcs/vdc-data/rds/terraform.tfstate"
    region = "<region>"
 
    endpoints = {
      s3 = "http://<obs-endpoint>"
    }
 
    use_path_style              = true
    skip_region_validation      = true
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_requesting_account_id  = true
  }
}
```
 
Downstream stacks read upstream outputs through `terraform_remote_state` rather than the pipeline passing variables between stages:
 
```hcl
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket     = "<tfstate-bucket>"
    key        = "dev/vdcs/vdc-landing-zone/vpc/terraform.tfstate"
    region     = "<region>"
    access_key = var.access_key
    secret_key = var.secret_key
    endpoints  = { s3 = "http://<obs-endpoint>" }
    use_path_style              = true
    skip_region_validation      = true
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_requesting_account_id  = true
  }
}
```
 
State isolation per stack means a failed RDS apply cannot block a VPC change, unrelated stacks can be planned and applied in parallel, and state file access can be scoped per stack. This is precisely what the pipeline's state management identity, described in the companion access governance repository, is scoped to.
 
### CI/CD: one reusable template, three modes
 
Every stack in every environment deploys through one template supporting three modes, selected by parameters:
 
| Mode | Behaviour |
|---|---|
| `placeholder: true` | Prints a skip message, used for stacks not yet implemented |
| `requiresApproval: false` | Single stage: lint, scan, init, plan, apply (dev fast path) |
| `requiresApproval: true` | Two stages: plan publishes an artifact, then an environment approval gate, then apply consumes the exact reviewed plan |
 
Quality gates run in this order on every stack: `terraform fmt -check -recursive`, `tflint --force`, `tfsec . --minimum-severity HIGH` (hard fail on HIGH or CRITICAL), `terraform validate`, `terraform plan`. Full design rationale for this template lives in the companion pipeline repository.
 
The four gated promotion environments each call one shared template rather than duplicating a stage graph:
 
```yaml
extends:
  template: templates/env-pipeline.yml
  parameters:
    envDisplayName: "test"
    envStageLabel: "test"
    envApprovalName: "test"
    envFolder: "1-test"
```
 
Command line equivalent of what the pipeline runs for a single stack:
 
```bash
cd infra-live/1-vdc-landing-zone/0-dev/vpc
 
export AWS_ACCESS_KEY_ID="<hcs-access-key>"
export AWS_SECRET_ACCESS_KEY="<hcs-secret-key>"
export AWS_REGION="<region>"
export AWS_EC2_METADATA_DISABLED="true"
 
terraform init -input=false
terraform validate
terraform plan -input=false -out=tfplan
terraform apply -input=false tfplan
```
 
## IAM Role and Permissions
 
Full architectural depth on the platform's identity and access model, including the threat model, the three service account pipeline identity design, the least privilege custom role catalogue, and the phased no clickops migration strategy, lives in the companion access governance repository. Summarised here at the level relevant to this repository's own scope:
 
* Pipeline identity is separated from authorisation content. The Layer 0 guardrails split into `rbac/` (which service accounts the pipeline itself runs as) and `policy-as-code/` (what human personas are permitted to do), because these are different concerns with different blast radii, not because of a naming preference.
* Three distinct service accounts, a read only plan identity, a write scoped apply identity, and a state management identity, replace a single standing pipeline credential. Access key and secret key pairs are rotated on a fixed cadence of at most every 90 days, since this platform does not support workload identity federation with Azure DevOps.
* No secrets are committed to this repository. All credentials (`HCS_ACCESS_KEY`, `HCS_SECRET_KEY`, `DB_PASSWORD`, `OC_PASSWORD`, `GRAFANA_ADMIN_PASSWORD`) are stored in a restricted Azure DevOps variable group and substituted into configuration files from placeholder tokens only at deploy time.
* The database engine layer has its own least privilege account, separate from cloud IAM. The metrics exporter connects through a dedicated read only PostgreSQL role rather than an administrative database account, and its connection string is supplied through a mode 0600, root owned credential file rather than a command line argument, which would otherwise leak the password through the process list.
* SSH access to the bastion host uses strict host key verification. `StrictHostKeyChecking=yes` against a `known_hosts` file populated dynamically at job start, and the SSH private key is provided as an Azure DevOps Secure File, downloaded per job and never persisted to the agent disk or the repository.
## Project Features (Detailed Breakdown)
 
**Layered, dependency ordered infrastructure**
Purpose: contain the blast radius of any single infrastructure change and allow independent stacks to be reasoned about, planned and applied in isolation. Implementation summary: five numbered top level layers, each with its own environment subfolders, deployed through pipeline level `dependsOn` ordering that mirrors the real infrastructure dependency graph. Operational considerations: a failure in one layer's apply does not corrupt or block an unrelated layer. `prevent_destroy` lifecycle rules on foundational resources (VPC, subnets, security groups) require a deliberate manual override rather than allowing an accidental destroy through the normal pipeline path.
 
**Reusable module library**
Purpose: eliminate duplicated resource definitions across environments and stacks. Implementation summary: eight single purpose modules (VPC, ECS, RDS, security groups, VDC, golden image, OBS bucket, RBAC bootstrap) consumed by live configuration through parameters only. Operational considerations: a module change is reviewed once and takes effect everywhere it is consumed, which raises the stakes of a module level regression, mitigated by the same lint and security scan gates applying to module source as to live configuration.
 
**Access governance (RBAC and policy as code)**
Purpose: enforce least privilege, auditable identity and access management, replacing informal standing administrative access. Implementation summary: a two stack model separating pipeline identity from authorisation content, with a documented threat model and a phased plan to eliminate console write access entirely. Operational considerations: full logging, monitoring and failure mode detail, including drift detection design and break glass procedure, is documented in the companion repository rather than duplicated here. Artefacts: `hcs-iam-rbac-policy-as-code-design`.
 
**Shared observability stack**
Purpose: give the platform team proactive visibility into a fully managed RDS fleet that exposes no native host level metrics. Implementation summary: a custom metrics exporter, a reconciler managed `postgres_exporter` fleet, Prometheus alerting rules, and Grafana dashboards, all running on the Layer 2 bastion host and deployed through an independent pipeline. Operational considerations: dashboard deploys require temporarily disabling Grafana's brute force login protection for the duration of the API push, a deliberate and time boxed trade off rather than an oversight. See [Errors Encountered and Resolved](#errors-encountered-and-resolved). Artefacts: `prometheus-grafana-managed-rds-observability`.
 
**Reusable CI/CD pipeline template**
Purpose: guarantee every stack, in every environment, receives identical quality gates without duplicating pipeline logic. Implementation summary: a single template supporting placeholder, unapproved dev, and approval gated controlled modes, plus a shared environment template so the four gated environments cannot drift from one another. Operational considerations: the apply stage always executes the exact plan artifact published by the plan stage rather than regenerating it, closing the gap between what a reviewer approved and what actually gets applied. Artefacts: `azure-devops-terraform-stages-pipeline`.
 
**Cross stack state isolation**
Purpose: give every stack an independent blast radius and allow fine grained, per stack access control on Terraform state itself. Implementation summary: one state file per stack per environment on an S3 compatible object storage backend, with downstream stacks reading upstream outputs through `terraform_remote_state` rather than hardcoded values. Operational considerations: the OBS backend's S3 compatibility flags (`use_path_style`, `skip_region_validation`, and a custom `endpoints` block) are required in every backend configuration. Omitting any of them causes `terraform init` to fail against this private cloud storage backend. Artefacts: the `5-backend.tf` file present in every stack folder.
 
## Design Decisions and Highlights
 
| Decision | Alternatives Considered | Rationale |
|---|---|---|
| Five numbered, dependency ordered layers instead of one flat configuration | A single monolithic Terraform root module | Contains the blast radius of any one change and allows independent stacks to be planned, applied and reasoned about in isolation |
| Reusable module library in a separate repository from live configuration | Defining resources inline in every live stack | A module change is reviewed once and takes effect everywhere it is consumed, rather than needing to be copy pasted and kept in sync across every environment |
| `for_each` over map variables for ECS and RDS instances | Individually named resource blocks per instance | Adding an instance becomes a configuration change (`.auto.tfvars`) rather than a Terraform code change, at the accepted cost that renaming a map key forces a destroy and recreate |
| Per stack Terraform state with `terraform_remote_state` for cross stack reads | One shared state file for the whole platform | Isolates blast radius per stack and allows state file access itself to be scoped per pipeline identity |
| Layer 4 (cloud native monitoring) explicitly deprioritised in favour of Layer 5 (self hosted observability) | Building both, or delaying Layer 5 until Layer 4 was complete | The fully managed RDS service exposed no usable host level metrics through any native path, making a custom exporter the higher priority, higher value investment. Layer 4 is documented as descoped rather than silently dropped |
| Non dev environments pre scaffolded with the same folder shape as dev, populated on demand | Creating each environment's folder structure only when it is actually onboarded | Onboarding a new environment becomes filling in `.tfvars` and pipeline parameters rather than a structural repository change |
| Monitoring deployment pipeline kept fully independent from the Terraform pipeline | Bolting SSH/SCP deployment steps onto the Terraform stage template | Monitoring configuration iterates on a faster cadence and needs different validation tooling (`promtool`, Python JSON checks) than Terraform does |
| Original single IAM style stack split into separate `rbac/` and `policy-as-code/` stacks | Keeping pipeline identity and authorisation content in one stack | Pipeline identity and human/service authorisation are different concerns with different blast radii and different reviewers. Full rationale in the companion access governance repository |
 
## Local Testing and Validation
 
Local prerequisites, matching what the pipeline enforces:
 
* Terraform 1.5 or later
* The HCS Terraform provider, pinned to a tested `~> 2.4.x` release
* Access to the platform tenant with VDC scoped credentials
* An object storage bucket for Terraform state, with versioning enabled
* `tflint` and `tfsec` installed and on `PATH`
Validating a single stack locally, in the same order the pipeline runs:
 
```bash
cd infra-live/1-vdc-landing-zone/0-dev/vpc
 
terraform fmt -check -recursive
tflint --init && tflint --force
tfsec . --minimum-severity HIGH
 
export AWS_ACCESS_KEY_ID="<hcs-access-key>"
export AWS_SECRET_ACCESS_KEY="<hcs-secret-key>"
export AWS_REGION="<region>"
export AWS_EC2_METADATA_DISABLED="true"
 
terraform init -input=false
terraform validate
terraform plan -input=false -out=tfplan
```
 
Expected outcome: `terraform validate` returns `Success! The configuration is valid.`, and the plan output shows only the resources genuinely expected to change. Any unexpected resource in the plan is treated as a signal to stop before applying, not a signal to proceed and check later.
 
Dashboard and alerting rule changes for the shared observability layer are validated with `promtool check config`, `promtool check rules`, and a Python JSON load check before deployment. Full detail in the companion observability repository.
 
A small HTTP report service (`ci-rds-report`) surfaces a live summary of RDS stack state during CI runs, giving a fast visual check of what a given pipeline run actually did without needing to read the full Terraform plan output.
 
## Errors Encountered and Resolved
 
| Symptom | Root Cause | Fix | Preventative Measure |
|---|---|---|---|
| `terraform init` failed against the object storage backend with default S3 backend settings | The private cloud object storage backend is S3 compatible but not true AWS S3 | Set `use_path_style`, `skip_region_validation`, and a custom `endpoints` block in every backend configuration | Baked into the standard backend configuration pattern used by every stack, so new stacks inherit the fix automatically |
| Terraform state operations failed without `AWS_*` environment variables set, despite no AWS usage in the platform | The provider delegates state operations to the S3 compatible backend, which reads AWS named environment variables regardless of the underlying cloud | Pipeline sets `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, and `AWS_EC2_METADATA_DISABLED=true` for every stack | Standardised as a required environment block in the reusable pipeline stage template |
| A pipeline run attempted to destroy a foundational VPC or security group resource | `prevent_destroy` lifecycle rules exist specifically to block this | Blocked as designed, a manual override is required for a genuinely intended destroy | Documented as an intentional guardrail rather than a defect, so operators expect the manual step rather than treating it as a pipeline failure |
| The ECS stack cannot resolve `subnet_id` and `security_group_id` automatically | Unlike the RDS stack, ECS still takes these as plain variables rather than reading them from `terraform_remote_state` | Values are currently supplied manually or via `TF_VAR_*` injection | Open, tracked as an alignment task to bring ECS onto the same remote state pattern RDS already uses |
| The planned cloud native monitoring layer (Layer 4) was never built | Deprioritised in favour of the self hosted observability stack once it became clear the fully managed RDS service exposed no usable native metrics | Documented explicitly as descoped rather than left as a silent gap | Audit log export beyond the platform's default retention window remains a tracked follow up |
| The `postgres_exporter` fleet exceeded its designed process count and per process memory, causing memory exhaustion on the bastion host | The reconciler's target instance count and the 80 MiB per process memory ceiling were both exceeded in practice as the fleet grew | Remediation path is to fully activate and verify the reconciler, prune stale exporter units, and evaluate a pooled connection exporter model for the longer term | A hard `MemoryMax` ceiling was already set on every exporter unit before this incident, which contained the blast radius to the bastion rather than an individual database |
| The metrics exporter's shared service account risked extending its own lockout during retries | The account is subject to both a CAPTCHA based brute force lockout and a separate account level failed login lockout, and naive scripted retries could trigger either | Exponential backoff is implemented in the exporter's authentication retry logic. See the companion observability repository for the full pattern | A dedicated, non shared service account for the exporter is planned to remove the shared lockout risk entirely |
| Grafana dashboard deploys required disabling brute force login protection mid deploy | The dashboard push API pattern authenticates repeatedly in a way that trips the same brute force protection meant to protect the admin account | Mitigated with a fixed, fully scripted disable, push, re enable sequence rather than manual steps | Scripting the full sequence end to end minimises the window during which brute force protection is disabled, rather than leaving it open for an arbitrary manual process |
| Email alerting was configured but never went live | The Alertmanager email receiver is fully configured but depends on a shared mailbox and SMTP relay that has not yet been provisioned | Microsoft Teams webhook delivery serves as the working interim alert channel | Alerting coverage was never lost while the email path remained pending, because the interim channel was built first |
| Non dev environments exist only as scaffolding | `1-test` through `4-prod` folders exist under every layer, but most `.tf` files remain empty placeholders | Environments are populated on demand as they are formally onboarded | The folder shape was pre created for every environment from day one, so onboarding is additive rather than a restructuring exercise |
| Renaming an RDS or ECS instance's map key forces a destroy and recreate rather than an in place rename | Terraform's `for_each` treats the map key as part of each resource's identity | Accepted, instance keys are treated as permanent once created | The naming convention for instance keys is documented and treated as a hard constraint from the start, rather than discovered after an accidental destroy |
 
## Conclusion
 
This platform demonstrates that a database infrastructure estate can be made self service, auditable and environment consistent without sacrificing the security posture a regulated environment requires. The layered architecture contains blast radius, the reusable module library keeps every environment honest against drift, and the CI/CD template guarantees no stack skips a quality gate regardless of how it was added. Just as importantly, the platform is documented as it actually is: a deliberately descoped monitoring layer, environments that are scaffolded but not yet active, and a living issues log of what remains open, rather than a system presented as more finished than it is. The three companion repositories linked throughout, covering access governance, observability, and the CI/CD template in full depth, are the evidence that each of those subsystems was built with the same rigour as this integration layer.

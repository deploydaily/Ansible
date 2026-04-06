# Enterprise AWX Platform — Architecture Reference

> Sandbox build on AWS · Mirrors enterprise Red Hat AAP deployment patterns  
> Repo: [deploydaily/Ansible](https://github.com/deploydaily/Ansible)  
> Infrastructure: [deploydaily/terraform-aws](https://github.com/deploydaily/terraform-aws)

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture Diagram](#architecture-diagram)
3. [Repository Structure](#repository-structure)
4. [Infrastructure Layer](#infrastructure-layer)
5. [AWX Installation Pattern](#awx-installation-pattern)
6. [Collection Structure](#collection-structure)
7. [Inventory Model](#inventory-model)
8. [RBAC Model](#rbac-model)
9. [SDLC Pipeline](#sdlc-pipeline)
10. [Execution Environments](#execution-environments)
11. [Sandbox vs Enterprise Comparison](#sandbox-vs-enterprise-comparison)
12. [Architecture Decision Records](#architecture-decision-records)

---

## Overview

This platform demonstrates an enterprise-grade Ansible Automation Platform (AAP)
deployment on AWS, built to mirror the patterns used in production AAP environments
running on-premises with Red Hat Enterprise Linux.

The infrastructure is provisioned by Terraform and the AWX controller is configured
entirely via Ansible — no manual UI clicks. This reflects the real-world AAP engineering
practice of treating automation infrastructure as code.

**Key design goals:**

- AWX runs as systemd services on RHEL 9 — identical to the AAP installer pattern
- All AWX configuration managed as code via the `awx.awx` collection
- Shared content catalog via a custom `platform.automation` Ansible collection
- Full SDLC chain: lint → molecule test → publish → AWX sync on every merge
- Dynamic inventory via EC2 tags — no static host files
- Secrets managed via AWS Secrets Manager — no hardcoded credentials anywhere

---

## Architecture Diagram

```
Developer Workstation / GitHub Actions
              |
              | HTTPS :443
              v
+-------------------------------------------------------------+
|  AWS VPC  10.0.0.0/16  .  us-east-1                        |
|                                                             |
|  +--- Public Subnet 10.0.1.0/24 -------------------------+ |
|  |                                                        | |
|  |  AWX Controller (t3.medium . RHEL 9)                  | |
|  |  +-- awx-web.service      (Daphne ASGI . :8052)       | |
|  |  +-- awx-task.service     (Celery worker)              | |
|  |  +-- awx-receptor.service (Execution mesh)             | |
|  |  +-- redis.service        (Queue + cache . :6379)      | |
|  |  +-- nginx                (TLS termination . :443)     | |
|  |                                                        | |
|  |  Internet Gateway (outbound: GitHub, ECR, dnf)        | |
|  +--------------------------------------------------------+ |
|           |                                                  |
|           | SSH :22 / WinRM :5985-5986                       |
|           | (private IP, SG-scoped)                          |
|           v                                                  |
|  +--- Public Subnet 10.0.1.0/24 (no public IP) ----------+ |
|  |  linux-node-01    Amazon Linux 2023  t3.micro  dev     | |
|  |  linux-node-02    Amazon Linux 2023  t3.micro  prod    | |
|  |  windows-node-01  Windows Server 2022  t3.micro  dev   | |
|  +--------------------------------------------------------+ |
|                                                              |
|  +--- Private Subnets 10.0.11.0/24 . 10.0.12.0/24 ------+ |
|  |  RDS PostgreSQL 15  (db.t3.micro . 20GB gp2)          | |
|  |  Secrets Manager    (awx/db-password, awx/admin)      | |
|  +--------------------------------------------------------+ |
+-------------------------------------------------------------+

GitHub (external)
  +-- terraform-aws     Terraform infra repo
  +-- Ansible           AWX content + config repo
```

---

## Repository Structure

```
Ansible/
|
+-- ansible.cfg                         # Global Ansible configuration
+-- requirements.yml                    # Collection dependencies
+-- .gitignore
+-- README.md
|
+-- collections/
|   +-- platform/
|       +-- automation/                 # Custom enterprise collection
|           +-- galaxy.yml              # Collection metadata
|           +-- plugins/
|           |   +-- inventory/          # Custom inventory plugins (future)
|           +-- roles/
|               +-- os_baseline/        # OS prep -- RHEL + Windows
|               |   +-- defaults/
|               |   |   +-- main.yml
|               |   +-- handlers/
|               |   |   +-- main.yml
|               |   +-- meta/
|               |   |   +-- main.yml
|               |   +-- tasks/
|               |   |   +-- main.yml
|               |   |   +-- redhat.yml
|               |   |   +-- windows.yml
|               |   +-- templates/
|               |       +-- chrony.conf.j2
|               |       +-- sshd_config.j2
|               |
|               +-- awx_install/        # AWX systemd install -- mirrors AAP pattern
|               |   +-- defaults/
|               |   |   +-- main.yml
|               |   +-- handlers/
|               |   |   +-- main.yml
|               |   +-- meta/
|               |   |   +-- main.yml
|               |   +-- tasks/
|               |   |   +-- main.yml
|               |   |   +-- preflight.yml
|               |   |   +-- dependencies.yml
|               |   |   +-- setup.yml
|               |   |   +-- secrets.yml
|               |   |   +-- install.yml
|               |   |   +-- configure.yml
|               |   |   +-- database.yml
|               |   |   +-- services.yml
|               |   |   +-- nginx.yml
|               |   |   +-- healthcheck.yml
|               |   +-- templates/
|               |       +-- settings.py.j2
|               |       +-- awx-web.service.j2
|               |       +-- awx-task.service.j2
|               |       +-- awx-receptor.service.j2
|               |       +-- receptor.conf.j2
|               |       +-- nginx-awx.conf.j2
|               |
|               +-- awx_config/         # AWX config as code (awx.awx collection) -- roadmap
|               |   +-- defaults/
|               |   |   +-- main.yml
|               |   +-- meta/
|               |   |   +-- main.yml
|               |   +-- tasks/
|               |       +-- main.yml
|               |       +-- organizations.yml
|               |       +-- teams.yml
|               |       +-- credentials.yml
|               |       +-- inventories.yml
|               |       +-- projects.yml
|               |       +-- job_templates.yml
|               |
|               +-- os_hardening/       # CIS-aligned OS hardening (roadmap)
|               +-- patching/           # Linux + Windows patching (roadmap)
|
+-- playbooks/
|   +-- site.yml                        # Master playbook -- full stack
|   +-- install_awx.yml                 # os_baseline -> awx_install
|   +-- configure_awx.yml               # awx_config as code
|   +-- os_baseline.yml                 # Managed node baseline
|   +-- os_hardening.yml                # Managed node hardening
|   +-- patching.yml                    # Linux + Windows patching
|
+-- inventories/
|   +-- aws_ec2.yml                     # Dynamic inventory -- aws_ec2 plugin
|   +-- group_vars/
|   |   +-- all/
|   |   |   +-- main.yml                # Global vars
|   |   +-- awx_controllers/
|   |   |   +-- main.yml                # Controller-specific vars
|   |   +-- linux/
|   |   |   +-- main.yml                # Linux node vars
|   |   +-- windows/
|   |       +-- main.yml                # Windows node vars (WinRM)
|   +-- host_vars/                      # Per-host overrides (future)
|
+-- execution_environments/
|   +-- awx-ee/
|       +-- execution-environment.yml   # ansible-builder definition
|       +-- requirements.yml            # EE Python + collection deps
|
+-- awx_config/
|   +-- configure_awx.yml               # Standalone AWX config playbook
|
+-- .github/
|   +-- workflows/
|       +-- ci.yml                      # lint -> molecule -> publish -> sync
|
+-- docs/
    +-- ARCHITECTURE.md                 # This document
```

---

## Infrastructure Layer

Provisioned by Terraform in the `terraform-aws` repo under `deploy/awx/`.

| Resource | Type | Purpose |
|---|---|---|
| EC2 - AWX controller | t3.medium - RHEL 9 | Runs all AWX services |
| EC2 - linux-node-01 | t3.micro - AL2023 | Managed node - dev |
| EC2 - linux-node-02 | t3.micro - AL2023 | Managed node - prod |
| EC2 - windows-node-01 | t3.micro - WS2022 | Managed node - WinRM - dev |
| RDS - PostgreSQL 15 | db.t3.micro - 20GB | AWX database backend |
| Secrets Manager | - | DB password + admin password |
| IAM instance profile | - | EC2 to Secrets Manager + ECR + EC2 describe |
| VPC | 10.0.0.0/16 | Network isolation |
| Security groups | x3 | Least-privilege network access |

**Terraform module structure:**

```
terraform-aws/
+-- modules/
|   +-- vpc/              Public + private subnets, IGW, route tables
|   +-- iam/              AWX controller role + instance profile
|   +-- ec2/              AWX controller instance + SG + key pair
|   +-- managed-node/     Managed node instances + SG (Linux + Windows)
|   +-- rds/              PostgreSQL 15 + DB subnet group + SG
|   +-- secrets/          Secrets Manager secrets + random passwords
+-- deploy/
    +-- awx/              Root module -- wires all modules together
```

---

## AWX Installation Pattern

The controller is bootstrapped in two stages — mirroring enterprise AAP deployment practice.

**Stage 1 — Terraform user_data (minimal):**

```
dnf install ansible-core git
git clone https://github.com/deploydaily/Ansible.git
ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/install_awx.yml
```

**Stage 2 — Ansible role platform.automation.awx_install:**

```
preflight       Assert RHEL 9, memory, disk
dependencies    dnf + pip packages
setup           awx user, directories
secrets         Fetch passwords from Secrets Manager via IAM instance profile
install         git clone AWX source, pip install
configure       stat check -> generate secret key once -> persist to /etc/awx/.secret_key -> settings.py from Jinja2 template
database        migrate, collectstatic, create admin user
services        Deploy + enable 4 systemd units
nginx           Self-signed TLS, reverse proxy to :8052
healthcheck     Poll /api/v2/ping/ until ready
```

**Resulting systemd services — identical to AAP installer output:**

```
systemctl status awx-web        # Django/Daphne ASGI - port 8052
systemctl status awx-task       # Celery worker - job dispatch
systemctl status awx-receptor   # Execution mesh - receptor
systemctl status redis          # Queue + cache - port 6379
systemctl status nginx          # TLS termination - port 443
```

---

## Collection Structure

The `platform.automation` collection is the shared content catalog —
the enterprise equivalent of what teams publish to Private Automation Hub.
Dependencies declared in `requirements.yml` include `chocolatey.chocolatey`
for Windows package management via the `os_baseline` role.

| Role | Purpose | Targets |
|---|---|---|
| `os_baseline` | Packages, NTP, SSH hardening, ansible user — uses `chocolatey.chocolatey` for Windows | RHEL 9, Windows 2022 |
| `awx_install` | AWX systemd install, mirrors AAP pattern | RHEL 9 |
| `awx_config` | AWX orgs, teams, RBAC, credentials, inventories as code | AWX API |
| `os_hardening` | CIS Level 1 hardening | RHEL 9 (roadmap) |
| `patching` | dnf + Windows Update patching workflows | RHEL 9, Windows 2022 (roadmap) |

---

## Inventory Model

Dynamic inventory via the `amazon.aws.aws_ec2` plugin — nodes are
auto-discovered from EC2 tags, no static host files. `ansible_host` resolves
to `private_ip_address` — managed nodes have no public IP (see ADR-005).

**EC2 tags to AWX inventory groups:**

| EC2 Tag | Value | AWX Group |
|---|---|---|
| `role` | `controller` | `awx_controllers` |
| `os_type` | `linux` | `linux` |
| `os_type` | `windows` | `windows` |
| `env` | `dev` | `env_dev` |
| `env` | `prod` | `env_prod` |
| `managed_by` | `awx` | filter - only AWX-managed nodes |

**Resulting group hierarchy:**

```
all
+-- awx_controllers
|   +-- awx-controller
+-- linux
|   +-- linux-node-01  (env_dev)
|   +-- linux-node-02  (env_prod)
+-- windows
|   +-- windows-node-01 (env_dev)
+-- env_dev
|   +-- linux-node-01
|   +-- windows-node-01
+-- env_prod
    +-- linux-node-02
```

---

## RBAC Model

Mirrors enterprise AAP multi-tenant org structure.

```
AWX
+-- Organization: Platform-Ops
|   +-- Team: Platform-Admins      (admin on all resources)
|   +-- Team: Platform-Engineers   (execute + read)
|   +-- Credentials
|       +-- Machine - Linux nodes  (SSH key from Secrets Manager)
|       +-- Machine - Windows nodes (WinRM password from Secrets Manager)
|       +-- AWS - Dynamic inventory (IAM instance profile)
|
+-- Organization: AppDev
    +-- Team: AppDev-Engineers     (execute job templates only)
    +-- Projects
        +-- Ansible (GitHub SCM)
```

---

## SDLC Pipeline

Every push to `main` runs the full SDLC chain via GitHub Actions.

```
Push to main / PR
      |
      v
  ansible-lint            Lint all roles + playbooks
      |
      v
  molecule test           Unit test each role in Docker
      |
      v
  collection build        ansible-galaxy collection build
      |
      v
  AWX project sync        POST /api/v2/projects/<id>/update/
      |
      v
  AWX inventory sync      POST /api/v2/inventory_sources/<id>/update/
```

---

## Execution Environments

Custom EE built with `ansible-builder`, pushed to ECR.

```
execution_environments/awx-ee/
+-- execution-environment.yml   # Base image + deps definition
+-- requirements.yml            # Collections + Python packages baked in
```

Collections baked into the EE:

- `amazon.aws` — dynamic inventory + Secrets Manager lookup
- `ansible.posix` — SELinux, firewalld, systemd
- `community.general` — general Linux management
- `ansible.windows` — Windows node management
- `platform.automation` — this collection

---

## Sandbox vs Enterprise Comparison

| Component | Sandbox (this build) | Enterprise |
|---|---|---|
| AWX install | Source + systemd | AAP RPM installer (Red Hat subscription) |
| Controller | t3.medium - single node | t3.xlarge+ - HA pair |
| Database | RDS db.t3.micro - single-AZ | RDS Multi-AZ - automated backups |
| Redis | Local systemd on controller | ElastiCache - Multi-AZ |
| TLS | Self-signed on nginx | ACM cert - ALB termination |
| DNS | EC2 public IP direct | Route 53 - awx.company.com |
| Network entry | Public IP + SG | ALB in public subnet - controller in private |
| Secrets | AWS Secrets Manager | Secrets Manager + HashiCorp Vault / CyberArk |
| Content hub | Galaxy (public) | Private Automation Hub |
| Execution nodes | Receptor on controller | Dedicated nodes per network zone |
| Managed nodes | Public subnet, no public IP | Private subnet + NAT gateway |
| Node count | 3 (sandbox budget) | Hundreds - dynamic inventory at scale |

---

## Architecture Decision Records

### ADR-001 — AWX on RHEL 9 via systemd, not Kubernetes

**Decision:** Run AWX directly on RHEL 9 as systemd services.

**Reason:** The target role uses on-premises servers, not Kubernetes. The AWX systemd
service pattern (awx-web, awx-task, awx-receptor, redis, nginx) is structurally identical
to what the AAP installer produces on bare metal or VMware VMs.

**Enterprise equivalent:** AAP RPM installer on RHEL 9 — same services, same config files,
same `awx-manage` CLI. EKS only if the org already runs EKS as a platform standard.

---

### ADR-002 — Two-stage bootstrap: user_data + Ansible role

**Decision:** `user_data` installs only `ansible-core` and pulls the repo.
The Ansible `awx_install` role does all real work.

**Reason:** Shell scripts are not idempotent or testable. The role is re-runnable,
Molecule-testable, and directly demonstrates the skill set the role requires.
The `user_data` script mirrors how AAP teams bootstrap a golden RHEL image.

---

### ADR-003 — Secrets Manager over hardcoded credentials

**Decision:** All passwords fetched at runtime from Secrets Manager via IAM instance profile.

**Reason:** No credentials in code, no credentials in user_data, no credentials in logs.
The IAM instance profile scoped to `awx/*` secrets replicates the enterprise pattern of
least-privilege service accounts.

---

### ADR-004 — RDS over local PostgreSQL

**Decision:** Use RDS PostgreSQL even though it could run locally.

**Reason:** AWX state survives a controller restart. Demonstrates the enterprise pattern
of separating the database tier. In production this would be RDS Multi-AZ with automated
backups and encryption at rest — same resource, larger instance class.

---

### ADR-005 — Managed nodes in public subnet, no public IP

**Decision:** Nodes in public subnet with `associate_public_ip_address = false`.

**Reason:** ACG sandbox has no NAT gateway allowance. Nodes still have outbound internet
via IGW for package installs. AWX reaches them by private IP. SG rules allow inbound
only from the controller SG — equivalent security posture to private subnet in production.

---

### ADR-006 — Dynamic inventory over static hosts

**Decision:** `amazon.aws.aws_ec2` plugin, grouped by EC2 tags.

**Reason:** Scales to hundreds of nodes without config changes. Nodes are auto-discovered
when they come online with the correct tags. This is the standard pattern in enterprise
AWS environments running AAP.

---

### ADR-007 — AWX secret key persisted to disk, not regenerated

**Decision:** The AWX `SECRET_KEY` is generated once via `openssl rand -hex 64`,
written to `/etc/awx/.secret_key`, and read from disk on subsequent Ansible runs.

**Reason:** Regenerating the secret key on every run would invalidate all existing AWX
sessions, encrypted credentials, and tokens — effectively breaking AWX on every re-run.
A `stat` check gates generation: if the file exists, skip generation and read the
persisted value. This makes the `configure` task fully idempotent.

**Enterprise equivalent:** In AAP the secret key is set during initial install and never
rotated without a full credential re-encryption procedure.

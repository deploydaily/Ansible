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
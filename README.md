# Zero-Day PHP RCE Mitigation & Zero-Downtime Container Hotfix

**Incident Response Case Study**  
CVE-2026-109618 | CVSS 9.2 | AWS + Docker

---

## Overview

A critical unauthenticated PHP Remote Code Execution vulnerability (CVE-2026-109618, CVSS 9.2) was publicly disclosed with active in-the-wild exploitation. Our production environment hosted public-facing containerized applications alongside internal services.

Initial proposals included shutting down the virtual machines entirely. This would have disrupted internal continuous operations, background processing tasks, and certificate renewal pipelines.

This repository documents the **zero-downtime isolation and targeted hotpatch strategy** that was successfully executed instead.

---

## Situation

- Critical zero-day RCE affecting PHP
- Active exploitation reported in the wild
- Public-facing containerized applications running on AWS
- Internal services accessible only via private network that could not tolerate downtime
- Multi-service Docker Compose stack (PHP application, reverse proxy, PostgreSQL, Redis, certificate management)

---

## Action Taken

### 1. Immediate Attack Surface Reduction (Zero Downtime)

- Dynamically detached public ingress Security Groups (ports 80/443) in AWS
- Preserved internal/private network access
- External attack vector eliminated within seconds without stopping any services

### 2. Fleet Audit & Temporary Mitigations

- Audited the multi-instance fleet to confirm non-vulnerable runtimes (Ruby, Python, Go, Java) were not affected
- Applied temporary in-container runtime mitigation:

```ini
session.upload_progress.enabled = Off
```

### 3. Safe Patch Validation

- Inspected the multi-stage Dockerfile used for the PHP service
- Forced a rebuild that pulled the latest upstream base image
- Verified the patched binary (PHP 8.4.27) inside an isolated container **before** touching production:

```bash
docker run --rm <php-image> php -v
```

### 4. Targeted Hotfix & Restore

- Rebuilt and redeployed **only** the PHP application container
- Left stateful services (PostgreSQL and Redis) completely untouched
- Re-attached public Security Groups after successful verification

---

## Key Commands Used

```bash
# Check currently running version
docker run --rm <php-image> php -v
# Output showed vulnerable version (8.4.24)

# Force rebuild using latest base image
docker compose build --pull php

# Verify the new image contains the patched version
docker run --rm <php-image> php -v
# Output confirmed PHP 8.4.27

# Redeploy only the application container
docker compose up -d --force-recreate php
```

---

## Result

- Entire infrastructure protected from zero-day exploitation
- Critical internal workflows remained uninterrupted
- Public traffic restored hours ahead of any full-downtime scenario
- Minimal blast radius and fully auditable process

---

## Technical Highlights

| Area                  | Approach                                          |
|-----------------------|---------------------------------------------------|
| Network Isolation     | AWS Security Group detachment (ports 80/443)     |
| Runtime Mitigation    | `session.upload_progress.enabled = Off`          |
| Patch Validation      | Isolated `docker run --rm` testing               |
| Deployment Strategy   | Targeted container recreate only                 |
| Stateful Services     | PostgreSQL + Redis left running                  |

---

## Lessons Learned

- Security Groups provide a fast, low-risk circuit breaker for public exposure
- Always verify the actual binary version inside the running image — image tags alone are not reliable
- Multi-stage Dockerfiles and floating base image tags require an explicit `--pull` during rebuilds
- Separating public ingress from internal access paths enables true zero-downtime incident response
- Temporary runtime mitigations buy valuable time while a proper patch is validated and deployed

---

## Disclaimer

All hostnames, internal domains, image names, security group identifiers, and infrastructure details have been generalized. This write-up is intended as a technical case study of the response process only.

---

**Author**: Prasanth Narava  
**Role**: Cyber Security & Linux Engineer  
**Focus**: Incident Response • Container Security • AWS Hardening  
**Date**: 21 October 2026

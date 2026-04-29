# Illumio Policy

<!-- BADGES_START -->
![provision](https://img.shields.io/badge/provision-success-success?style=flat-square)  ![last sync](https://img.shields.io/badge/last_sync-2026--04--29-informational?style=flat-square)  ![rulesets](https://img.shields.io/badge/rulesets-4-blue?style=flat-square)  ![ip-lists](https://img.shields.io/badge/ip--lists-16-blue?style=flat-square)  ![services](https://img.shields.io/badge/services-96-blue?style=flat-square)
<!-- BADGES_END -->

Policy-as-code for Illumio PCE. Rulesets, IP lists, and services managed as YAML in Git with automated validation, security checks, traffic evidence, and provisioning.

## How It Works

```
Edit YAML ──▶ Open PR ──▶ Pipeline validates ──▶ Team reviews ──▶ Merge ──▶ Provisions to PCE draft
                              │
                              ├── YAML lint
                              ├── Security checks (8 rules)
                              ├── Traffic evidence (proves rule is needed)
                              └── PR comment with full report
```

## Repository Structure

```
scopes/                      Policy rulesets organized by scope
├── _global/                 Unscoped rulesets (e.g., coreservices)
│   ├── default.yaml
│   └── coreservices.yaml
├── payments-prod/           Team A's scope
│   ├── _scope.yaml          Scope definition (labels)
│   └── *.yaml               Rulesets
├── shareddb-prod/           Team B's scope
│   └── ...
ip-lists/                    IP list definitions
services/                    Service definitions
.illumio/                    Configuration
├── config.yaml              PCE + pipeline settings
└── security-rules.yaml      Security check rules
.github/
├── workflows/
│   ├── validate-policy.yml  Runs on PR: lint + security + evidence
│   └── provision-policy.yml Runs on merge: push to PCE draft
└── scripts/
    ├── security-check.py    Security rule evaluator
    ├── traffic-evidence.py  PCE traffic query for justification
    └── provision.py         YAML → PCE draft provisioning
CODEOWNERS                   Team ownership for PR reviews
```

## Setup

### 1. Configure GitHub Secrets

The pipeline needs PCE credentials to query traffic and provision policy. Store them as GitHub repository secrets — **never commit credentials to the repository**.

```bash
# Set each secret individually
gh secret set PCE_HOST     --repo your-org/illumio-policy --body "pce.example.com"
gh secret set PCE_PORT     --repo your-org/illumio-policy --body "8443"
gh secret set PCE_ORG_ID   --repo your-org/illumio-policy --body "1"
gh secret set PCE_API_KEY  --repo your-org/illumio-policy --body "api_xxxxxxxxxxxx"
gh secret set PCE_API_SECRET --repo your-org/illumio-policy --body "your-api-secret-here"
```

Or via the GitHub UI: **Settings → Secrets and variables → Actions → New repository secret**

| Secret | Description |
|--------|-------------|
| `PCE_HOST` | PCE hostname (e.g., `pce.example.com`) |
| `PCE_PORT` | PCE API port (typically `8443`) |
| `PCE_ORG_ID` | Organization ID (typically `1`) |
| `PCE_API_KEY` | PCE API key (starts with `api_`) |
| `PCE_API_SECRET` | PCE API secret |

**Security notes:**
- Use a **dedicated API key** for the pipeline — don't reuse personal keys
- Scope the API key to minimum required permissions (read workloads/traffic for validation, write rulesets/IP lists for provisioning)
- Rotate keys periodically
- GitHub Secrets are encrypted and never exposed in logs

### 2. Enable Branch Protection

Go to **Settings → Branches → Add rule** for `main`:
- [x] Require pull request reviews before merging
- [x] Require review from Code Owners
- [x] Require status checks to pass (select "Policy Validation")
- [x] Do not allow bypassing the above settings

### 3. Configure CODEOWNERS

Edit `CODEOWNERS` to map scopes to your teams:

```
# Each team owns their application scope
scopes/payments-prod/   @your-org/payments-team
scopes/shareddb-prod/   @your-org/database-team
scopes/ordering-prod/   @your-org/ordering-team

# Security team reviews all cross-scope changes
scopes/*/cross-scope/   @your-org/security-team
scopes/*/inbound/       @your-org/security-team

# Global policy requires security team
scopes/_global/         @your-org/security-team
ip-lists/               @your-org/security-team
```

### 4. Initial Export

The policy-gitops plugin exports your existing PCE policy to this repo automatically. If starting fresh, install the plugin:

```bash
plugger install policy-gitops
# Configure with your repo URL and a GitHub token with repo write access
plugger start policy-gitops
```

## Making Changes

### Add a new rule

1. Create a branch: `git checkout -b add-web-to-db-rule`
2. Edit the YAML file in the appropriate scope directory
3. Commit and push
4. Open a PR — the validate pipeline runs automatically
5. Review the PR comment for security findings and traffic evidence
6. Get required reviews (CODEOWNERS enforced)
7. Merge — provision pipeline pushes to PCE draft

### Example: Adding a rule

```yaml
# scopes/payments-prod/payments-intra.yaml
name: payments-prod-intra
description: "Intra-scope rules for payments"
enabled: true
scopes:
  - - label: {app: payments}
    - label: {env: prod}
rules:
  - name: web-to-db
    enabled: true
    consumers:
      - label: {role: web}
    providers:
      - label: {role: db}
    services:
      - {port: 5432, proto: tcp}
```

### Delete a rule or ruleset

Delete the YAML file and commit. The provision pipeline detects the deletion and removes the corresponding object from PCE draft.

```bash
git rm scopes/ordering-prod/old-ruleset.yaml
git commit -m "Remove old ordering ruleset"
git push
```

### Add an IP list

```yaml
# ip-lists/monitoring-servers.yaml
name: Monitoring Servers
description: "Nagios and Prometheus servers"
ip_ranges:
  - from_ip: 10.0.100.0/24
    exclusion: false
fqdns: []
```

## Security Checks

Every PR is evaluated against `.illumio/security-rules.yaml`:

| Rule | Severity | What It Catches |
|------|----------|----------------|
| SEC-001 | **Critical** (blocks PR) | Any-to-any rules (ams ↔ ams) |
| SEC-002 | **Critical** (blocks PR) | Port ranges > 1000 ports |
| SEC-003 | **Critical** (blocks PR) | Insecure protocols (FTP, Telnet, rsh) |
| SEC-004 | High (warning) | Cross-scope rules without justification |
| SEC-005 | High (warning) | RDP (3389) or SMB (445) |
| SEC-006 | High (warning) | Database ports without role-scoped consumers |
| SEC-007 | Medium (warning) | IP lists with /8 or broader CIDRs |
| SEC-008 | Medium (warning) | HTTP (80) without HTTPS |

### Exemptions

Add exemptions in `.illumio/security-rules.yaml`:

```yaml
exemptions:
  - ruleset_pattern: "coreservices"
    exempt_rules: [SEC-005]
    reason: "Active Directory requires SMB"
```

## Traffic Evidence

For each new or changed rule, the pipeline queries the PCE for **blocked traffic** matching the rule's pattern. This proves the rule is needed:

```
✅ Justified — 4,523 blocked connections over 30 days from 3 sources
⚠️ No traffic found — rule may not be needed yet
```

## What Gets Provisioned

On merge to `main`, only the **changed files** are provisioned to PCE **draft** policy. This means:
- Changes are visible in the PCE GUI under draft
- Nothing goes to active enforcement automatically
- A human must still provision draft → active in the PCE (or configure auto-provision)

## Powered By

- [Illumio Policy GitOps](https://github.com/alexgoller/illumio-policy-gitops) — sync engine and pipeline scripts
- [Illumio Plugger](https://github.com/alexgoller/illumio-plugger) — plugin framework

# Build Log

## Session 1

### Date

2026-09-18

### Phase

Phase 1 - Repository Initialization

### Objectives

- Create project repository
- Establish documentation framework
- Establish ADR framework
- Establish validation framework
- Establish runbook framework

### Activities Completed

Created GitHub repository:

```text
dev-ops-12-gitops-configuration-management-platform

---

## Secrets Governance Integration

### Overview

Completed end-to-end Secrets Governance by integrating Terraform/HCL, Azure Key Vault, Microsoft Entra ID, Azure RBAC, External Secrets Operator, Kubernetes, Git, and ArgoCD.

The validated architecture is:

```text
Private Git
↓
ArgoCD
↓
SecretStore + ExternalSecret
↓
External Secrets Operator
↓
Microsoft Entra ID
↓
Service Principal
↓
Azure RBAC
↓
Azure Key Vault
↓
Kubernetes Secret
```

### Terraform / HCL

Terraform and HCL were used to provision the Azure infrastructure required for Secrets Governance.

Validated:

- Terraform initialization, planning, application, state, and destruction
- AzureRM provider integration
- Azure infrastructure provisioning
- Azure Key Vault provisioning
- Reproducible infrastructure lifecycle

Key lesson:

```text
HCL
↓
Terraform
↓
AzureRM Provider
↓
Azure API
↓
Desired Infrastructure
```

### Identity and Least-Privilege Access

A dedicated Microsoft Entra service principal was created for External Secrets Operator.

Azure RBAC was configured using least privilege so the workload could retrieve required secret data without unnecessary administrative access.

Validated concepts:

- WHO + WHAT + WHERE
- Control plane vs. data plane
- Scope and inheritance
- Least privilege
- Service principal authentication
- Key Vault secret authorization

### External Secrets Operator

External Secrets Operator was installed using Helm.

Kyverno Label Governance initially rejected non-compliant platform resources. The deployment was corrected to meet existing governance requirements and ESO subsequently deployed successfully.

This validated that governance policies protect platform components as well as application workloads.

ESO resources validated:

- SecretStore
- ExternalSecret
- Kubernetes Secret synchronization

### GitOps Secret Management

Operational SecretStore and ExternalSecret manifests were introduced into the private Git repository.

Git contains:

- Secret references
- SecretStore configuration
- ExternalSecret configuration
- Desired-state definitions

Git does not contain:

- Azure Key Vault secret values
- Service principal client secret values
- Kubernetes Secret values
- Passwords, tokens, or private keys

ArgoCD reconciled the Secrets Governance configuration successfully.

Final validation:

```text
ArgoCD Application
Synced / Healthy

ExternalSecret
SecretSynced / Ready=True

Kubernetes Secret
Created Successfully
```

### Key Architecture Lesson

The operational model is:

```text
Git declares.
ArgoCD reconciles.
ESO retrieves.
Entra authenticates.
Azure RBAC authorizes.
Key Vault protects.
Kubernetes consumes.
```

### Repository Architecture Decision

Dev-Ops-12 will use separate operational and portfolio repositories.

During active development, the existing repository remains the private operational GitOps source of truth.

After final platform validation, the operational repository will be renamed:

```text
dev-ops-12-gitops-platform-operations
```

It will remain permanently private and continue serving as the real ArgoCD source of truth.

A separate repository with fresh Git history will then be created:

```text
dev-ops-12-gitops-configuration-management-platform
```

The new repository will contain only sanitized portfolio/reference material and will not serve as the live platform source of truth.

Guiding principle:

> Machines consume the private operational repository. Humans and recruiters consume the sanitized public portfolio repository.

Repository separation is a final release activity and does not introduce an additional engineering phase.

### Public Release Sanitization Requirements

The eventual public repository must not expose operational or environment-identifying information.

Sanitize:

- Internal IP addresses
- Internal hostnames
- Usernames
- Azure tenant or subscription identifiers
- Service principal identifiers
- Azure Key Vault names and URLs
- Azure resource IDs
- Private repository identities or URLs
- Kubernetes UIDs and resource versions
- Internal URLs
- Screenshots and terminal output containing environment identifiers

Never commit or publish:

- Passwords
- Service principal client secret values
- Azure Key Vault secret values
- Kubernetes Secret values
- Tokens
- Private keys
- Bootstrap credential contents

Sanitized examples use placeholders and generic identifiers while preserving the validated architecture.

### Current Roadmap

```text
✅ Replica Governance
✅ Label Governance
✅ Image Governance
✅ Resource Governance
✅ Kubernetes Secrets
✅ Secrets Governance

🟡 RBAC
   ✅ Azure RBAC Foundation
   ⬜ ArgoCD-Specific RBAC

⬜ PKI
⬜ Network Governance
⬜ Observability
⬜ Final Validation
```

### Resume Point

Secrets Governance is COMPLETE.

Next engineering task:

```text
ArgoCD-Specific RBAC
```

After RBAC:

```text
PKI
↓
Network Governance
↓
Observability
↓
Final Validation
```

No additional engineering phases are to be introduced.

## Final Session Takeaway

Dev-Ops-12 evolved from learning individual Kubernetes and GitOps technologies into building and validating a governed platform.

The project now demonstrates:

```text
Git
|
v
ArgoCD
|
v
Desired State
|
v
Governance
|
+-- Kyverno
+-- RBAC
+-- Secrets
+-- PKI
+-- Network Controls
|
v
Enforcement
|
v
Behavioral Validation
```
---

# ArgoCD RBAC, PKI, and Network Governance Closeout

## Session Overview

This phase completed three major platform security and governance layers:

- ArgoCD-specific RBAC
- PKI and trusted TLS
- Network Governance

The work reinforced the primary Dev-Ops-12 engineering pattern:

```text
Declare
  |
  v
Enforce
  |
  v
Behaviorally Validate
  |
  v
GitOps Manage
  |
  v
Revalidate
```

A control was not considered complete simply because a Kubernetes resource or policy existed. Enforcement and actual platform behavior were validated independently.

---

# ArgoCD-Specific RBAC

## RBAC Baseline Discovery

The existing ArgoCD RBAC configuration was inspected before changes were introduced.

Resources reviewed included:

```text
argocd-cm
argocd-rbac-cm
default AppProject
```

Configuration backups were captured before RBAC changes.

The baseline established:

- No custom platform-operator authorization policy was present.
- No platform-operator local account existed.
- Existing ArgoCD configuration needed to be preserved.
- The RBAC implementation should follow least privilege.

This created a known-good baseline before authorization changes were introduced.

---

## Least-Privilege RBAC Design

A dedicated ArgoCD operator identity was implemented.

Identity:

```text
platform-operator
```

The authorization objective was intentionally narrow:

```text
WHO

platform-operator


WHAT

GET
SYNC


WHERE

default/secrets-governance
```

Required behavior:

```text
GET secrets-governance       ALLOW

SYNC secrets-governance      ALLOW

DELETE secrets-governance    NOT GRANTED

UPDATE                        NOT GRANTED

CREATE                        NOT GRANTED

Other Applications            NOT GRANTED
```

No action wildcard was required.

No application wildcard was required.

The design intentionally granted only the permissions necessary for the operator's responsibility.

---

## Authentication and Authorization

The implementation reinforced the distinction between authentication and authorization.

```text
Authentication
=
Who are you?


Authorization
=
What are you allowed to do?
```

The ArgoCD model became:

```text
platform-operator
        |
        v
Authentication
        |
        v
ArgoCD RBAC
        |
        v
secrets-governance
```

This mirrors the security model already established during Secrets Governance:

```text
External Secrets Operator
        |
        v
Entra Service Principal
        |
        v
Authentication
        |
        v
Azure RBAC
        |
        v
Authorization
        |
        v
Azure Key Vault
```

Different technologies implement the controls, but the security model remains:

```text
Identity
  |
  v
Authentication
  |
  v
Authorization
  |
  v
Protected Resource
```

---

## ArgoCD RBAC Policy

The least-privilege policy implemented was:

```text
p, platform-operator, applications, get, default/secrets-governance, allow

p, platform-operator, applications, sync, default/secrets-governance, allow
```

This translates to:

```text
WHO
platform-operator

WHAT
get
sync

WHERE
default/secrets-governance

EFFECT
allow
```

The policy demonstrated an important difference from the Azure custom-role exercises.

It was unnecessary to create a separate NotAction-style rule for DELETE.

Instead:

```text
ALLOW GET

ALLOW SYNC

DELETE not granted

Therefore:

DELETE denied
```

Least privilege begins narrowly and expands permissions only when required.

---

## Effective RBAC Validation

Authorization was tested using the actual platform-operator identity.

Positive validation:

```text
GET default/secrets-governance
=
YES
```

```text
SYNC default/secrets-governance
=
YES
```

Negative validation:

```text
DELETE default/secrets-governance
=
NO
```

The design also prevented unintended broad access because the policy targeted:

```text
default/secrets-governance
```

rather than:

```text
default/*
```

Therefore ArgoCD-specific RBAC was behaviorally validated rather than merely configured.

---

## Declarative ArgoCD RBAC

After manual validation, ArgoCD RBAC was converted into Git-managed desired state.

Platform configuration path:

```text
kubernetes/platform/argocd/
```

RBAC manifest:

```text
kubernetes/platform/argocd/rbac-config.yaml
```

The operational model changed from:

```text
Manual Configuration
        |
        v
argocd-rbac-cm
```

to:

```text
Private Git
        |
        v
ArgoCD
        |
        v
argocd-rbac-cm
        |
        v
platform-operator
        |
        +-- GET
        |
        +-- SYNC
```

No credentials, passwords, tokens, or secret values were introduced into the declarative RBAC configuration.

---

## Dedicated ArgoCD Platform Application

A dedicated ArgoCD Application managed the platform configuration path:

```text
argocd-platform-rbac
```

Source ownership:

```text
argocd-platform-rbac
        |
        v
kubernetes/platform/argocd/
```

This established an important repository distinction:

```text
kubernetes/environments/
=
Application and environment desired state


kubernetes/platform/
=
Platform desired state
```

ArgoCD detected the difference between Git desired state and live state.

Initial state:

```text
OutOfSync
```

After synchronization:

```text
Synced
Healthy
```

ArgoCD authorization therefore became declarative GitOps-managed platform state.

---

# PKI and TLS

## PKI Discovery

PKI began with discovery rather than immediate certificate modification.

The important ArgoCD configuration discovered was:

```yaml
server.insecure: "true"
```

This setting was not changed immediately.

Instead, the existing traffic path and TLS topology were investigated first.

The key question became:

```text
Where does TLS actually terminate?
```

The architecture showed ArgoCD operating behind NGINX Ingress.

Therefore PKI design focused on the ingress security boundary rather than unnecessarily modifying the internal ArgoCD server configuration.

---

## Existing Traffic Architecture

The observed application path was:

```text
Client
  |
  v
NGINX Ingress
  |
  v
argocd-server
  |
  v
ArgoCD
```

The server backend was already functioning.

Rather than changing multiple architectural layers simultaneously, the implementation preserved the existing backend behavior while establishing trusted HTTPS at NGINX.

---

## TLS Architecture Decision

The final client traffic model became:

```text
CLIENT
  |
  | HTTPS
  | TCP 443
  v
NGINX INGRESS
  |
  | TLS Handshake
  | X.509 Certificate
  | Server Identity
  | TLS Termination
  v
HTTP
  |
  v
argocd-server
  |
  v
ArgoCD
```

The important architectural boundary became:

```text
External Client Side
=
HTTPS / TLS


Internal Backend Side
=
Existing HTTP service path
```

This allowed TLS to be introduced without destabilizing the functioning ArgoCD backend.

---

## Jenkins Change Isolation

Jenkins already had its own functioning certificate and connectivity configuration.

The PKI implementation deliberately avoided modifying Jenkins.

The boundary was:

```text
JENKINS

Existing Certificate
Existing Configuration
Existing Traffic Path

KEEP UNCHANGED
```

while:

```text
ARGOCD

Dedicated PKI
Dedicated Certificate
Dedicated Ingress TLS Configuration
```

This was a deliberate blast-radius decision.

The goal of improving ArgoCD PKI did not justify introducing unnecessary risk to another functioning platform component.

An important engineering lesson was:

> Secure the component within scope without casually modifying unrelated working infrastructure.

---

## Private PKI Hierarchy

The ArgoCD certificate architecture moved beyond a simple self-signed server certificate model.

The resulting PKI hierarchy became:

```text
SelfSigned Bootstrap Issuer
            |
            v
         Root CA
            |
            v
         CA Issuer
            |
            v
   ArgoCD Leaf Certificate
            |
            v
   Kubernetes TLS Secret
            |
            v
      NGINX Ingress
```

The Root CA acts as the trust anchor.

The CA Issuer uses the established CA to issue the ArgoCD server certificate.

The ArgoCD leaf certificate identifies the HTTPS endpoint presented by NGINX.

---

## Private CA Versus Public CA

The implementation should be described accurately.

What was implemented:

```text
Trusted HTTPS/TLS
using an internally managed private CA
```

It should not be described as:

```text
Internet Public CA TLS
```

The client trusts the ArgoCD server because the internal Root CA was deliberately installed into the client's trust store.

This distinction is important because trusted TLS does not necessarily require a certificate issued by an Internet public Certificate Authority.

---

## Root and Leaf Certificate Understanding

The PKI work reinforced the difference between the Root CA and the server leaf certificate.

Root CA:

```text
Subject
=
Root CA

Issuer
=
Root CA
```

This is expected for the self-signed trust anchor.

The ArgoCD server certificate has a different relationship:

```text
Subject
=
ArgoCD Server Identity

Issuer
=
Internal Root CA
```

Therefore:

```text
ROOT CA

Subject == Issuer
Expected


ARGOCD LEAF

Subject != Issuer
Expected
```

The ArgoCD leaf certificate was CA-signed rather than independently self-signed.

---

## Certificate Intent Versus Secret Material

PKI reinforced the same secret-management principle learned during Secrets Governance.

Git contains declarative certificate intent.

For example, Git can define:

```text
Certificate Resource

Issuer Reference

DNS Identity

TLS Secret Name
```

Git does not contain generated private key material.

Conceptually:

```text
Git
 |
 | Desired PKI State
 v
ArgoCD
 |
 v
cert-manager
 |
 +-- manages certificate lifecycle
 |
 +-- creates or manages private key material
 |
 +-- creates certificate resources
 |
 v
Kubernetes TLS Secret
 |
 v
NGINX Ingress
```

Sensitive cryptographic material must remain outside Git.

Git must not contain:

```text
TLS private keys
CA private keys
certificate Secret values
passwords
tokens
Azure credentials
service principal client secrets
```

The broader principle is:

> Git contains instructions and references. Sensitive values are generated, retrieved, or stored through the appropriate security system.

---

## Outside-In Networking Understanding

PKI also became a networking lesson.

The connection was understood outside-in:

```text
Layer 3
IP Routing
    |
    v
Layer 4
TCP 443
    |
    v
TLS / PKI
Handshake and X.509 Trust
    |
    v
Layer 7
Ingress Host Routing
    |
    v
Kubernetes Service
    |
    v
ArgoCD
```

This clarified the difference between several concepts:

```text
Layer 3
=
Where should packets travel?


Layer 4
=
Which transport and port?


PKI / TLS
=
Can the endpoint prove its identity,
and can the client trust it?


Layer 7
=
Which application should receive
the request?
```

This understanding directly prepared the platform for Network Governance.

---

## Progressive PKI Validation

PKI was not considered successful merely because cert-manager reported a certificate.

Each layer was validated independently.

Validation included:

```text
Root CA                         PASS

CA Issuer                       PASS

Bootstrap Issuer                PASS

ArgoCD Leaf Certificate         PASS

Kubernetes TLS Secret           PASS

Correct Server Identity         PASS

Correct Certificate Issuer      PASS

Correct DNS SAN                 PASS

NGINX Certificate Presentation  PASS

TCP 443                         PASS

TLS Handshake                   PASS

Ingress Routing                 PASS

ArgoCD Backend                  PASS

Explicit CA Verification        PASS

Native Client Trust

The strongest lesson from the project is:

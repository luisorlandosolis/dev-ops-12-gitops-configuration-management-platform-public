---

## Secrets Governance Lessons

### Secret References vs. Secret Values

Git should contain the desired-state configuration and references required to locate secrets, but never the secret values themselves.

```text
Git
✅ Secret references
✅ SecretStore
✅ ExternalSecret

Git
❌ Passwords
❌ Client secret values
❌ Key Vault secret values
❌ Kubernetes Secret values
❌ Tokens or private keys
```

### Authentication and Authorization Are Different

Successful workload identity requires both authentication and authorization.

```text
Microsoft Entra ID
=
Who are you?

Azure RBAC
=
What are you allowed to do?
```

A workload can authenticate successfully and still be unable to retrieve a secret if the appropriate authorization has not been granted.

### Least Privilege Matters

External Secrets Operator requires permission to retrieve the required secrets, not broad administrative control over the secret-management platform.

The validated design used a dedicated service principal with limited Key Vault secret access.

### ArgoCD Does Not Need the Secret Value

ArgoCD manages the desired-state declarations for SecretStore and ExternalSecret resources.

External Secrets Operator performs the runtime retrieval of sensitive data.

```text
Git
↓
ArgoCD
↓
ExternalSecret
↓
ESO
↓
Azure Key Vault
↓
Kubernetes Secret
```

This separates GitOps reconciliation from sensitive-value retrieval.

### Governance Applies to Platform Components

Kyverno governance applies to infrastructure and platform components in addition to application workloads.

During External Secrets Operator deployment, governance correctly rejected resources that did not meet established labeling standards.

The failed deployment became evidence that existing platform policies were actively protecting the cluster.

### Terraform and GitOps Represent Different Desired-State Layers

Terraform/HCL defines desired infrastructure state.

Git/YAML defines desired Kubernetes state.

```text
HCL
↓
Terraform
↓
Azure Infrastructure
```

and:

```text
Git / YAML
↓
ArgoCD
↓
Kubernetes
```

Both use desired-state principles but operate at different layers of the platform.

### Operational and Public Repositories Have Different Responsibilities

The operational GitOps repository and public portfolio repository serve different audiences.

```text
Private Operational Repository
=
Machines / ArgoCD / Real Platform

Public Portfolio Repository
=
Humans / Recruiters / Reference Implementation
```

The operational repository remains private and contains the real deployable configuration.

The public repository will use fresh Git history and contain only sanitized documentation, examples, diagrams, screenshots, workflows, and validation evidence.

### Sanitization Is Part of Platform Engineering

Removing secret values alone is not sufficient for safe public publication.

Public documentation must also be reviewed for environment-identifying information including:

- Internal IP addresses
- Internal hostnames
- Usernames
- Cloud account identifiers
- Service principal identifiers
- Resource identifiers
- Internal URLs
- Private repository identities
- Screenshots and terminal output

Sanitization is therefore part of the release process, not an afterthought.

### Key Takeaway

The completed Secrets Governance architecture can be summarized as:

```text
Git declares.
ArgoCD reconciles.
ESO retrieves.
Entra authenticates.
RBAC authorizes.
Key Vault protects.
Kubernetes consumes.
```
ArgoCD RBAC
PKI / private CA
Jenkins blast-radius isolation
argocd-platform-rbac naming/ownership
TLS termination
Native trust
NetworkPolicy exists != enforcement
DENY + ALLOW behavioral testing
Application ownership boundary first
Control -> Enforcement -> Validation

---

# ArgoCD RBAC, PKI, and Network Governance Lessons

## ArgoCD RBAC and Least Privilege

ArgoCD RBAC reinforced the difference between authentication and authorization.

Authentication establishes identity.

Authorization determines which actions that identity may perform.

The platform-operator identity was intentionally restricted to the minimum required permissions:

- GET: allowed
- SYNC: allowed
- DELETE: denied

The implementation demonstrated that least privilege should begin with narrowly granted permissions rather than broad access that is reduced later.

Another important lesson was that authorization controls must be tested behaviorally. The existence of an RBAC policy is not sufficient evidence that the intended access boundary works.

Positive and negative tests were therefore both required.

## GitOps Ownership Boundaries

The argocd-platform-rbac Application began with an RBAC-specific purpose but later managed PKI and ingress configuration because those resources were added beneath the same Git source path.

Renaming the existing Application solely for naming consistency was not worth introducing an unnecessary ownership migration.

The stronger lesson was architectural:

Define the platform function first.

Then define its Git ownership boundary.

Then create the ArgoCD Application.

Then introduce the resources it will manage.

Network Governance applied this lesson from the beginning by receiving its own Git path and dedicated ArgoCD Application before permanent network resources were added.

## PKI and Trusted TLS

PKI reinforced the importance of discovering the existing traffic and trust architecture before changing certificate configuration.

The initial ArgoCD configuration included server.insecure=true. Rather than treating that setting alone as proof of an insecure external architecture, the complete traffic path was traced.

The resulting model was:

Client
-> HTTPS / TLS
-> NGINX Ingress
-> TLS Termination
-> ArgoCD Backend

This demonstrated that TLS termination can occur at the ingress boundary while the backend service retains its existing internal communication model.

The implementation established an internally managed private CA hierarchy using:

SelfSigned Bootstrap Issuer
-> Root CA
-> CA Issuer
-> ArgoCD Leaf Certificate
-> Kubernetes TLS Secret
-> NGINX Ingress

An important PKI lesson was the distinction between the self-signed Root CA and the CA-signed ArgoCD leaf certificate.

The Root CA acts as the trust anchor.

The ArgoCD leaf certificate is issued by that CA and identifies the ArgoCD ingress endpoint.

The resulting implementation provides trusted HTTPS using an internally managed private CA rather than an Internet public Certificate Authority.

## Certificate Trust Must Be Tested

Certificate resources reporting a successful state did not by themselves prove trusted HTTPS.

Validation progressed through multiple layers:

- Certificate creation
- Correct certificate issuer
- Correct server identity
- Correct DNS SAN
- NGINX certificate presentation
- TCP 443 connectivity
- HTTPS routing
- Explicit CA verification
- Native operating-system trust
- Successful HTTPS response without bypassing certificate verification

The final client test succeeded without using an insecure verification bypass or manually supplying the CA certificate for each request.

This demonstrated that the client trust store, certificate chain, hostname validation, TLS handshake, ingress routing, and ArgoCD backend were functioning together.

## Protect Unrelated Working Systems

Jenkins already had a functioning certificate and traffic configuration.

It was deliberately excluded from the ArgoCD PKI change.

This reinforced an operational principle:

Do not expand the blast radius of a change simply because another component uses related technology.

The correct change should modify only the systems necessary to achieve the intended outcome.

## Git Stores PKI Intent, Not Private Keys

Git contains declarative PKI resources and references.

Private cryptographic material must remain outside Git.

The architecture follows the same principle established during Secrets Governance:

Git declares what should exist.

The appropriate security system generates, retrieves, or stores the sensitive material.

This keeps desired state declarative without turning Git into a secret store.

## Network Governance and Enforcement

Network Governance produced one of the strongest lessons in Dev-Ops-12:

NetworkPolicy exists
!=
NetworkPolicy is enforced

The cluster contained NetworkPolicy resources, but behavioral testing demonstrated that unauthorized traffic could still reach the protected workload.

This showed that a policy definition and an enforcement mechanism are separate concerns.

The validation process was:

Baseline connectivity
-> Apply DENY policy
-> Test traffic
-> Traffic still succeeds
-> Identify enforcement gap
-> Implement enforcement
-> Repeat test
-> Unauthorized traffic blocked

The negative test alone was not sufficient.

An explicit allowed path was also tested successfully.

The completed control therefore demonstrated both behaviors:

- Unauthorized traffic: BLOCKED
- Authorized traffic: ALLOWED

This reinforced that security controls must be validated through actual behavior rather than configuration state alone.

## Connectivity and Authorization Are Different

The networking work clarified the distinction between connectivity and policy.

Flannel provides Pod network connectivity.

NetworkPolicy defines which communication should be permitted.

The enforcement layer makes those policy decisions affect actual traffic.

This produced a clearer model:

Connectivity
-> Policy
-> Enforcement
-> Observed Network Behavior

## Control, Enforcement, Validation

The strongest engineering lesson across Dev-Ops-12 was repeated in multiple technologies:

Control Definition
-> Enforcement Mechanism
-> Behavioral Validation

Examples included:

Kyverno policy
-> Admission enforcement
-> Non-compliant workload rejected

ArgoCD RBAC
-> Authorization enforcement
-> GET and SYNC allowed while DELETE was denied

PKI configuration
-> TLS and certificate trust
-> Verified HTTPS connection

NetworkPolicy
-> Network enforcement
-> Unauthorized traffic actually blocked

The existence of configuration is not proof that a control works.

The platform must demonstrate the expected behavior.

## Dev-Ops-12 as a Proven Platform Foundation

Dev-Ops-12 established and validated
the governance and desired-state patterns that can now be applied by later operational platforms.

The project followed an iterative engineering model:

Design
-> Implement
-> Test
-> Observe
-> Troubleshoot
-> Correct
-> Retest
-> Validate
-> Standardize

The resulting proven patterns include:

- GitOps reconciliation
- Kubernetes governance
- Secrets Governance
- Least-privilege authorization
- Private PKI and trusted TLS
- Network Governance
- Behavioral validation
- Git and ArgoCD ownership boundaries

These patterns provide the configuration and governance foundation for Dev-Ops-13 Backup & Platform Recovery.

Dev-Ops-12 provides known-good configuration through Git and ArgoCD.

Dev-Ops-13 will combine that capability with backup, restoration, and recovery to provide known-good data and platform recovery.

This relationship allows Dev-Ops-12 to document proven platform patterns while Dev-Ops-13 demonstrates their operational application.

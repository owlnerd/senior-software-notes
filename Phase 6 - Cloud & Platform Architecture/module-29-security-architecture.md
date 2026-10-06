# Module 29 — Security Architecture: Threat Modeling, OAuth 2.0/OIDC, Microsoft Entra ID and Key Vault
*Phase 6: Cloud & Platform Architecture · Senior/Architect Interview Prep for .NET & C#*

> **Platform state verified on October 6, 2026.** Identity and security platforms have moved a lot since 2024, and several of these facts are recent enough that a stale answer is a visible signal:
>
> - **OAuth 2.1** is still an IETF Internet-Draft (**draft-ietf-oauth-v2-1-15**, March 2026), not an RFC. Its content — PKCE for every authorization-code client, no implicit grant, no password grant, exact redirect URI matching, sender-constrained or rotated refresh tokens for public clients — is already what every serious provider enforces.
> - **RFC 9700 — Best Current Practice for OAuth 2.0 Security (BCP 240)** was published in **January 2025**. It updates RFC 6749, 6750 and 6819 and formally deprecates the less secure modes. It is the document to cite when someone asks "what's current OAuth security guidance?"
> - **FAPI 2.0 Security Profile** and **Attacker Model** became **OpenID Final specifications in February 2025**; certification for the final versions opened in July 2025.
> - **MCP (Model Context Protocol) authorization** profiles OAuth 2.1: an MCP server is a resource server that **must** publish **RFC 9728 Protected Resource Metadata**; the 2025-11-25 revision recommends **Client ID Metadata Documents** for client registration. AI agents are now a mainstream OAuth client type.
> - **Microsoft.Identity.Web** is on the **4.x** line (**4.14.x** in July–August 2026), bundling **MSAL.NET 4.87** and **Microsoft.IdentityModel 8.22**. 4.4 added AOT-compatible web API authentication for .NET 10; 4.14 added mTLS token binding for OIDC federated credentials.
> - **Azure.Identity:** `DefaultAzureCredential` is documented by Microsoft as **suitable for early development, not production**; since **Azure.Identity 1.14** the **`AZURE_TOKEN_CREDENTIALS`** environment variable can restrict the chain to `prod` or `dev` credentials (and, from 1.15, to a single named credential). Use a deterministic credential (`ManagedIdentityCredential`, `WorkloadIdentityCredential`) in production.
> - **Managed identities as federated identity credentials (FICs)** on Entra app registrations went **GA in May 2025** — the secretless way for an app registration (including a multi-tenant one) to authenticate using a managed identity instead of a client secret or certificate.
> - **Mandatory Azure MFA:** Phase 1 (portal, Entra and Intune admin centers) started **October 2024**; **Phase 2 began October 1, 2025** at the Azure Resource Manager layer for **create/update/delete** operations from **any client** (CLI, PowerShell, SDKs, REST, IaC). **Workload identities are not affected** — which is exactly why automation must run as managed identities or service principals, never as users.
> - **Microsoft Entra:** **synced passkeys** and **passkey profiles** became **GA in March 2026**. **ROPC** (the password grant) remains deprecated guidance-wise and is incompatible with MFA; Microsoft's SQL drivers deprecated `ActiveDirectoryPassword` on the same grounds.
> - **Azure AD B2C:** **end of sale to new customers May 1, 2025**; **B2C Premium P2 discontinued March 15, 2026**; Microsoft continues to support existing B2C tenants **until at least May 2030**. **Microsoft Entra External ID** (GA September 2024) is the successor for customer identity (CIAM).
> - **Microsoft Entra Agent ID** — identities for AI agents — and new **Conditional Access for agents / ID Protection for agents** service plans (rolling out July–August 2026, tied to Microsoft Agent 365 / Microsoft 365 E7 licensing). Treat individual Agent ID features as preview unless the docs say GA.
> - **Azure Key Vault:** control-plane **API version 2026-02-01** makes **Azure RBAC the default** access model for **newly created** vaults (`enableRbacAuthorization = true`). **All control-plane API versions before 2026-02-01 retire on February 27, 2027** — update ARM, Bicep, Terraform and SDK calls. Access policies remain supported but are the legacy model.
> - **HSM lineup:** Key Vault **Standard** (software-protected keys), **Premium** and **Managed HSM** (FIPS 140-3 Level 3 validated HSMs, per current Microsoft and partner documentation — older pages still say 140-2), and **Azure Cloud HSM** (GA, single-tenant, the successor to **Azure Dedicated HSM**, which is retired for new customers and supported for existing ones until **July 31, 2028**).
> - **Network security perimeter (NSP)** is **GA in all public regions** for Key Vault, Storage, Event Hubs, Service Bus, Azure Monitor, AI Search and others; Cosmos DB and SQL were preview at last check.
> - **Default outbound access:** since **March 31, 2026**, **new VNets default to private subnets** — VMs need an explicit egress method (NAT Gateway, load balancer outbound rules, firewall, public IP). Existing VNets are unchanged.
> - **Public TLS certificate lifetimes (CA/B Forum Ballot SC-081v3):** **max 200 days since March 15, 2026**, **100 days from March 15, 2027**, **47 days from March 15, 2029** (domain-validation reuse falls to 10 days). Manual certificate renewal is no longer a viable operating model.
> - **.NET 10 (LTS, November 2025):** **passkeys (WebAuthn) built into ASP.NET Core Identity** (schema version 3); **cookie authentication returns 401/403 instead of login redirects for known API endpoints** (a breaking behavior change); new authentication and authorization **metrics**; **post-quantum cryptography** in `System.Security.Cryptography` — **ML-KEM** (FIPS 203), **ML-DSA** (FIPS 204), **SLH-DSA** (FIPS 205) and **Composite ML-DSA** (draft), where the OS crypto provider supports them.
> - **OWASP:** **Top 10:2025** (final list November 2025): A01 Broken Access Control, A02 Security Misconfiguration, **A03 Software Supply Chain Failures (new)**, A04 Cryptographic Failures, A05 Injection, A06 Insecure Design, A07 Authentication Failures, A08 Software or Data Integrity Failures, A09 Security Logging & Alerting Failures, **A10 Mishandling of Exceptional Conditions (new)**; SSRF folded into A01. **ASVS 5.0.0** released **May 30, 2025**. **API Security Top 10** is still the **2023** edition. **Top 10 for Agentic Applications** (December 2025) and the **2026 Top 10 for LLM Applications** (August 2026) cover AI systems.
>
> Version numbers and preview/GA states change monthly; treat them as "verified on this date", and say so in an interview when it matters.

## Orientation

Here is the sentence to carry through the whole module: **security architecture is the discipline of deciding, explicitly and in advance, who and what is trusted to do what, where the boundaries of that trust lie, and what happens when — not if — one of those boundaries fails; identity, secrets, cryptography, networks and threat models are the tools, and least privilege with assumed breach is the stance.**

Every earlier module has been quietly carrying security debt that this module pays off:

- **Module 4** listed security among the non-functional requirements; here you'll learn which security questions actually change a design.
- **Modules 11 and 27** connected services through Service Bus, Event Hubs and Cosmos DB — every one of those connections needs an identity, an authorization decision and, ideally, no shared key.
- **Module 13** taught you to design for failure; this module treats an *attacker* as a failure mode that is intelligent, patient and adversarial.
- **Module 18** covered the ASP.NET Core middleware pipeline; authentication and authorization are two of its most important stages.
- **Module 21** split a system into services; every new network hop is a new trust boundary.
- **Module 26** chose compute; each compute option has its own identity and secret-delivery story.
- **Module 28** built telemetry; security logging, audit trails and the danger of secrets in telemetry are the bridge between the two modules.

Why it matters in an interview: senior and architect loops almost never ask "what is OAuth?" They ask *"How does this service authenticate to the database?"*, *"Where do the secrets live, and who can read them?"*, *"Walk me through how a user's request reaches the downstream API — what token is presented at each hop?"*, *"Our SPA keeps the access token in localStorage — is that a problem?"*, *"How would you threat-model this design?"*, *"A client secret leaked on GitHub this morning — what do you do, and what do you change so it can't happen again?"*, *"How do you isolate tenants?"*, *"Delegated or application permissions?"*, *"Why not just use API keys?"* Weak answers name products. Strong answers name **principals, trust boundaries, tokens and their audiences, credential types ranked by risk, blast radius, and the specific failure each control mitigates.**

This module has eight jobs:

1. **Build a first-principles model** — security as a system property, trust boundaries, defense in depth, least privilege, Zero Trust, identity as the control plane, and the ranking of credentials by risk.
2. **Teach threat modeling as a design tool** — the four questions, data flow diagrams, STRIDE, prioritization, mitigation and making it a habit rather than a ceremony.
3. **Teach OAuth 2.0 and OpenID Connect precisely** — roles, tokens, JWT validation, every flow that still matters, refresh tokens, scopes and audiences, OIDC, sessions, delegation chains, browser apps, sender-constrained tokens and the attacks the current BCP exists to stop.
4. **Map identity onto Microsoft Entra ID** — tenants, app registrations, permissions, token specifics, managed identities, workload identity federation, Conditional Access, privileged access, Azure RBAC, customer identity and agents.
5. **Teach secrets, keys and certificates** — Key Vault tiers, access models, networking, recovery, envelope encryption, rotation, certificate lifecycles and consuming it all from .NET.
6. **Make .NET do it right** — the ASP.NET Core authentication and authorization pipeline, token validation, downstream calls, Azure SDK credentials, Data Protection, injection classes, Identity and passkeys, cryptography and multi-tenant isolation.
7. **Secure the platform** — network isolation, the edge, PaaS hardening, encryption in transit and at rest, supply chain, CI/CD identity, posture and detection.
8. **Make it defensible** — standards, anti-patterns, hidden costs and a design review checklist, including when *not* to add a control.

Seven framings to carry through:

1. **Every request is a claim about identity that someone must verify.** Security architecture is mostly the question "who verifies which claim, with which evidence, at which hop?"
2. **Boundaries are where security lives.** Threats concentrate where data crosses from one level of trust to another; draw those boundaries first.
3. **The best secret is the one that doesn't exist.** Rank credentials: none (managed identity, federation) beats asymmetric (certificates, keys you never export) beats shared secrets; every secret you keep is a rotation and a breach waiting to happen.
4. **Tokens are bearer instruments with an audience.** Whoever holds one can use it, so you shorten its life, narrow its audience and scope, keep it out of reach of script, and — where it matters — bind it to its holder.
5. **Least privilege is measured by blast radius.** The question is never "does it work?" but "if this identity is compromised, what can the attacker reach?"
6. **Assume breach.** Design so that one compromised component, credential or tenant does not become a compromised system: segment, isolate, log, and make revocation fast.
7. **Security is a trade-off, not a maximum.** Every control costs latency, money, developer time or user friction; a senior architect picks controls from a threat model and says which risks they are knowingly accepting.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | What security architecture is | Protect confidentiality, integrity, availability and accountability of assets against threats, proportionate to risk |
| 2 | Trust boundaries and attack surface | Threats cluster where data crosses trust levels; shrink and harden the surface |
| 3 | Defense in depth | Independent layers, so one failure is not a breach |
| 4 | Least privilege and blast radius | Grant the minimum, for the minimum time, and measure what a compromise reaches |
| 5 | Zero Trust | Verify explicitly, least privilege, assume breach — the network grants nothing |
| 6 | Authentication, authorization, accountability | Who are you, what may you do, can we prove what you did — for users, workloads and agents |
| 7 | The credential hierarchy | No secret > federated > asymmetric key > shared secret > password |
| 8 | Secure by default and the economics of security | Safe defaults, fail closed, fix it in design where it's cheapest |
| 9 | Why threat model | Four questions that turn worry into a backlog |
| 10 | Data flow diagrams | Processes, stores, flows, external entities — and the boundaries between them |
| 11 | STRIDE | Six threat categories, each the violation of one security property |
| 12 | STRIDE per element and per interaction | A mechanical walk that doesn't depend on inspiration |
| 13 | Rating and prioritizing threats | Likelihood × impact, CVSS for vulnerabilities, a risk register — not DREAD |
| 14 | Responding to threats | Mitigate, eliminate, transfer, accept — and the STRIDE-to-control map |
| 15 | Other methods | PASTA, LINDDUN, attack trees, ATT&CK, CAPEC — what each is for |
| 16 | Threat modeling in practice | Small, frequent, owned by the team, kept next to the code |
| 17 | The delegation problem | Why OAuth exists: access without sharing passwords |
| 18 | OAuth roles and channels | Resource owner, client, authorization server, resource server; front and back channel |
| 19 | Tokens | Access, refresh and ID tokens — different audiences, different jobs |
| 20 | JWTs and how to validate them | Signature, issuer, audience, lifetime, algorithm — every time |
| 21 | Authorization code flow with PKCE | The one interactive flow, step by step, with the S256 math |
| 22 | Client types and client authentication | Public vs confidential; secrets, private_key_jwt, mTLS, federated assertions |
| 23 | Client credentials | Machine-to-machine with no user |
| 24 | Device flow and the removed grants | Device authorization for TVs and CLIs; why implicit and password are gone |
| 25 | Refresh tokens | Long-lived, high-value — rotate, detect reuse, constrain |
| 26 | Scopes, roles, claims and audience | What the client may ask for, what the subject may do, who the token is for |
| 27 | OpenID Connect | Authentication on top of OAuth: ID token, nonce, discovery, JWKS |
| 28 | Sessions and logout | Cookies, token lifetimes, and the hard problem of signing out everywhere |
| 29 | Delegation chains | On-behalf-of and token exchange across service hops |
| 30 | Browser-based apps and the BFF | Keep tokens out of JavaScript; cookies done right |
| 31 | Sender-constrained tokens and high-security profiles | DPoP, mTLS, PAR, RAR, JAR, FAPI 2.0 |
| 32 | OAuth 2.1, RFC 9700 and the attacks they stop | Code injection, mix-up, redirects, leakage, CSRF — and agents as clients |
| 33 | The Entra map | Tenants; workforce, B2B, External ID; what B2C's retirement means |
| 34 | App registrations and service principals | The app's definition vs its instance in a tenant |
| 35 | Permissions in Entra | Delegated vs application, scp vs roles, consent and .default |
| 36 | Entra tokens in detail | v1 vs v2, issuers per tenant, oid/tid, lifetimes, validation traps |
| 37 | Managed identities | System- vs user-assigned, how tokens are obtained, and propagation delays |
| 38 | Workload identity federation | Trust an external issuer instead of storing a secret — GitHub, AKS, other clouds, managed identity as FIC |
| 39 | Conditional Access and phishing-resistant authentication | Signals → decision → enforcement; passkeys; policies for workloads |
| 40 | Token lifetime, revocation and CAE | Short tokens, revocation events, and continuous access evaluation |
| 41 | Privileged access | Entra roles vs Azure RBAC, PIM, break-glass, mandatory MFA |
| 42 | Azure RBAC in depth | Control vs data plane, scopes, custom roles, conditions, deny assignments |
| 43 | Customer identity (CIAM) | Entra External ID and the alternatives |
| 44 | Agents as principals | Entra Agent ID, delegated authority for AI, the agentic threat list |
| 45 | Secrets, keys and certificates | Three different things, three different lifecycles |
| 46 | Key Vault tiers and HSMs | Standard, Premium, Managed HSM, Cloud HSM — and FIPS levels |
| 47 | Key Vault access control | RBAC is the default now; why access policies were dangerous |
| 48 | Key Vault networking | Private endpoints, firewall, network security perimeter |
| 49 | Recovery, availability and limits | Soft delete, purge protection, throttling, caching |
| 50 | Envelope encryption and customer-managed keys | KEKs wrap DEKs; rotation without re-encryption |
| 51 | Secret rotation | Eliminate first; otherwise dual secrets and event-driven rotation |
| 52 | Certificates and their shrinking lifetimes | 200 days now, 47 by 2029 — automate or fail |
| 53 | Consuming Key Vault from .NET | SDK clients, configuration provider, platform references, caching and reload |
| 54 | The ASP.NET Core authentication pipeline | Schemes, handlers, challenge and forbid; .NET 10's API 401 change |
| 55 | Validating JWT bearer tokens in ASP.NET Core | Microsoft.Identity.Web or JwtBearer, configured precisely |
| 56 | Authorization in ASP.NET Core | Policies, handlers, resource-based checks, deny-by-default fallback |
| 57 | Calling downstream APIs from .NET | Token acquisition, caches, OBO, IDownstreamApi |
| 58 | Azure SDK credentials and keyless services | TokenCredential, deterministic chains, disabling local auth |
| 59 | ASP.NET Core Data Protection | The key ring behind cookies and antiforgery, persisted and protected properly |
| 60 | Injection, XSS, CSRF, CORS and SSRF | The classic web classes and the .NET defaults that stop them |
| 61 | Passwords, Identity and passkeys | When to own credentials at all; hashing, lockout, WebAuthn in .NET 10 |
| 62 | Cryptography in .NET | Use high-level primitives, never invent; AEAD, HMAC, RNG, PQC and agility |
| 63 | Multi-tenant isolation and object-level authorization | Tenant in every query, cache key and token check; BOLA and IDOR |
| 64 | Network isolation on Azure | VNets, NSGs, private endpoints and DNS, explicit egress |
| 65 | The edge | Front Door, WAF, DDoS protection, API Management as a policy enforcement point |
| 66 | PaaS hardening | Public access off, local auth off, TLS 1.2+, perimeters |
| 67 | Encryption in transit and at rest | TLS everywhere, mTLS where it pays, platform vs customer-managed keys, confidential computing |
| 68 | Software supply chain security | SBOMs, NuGet audit, lock files, signing, provenance, image scanning |
| 69 | CI/CD identity | OIDC federation, no long-lived secrets, protected environments |
| 70 | Posture, detection and response | Policy, Defender for Cloud, Sentinel, security logs and incident response |
| 71 | Standards and requirements | OWASP Top 10:2025, ASVS 5.0, API Top 10, compliance frameworks |
| 72 | Anti-patterns | The mistakes that produce breaches |
| 73 | Hidden costs and trade-offs | Latency, money, friction and operational complexity of controls |
| 74 | The security design review, and when not to | A checklist to narrate, and the restraint that signals seniority |

---
# Part A — First principles: security as a property of the system

## Concept 1 — What security architecture is

Security is a **quality attribute** of the whole system, like availability or performance (Module 4). It can't be bolted onto one component, and it's never "done." Security architecture is the part of the design that decides how the system protects what matters, against whom, at what cost.

Start with vocabulary precise enough to reason with:

| Term | Meaning | Example |
|---|---|---|
| **Asset** | Something of value to protect | Customer PII, payment tokens, the signing key, the ability to issue refunds, uptime itself |
| **Threat** | A potential cause of harm — an actor with a goal and a method | A competitor scraping prices; a criminal stealing card data; a disgruntled admin |
| **Vulnerability** | A weakness a threat can exploit | Missing authorization check on `GET /orders/{id}`; a secret in a config file |
| **Exploit / attack** | The act of using a vulnerability | Iterating order IDs to read other customers' orders |
| **Control** | A measure that reduces risk | Object-level authorization; managed identity instead of the secret |
| **Risk** | Likelihood × impact of a threat exploiting a vulnerability against an asset | High: public endpoint, trivially exploitable, exposes all customers' data |
| **Residual risk** | What remains after controls | Accepted, documented, owned |

The properties being protected are the classic **CIA triad** plus one that matters enormously in distributed systems:

- **Confidentiality** — only authorized parties can read the data.
- **Integrity** — data and behavior can't be altered without authorization, and alteration is detectable.
- **Availability** — authorized parties can use the system when they need it (denial of service is a security problem, not just a reliability one).
- **Accountability (and non-repudiation)** — actions can be attributed to a principal, and the principal can't credibly deny them later.

Three ideas distinguish an *architect's* view from a checklist view:

1. **Security is relative to a threat model.** "Is this secure?" has no answer; "is this secure against an attacker who has stolen a developer's laptop?" does. You name the adversaries you're designing against — opportunistic internet scanners, targeted criminals, malicious insiders, compromised dependencies, nation-states — and you're explicit about the ones you're not.
2. **Security is proportionate.** A hobby blog, a B2B SaaS holding contracts, and a payments platform deserve different controls. Over-securing has real costs (Concept 73): latency, money, developer friction and users who route around controls.
3. **Security is a property of interactions.** Most breaches aren't a single broken component; they're a chain — a leaked token from a log, an over-privileged identity, a flat network, a missing alert. Architecture's job is to break chains, not polish links.

**The interview-grade sentence:** *"I treat security as a system quality attribute: I identify the assets, the threats against them and the vulnerabilities they could exploit, then choose controls proportionate to likelihood times impact, protecting confidentiality, integrity, availability and accountability — and I'm explicit about which adversaries I'm designing against and which residual risks we're accepting, because most breaches are chains of small weaknesses across components, not one broken box."*

---

## Concept 2 — Trust boundaries and the attack surface

A **trust boundary** is any place where data or control passes between parts of the system that have **different levels of trust** — different owners, privileges, or exposure. Examples: the internet → your API gateway; your API → a third-party payment provider; your service → its database; tenant A's data → tenant B's request; a browser's JavaScript → your backend; a CI pipeline → production; one microservice → another owned by a different team; your code → a NuGet package.

Why boundaries matter: inside a boundary, components can (to some extent) rely on each other's assumptions. **At** a boundary, nothing that arrives can be trusted until verified. Threat concentration follows: authentication, authorization, input validation, rate limiting, encryption and logging all belong at boundaries.

The **attack surface** is the sum of all the ways an attacker can try to get data into or out of the system: every endpoint, port, message queue, file upload, admin UI, webhook receiver, management API, dependency, and human with credentials. Two levers:

1. **Reduce it.** Don't expose what doesn't need exposing: private endpoints for PaaS (Concept 64), no public management ports, disable unused features and local authentication (Concept 66), remove dead endpoints, minimize dependencies (Concept 68). The cheapest endpoint to secure is the one that doesn't exist.
2. **Harden what remains.** Strong authentication, least-privilege authorization, validation, rate limits and monitoring at every remaining entry point.

A practical habit: when you draw an architecture diagram (Module 31's C4 model), **draw the trust boundaries as dashed boxes** and label every arrow that crosses one with *how the caller is authenticated* and *what authorizes the call*. An arrow you can't label is a finding.

Boundaries aren't only network boundaries. Some of the most important are logical:

- **Tenant boundaries** in a multi-tenant system (Concept 63).
- **Privilege boundaries** — user vs admin functions in the same app.
- **Process boundaries** — a sandboxed worker processing untrusted files.
- **Supply-chain boundaries** — your build consuming third-party code (Concept 68).
- **Human boundaries** — support staff who can see customer data.
- **Model boundaries** — an LLM consuming untrusted text that can contain instructions (Concept 44).

**The interview-grade sentence:** *"I start a security design by drawing trust boundaries — internet to edge, service to database, service to third party, tenant to tenant, pipeline to production — because threats concentrate where data crosses trust levels; then I shrink the attack surface by not exposing what doesn't need exposing and harden every remaining entry with authentication, authorization, validation, rate limiting and logging. If I can't label an arrow that crosses a boundary with how the caller is authenticated and authorized, that's a finding."*

---

## Concept 3 — Defense in depth

**Defense in depth** means layering independent controls so that the failure of any single one doesn't lead to a breach. It comes from military fortification (walls, moat, keep) and, in software, rests on a sober assumption: **every control will fail eventually** — a misconfiguration, a zero-day, a stolen credential, a bug in your authorization code.

A typical layering for a .NET API on Azure:

| Layer | Controls | What it stops alone |
|---|---|---|
| Edge | Front Door + WAF, DDoS protection, TLS termination, geo/IP rules, rate limiting | Commodity attacks, volumetric floods, known exploit signatures |
| Network | Private endpoints, NSGs, no public PaaS access, explicit egress | Direct access to data stores from the internet; exfiltration to arbitrary hosts |
| Identity | Entra ID, MFA, Conditional Access, managed identities, short-lived tokens | Credential stuffing, stolen long-lived secrets |
| Application | Token validation, authorization policies, object-level checks, input validation, output encoding | BOLA, injection, privilege escalation through the API |
| Data | Encryption at rest, column-level or field-level encryption for the most sensitive data, row-level security, CMK | Disk or backup theft; some insider access |
| Secrets & keys | Key Vault, HSM-backed keys, rotation, no secrets in code | Credential leakage from repos and images |
| Detection & response | Audit logs, Defender, Sentinel, alerts, runbooks | Shortens attacker dwell time when everything else failed |

Three refinements show depth of understanding:

1. **Layers must be independent.** Two controls that share a failure mode are one control. A WAF and application validation both relying on the same regex library; network isolation and identity both managed by the same over-privileged admin account; an "encrypted" database whose key sits in the same storage account. Independence is what makes the probability of a breach the *product* of layer failure probabilities rather than the minimum.
2. **The network is not a substitute for identity.** "It's on the private network" was the classic excuse that made lateral movement easy. Modern defense in depth puts identity checks at every hop (Concept 5), with the network as an *additional* layer.
3. **Detection is a layer.** Prevention eventually fails; the question becomes how fast you notice. Mean time to detect a breach has historically been measured in months; logging, alerting and audit (Module 28, Concept 70) are what shrink it.

**The interview-grade sentence:** *"Defense in depth means independent layers — edge WAF and DDoS protection, private networking, strong identity with short-lived tokens, application-level authorization including object-level checks, encryption at rest with separately held keys, and detection — so that a single misconfiguration or stolen credential isn't a breach. The layers must fail independently, the network never substitutes for identity at each hop, and detection is a layer because prevention eventually fails."*

---

## Concept 4 — Least privilege and blast radius

**Least privilege** (Saltzer and Schroeder, 1975) means every principal — user, service, pipeline, agent — gets only the permissions it needs, only on the resources it needs, only for as long as it needs them. It has three dimensions:

| Dimension | Too broad | Least privilege |
|---|---|---|
| **Actions** | `Contributor` on the subscription | `Storage Blob Data Reader` |
| **Scope** | Subscription or resource group | The one container or Key Vault the service reads |
| **Time** | Permanent admin role | Just-in-time elevation for two hours via PIM (Concept 41) |

**Blast radius** is the measurable form: *if this identity, credential, component or tenant is compromised, what can the attacker reach?* It's the most useful question in a security review because it's concrete. Compare:

- One managed identity shared by twelve microservices with `Contributor` on the resource group → compromise of any one service compromises every data store and can delete the infrastructure.
- One user-assigned identity per service, each with a data-plane role scoped to its own resources → compromise of the order service exposes the order database, nothing else.

Techniques that shrink blast radius:

1. **One identity per workload** (sometimes per environment and region), never shared across services (Concept 37).
2. **Data-plane roles, not control-plane roles**, for applications: an app reading blobs needs `Storage Blob Data Reader`, not `Contributor` on the account — the latter can rotate the account keys and read everything (Concept 42).
3. **Narrow token audiences and scopes** — a token for the orders API should be useless at the payments API (Concept 26).
4. **Segmentation** — network, subscription, tenant and data partition boundaries that an attacker must cross separately.
5. **Separate environments** — production identities and secrets are never reachable from dev or CI pull-request builds.
6. **Time-bound access** — just-in-time admin, short token lifetimes, expiring SAS tokens.
7. **Cell or tenant isolation** for the largest blast radius of all — one compromised tenant (or cell) can't reach the others (Concept 63; cells were introduced in Module 13).

Least privilege has a known enemy: **convenience drift.** Permissions are added to unblock a deploy at 6 p.m. and never removed. Counter it with periodic access reviews, tooling that flags unused permissions (Defender for Cloud's permissions management features, Entra access reviews), and permission grants defined in code (Bicep/Terraform) so they're reviewed like any other change.

**The interview-grade sentence:** *"I apply least privilege on three axes — the fewest actions, the narrowest scope and the shortest time — and I evaluate designs by blast radius: if this identity or component is compromised, what can an attacker reach? So every workload gets its own identity with data-plane roles scoped to its own resources, tokens have narrow audiences, admin access is just-in-time, environments and tenants are segmented, and grants live in infrastructure code with periodic reviews, because permissions otherwise only ever grow."*

---

## Concept 5 — Zero Trust

**Zero Trust** is the name for the architecture that results when you stop granting trust based on network location. The term was coined by John Kindervag at Forrester (2010); Google's **BeyondCorp** (2014) was the influential implementation; **NIST SP 800-207** (2020) is the reference definition; Microsoft frames it in three principles:

1. **Verify explicitly.** Authenticate and authorize every request using all available signals — identity, device health, location, application, data sensitivity, risk — rather than "it came from inside the network."
2. **Use least-privilege access.** Just-in-time and just-enough access, risk-adaptive policies, data protection (Concept 4).
3. **Assume breach.** Minimize blast radius, segment, encrypt end to end, and use analytics to detect and respond.

What it changes in practice:

| Perimeter model | Zero Trust model |
|---|---|
| VPN puts you "inside"; inside is trusted | No implicit trust from network location; every request is authenticated and authorized |
| Service-to-service calls on the internal network are unauthenticated | Every service authenticates every caller (tokens or mTLS) and authorizes the call |
| Flat internal network | Segmented networks; private endpoints; explicit egress |
| Long-lived credentials | Short-lived tokens; continuous evaluation (Concept 40) |
| Admins have standing access | Just-in-time privileged access with approval and audit |
| Detection focused on the perimeter | Telemetry from identities, endpoints, workloads and data |

Common misreadings to correct in an interview:

- **Zero Trust is not a product.** Vendors sell components; the architecture is a set of decisions.
- **Zero Trust doesn't mean "no network controls."** It means network controls aren't the *basis* of trust. Private endpoints and segmentation remain valuable layers (Concept 3).
- **Zero Trust doesn't mean "trust nothing, ever."** It means trust is *earned per request* from verified signals and is *scoped and temporary*.
- **It applies to workloads, not just users.** Microservices authenticating each other with managed identities and validating tokens is Zero Trust in the service mesh; so is requiring workload identity federation rather than static secrets in pipelines.

**The interview-grade sentence:** *"Zero Trust, as NIST 800-207 and Microsoft frame it, means no implicit trust from network location: verify every request explicitly using identity and context signals, grant least privilege just in time, and assume breach — segment, encrypt and monitor so a compromise stays small. In a .NET system on Azure that's every service-to-service call carrying a token validated for audience and scope, managed identities instead of shared secrets, private networking as an extra layer rather than the basis of trust, just-in-time admin and continuous access evaluation."*

---

## Concept 6 — Authentication, authorization, accountability — and the kinds of principals

Three questions, three mechanisms, often confused:

| | Question | Mechanism | Output |
|---|---|---|---|
| **Authentication (AuthN)** | *Who are you?* | Credentials and proofs: passwords + MFA, passkeys, certificates, signed assertions, managed identity tokens | An authenticated **principal** with **claims** |
| **Authorization (AuthZ)** | *What may you do, here, now?* | Policies evaluated against claims, the resource and context: roles, scopes, ownership, tenant, attributes | Allow or deny |
| **Accountability** | *Can we prove what you did?* | Audit logs, tamper-evident records, signed actions | Attribution and evidence |

The architectural error that causes the most breaches is **treating authentication as authorization** — "the token is valid, so the request is allowed." A valid token proves who the caller is; it says nothing about whether *this* caller may read *this* order. That gap is OWASP's #1 risk (Broken Access Control, A01:2025) and the API Top 10's #1 (BOLA, API1:2023).

A **principal** is any entity that can be authenticated. Modern systems have four kinds, and each needs its own design:

| Principal | Examples | Typical authentication | Design notes |
|---|---|---|---|
| **Human users** | Customers, employees, partners, admins | OIDC sign-in with MFA or passkeys via an identity provider | Never handle passwords yourself if you can avoid it (Concept 61) |
| **Workloads** | Services, functions, jobs, pipelines | Managed identity, workload identity federation, certificates; client credentials flow | Most numerous principal type in cloud systems; most often over-privileged |
| **Devices** | Laptops, IoT devices, kiosks | Device certificates, device compliance signals, device flow | Device state becomes an authorization signal (Conditional Access) |
| **Agents** | AI agents acting for a user or autonomously | Delegated tokens (on behalf of a user) or their own workload-style identity | A new principal type with its own risks — goal hijacking, tool misuse (Concept 44) |

**Delegation** is when one principal acts *on behalf of* another: a web app calling an API as the signed-in user, a service calling a downstream service with the user's identity preserved, an agent booking travel for you. Delegation is what OAuth was invented for (Concept 17), and keeping the *original* principal visible across hops is what makes downstream authorization and audit meaningful (Concept 29).

**Claims** are statements about a principal issued by an authority: `sub`, `email`, `roles`, `tid`, `amr` (authentication methods used). Authorization consumes claims; it should never trust claims that the *caller* asserted about itself without an issuer's signature.

**The interview-grade sentence:** *"I keep three questions separate: authentication proves who the caller is and yields claims, authorization decides whether that principal may perform this action on this resource now, and accountability lets us prove afterwards what happened — and the classic breach is treating a valid token as permission, which is why broken access control and BOLA top the OWASP lists. I design separately for four kinds of principals — humans, workloads, devices and, increasingly, AI agents — and keep the original user visible across delegated hops so downstream services can authorize and audit correctly."*

---

## Concept 7 — The credential hierarchy: secrets are liabilities

Every credential is a potential breach. They differ enormously in how they can leak, how long a leak stays useful, and how hard they are to rotate. Ranking them is one of the most practical tools an architect has:

| Rank | Credential type | Examples | How it leaks | Damage window |
|---|---|---|---|---|
| **1 (best)** | **No stored credential** — platform-issued identity | Managed identity; AKS workload identity; App Service / Functions / Container Apps identity | Only by compromising the running workload itself | Token lifetime (hours), revocable by removing role assignments |
| **2** | **Federated credential** — trust an external issuer's short-lived token | GitHub Actions OIDC → Entra; Kubernetes service-account token → Entra; AWS/GCP identity → Entra; managed identity as FIC | Compromise of the issuer or the exact subject (repo + branch/environment) | Minutes to an hour |
| **3** | **Asymmetric key / certificate, non-exportable** | Certificate in Key Vault used via signing operations; HSM-backed keys; client assertions signed by Key Vault | Hard — the private key never leaves the HSM/vault | Until revoked; signing requires vault access |
| **4** | **Asymmetric key / certificate, exportable** | PFX file on disk; certificate in a container image | Copying the file | Certificate lifetime (often a year) |
| **5** | **Shared secret** | Client secrets, API keys, storage account keys, SAS tokens, connection strings with passwords | Logs, repos, screenshots, CI variables, config dumps, browser history | Until rotated — often never |
| **6 (worst)** | **Human password used by automation** | Service accounts with passwords, ROPC flows | All of the above plus phishing, plus MFA can't apply | Until changed; and now broken by mandatory MFA |

The rule that follows: **eliminate before you protect.** The first question for any secret is not "where do we store it?" but "can we not have it?" Most Azure services accept Entra tokens for data-plane access (Storage, Service Bus, Event Hubs, Cosmos DB, Azure SQL, Key Vault, App Configuration, Azure OpenAI, Azure Monitor ingestion) — so the connection string with a key can be replaced by an endpoint plus a managed identity (Concept 58). Only when elimination is impossible (a third-party API that only issues API keys) do you store the secret in Key Vault, restrict who can read it, and rotate it (Concept 51).

Why shared secrets are so dangerous, specifically:

- **They are bearer credentials with no binding** — whoever holds the string *is* the client.
- **They are copied by design** — into config, CI variables, developer machines, incident tickets.
- **They are long-lived** because rotation is painful, so a leak from two years ago may still work.
- **They are invisible** — you rarely know a copy exists until it's used.
- **Secret-scanning statistics are grim**: GitHub and others report millions of secrets pushed to public repositories each year, and automated attackers scan new commits within minutes.

**The interview-grade sentence:** *"I rank credentials by how they leak and how long a leak stays useful: no stored credential at all — managed or workload identity — is best, then federated short-lived tokens from GitHub or Kubernetes, then non-exportable asymmetric keys in an HSM, then exportable certificates, then shared secrets like API keys and connection strings, and worst of all human passwords used by automation. So my first move with any secret is to eliminate it — most Azure data planes accept Entra tokens — and only what genuinely can't be eliminated goes into Key Vault with narrow read access and rotation."*

---

## Concept 8 — Secure by default, fail securely, and the economics of security

Four principles from the classic secure-design literature (Saltzer & Schroeder; Microsoft's SDL; OWASP) that shape every later decision:

**1. Secure by default.** The default configuration should be the safe one; insecurity must be an explicit, visible, reviewed choice. Examples on this platform: Key Vault now defaults to RBAC (2026-02-01 API); new VNets default to private subnets (since March 31, 2026); ASP.NET Core's `[Authorize]` fallback policy (Concept 56) can make *every* endpoint require authentication unless explicitly marked `AllowAnonymous`; `HttpOnly` and `Secure` cookies by default; HTTPS redirection and HSTS in templates. In your own designs: deny-by-default authorization, private-by-default data stores, opt-in public exposure.

**2. Fail securely (fail closed).** When a security check can't complete — the token validator can't fetch signing keys, the policy engine times out, the tenant lookup fails — the request is **denied**, not allowed. This collides with Module 13's availability instincts, so be precise: *security decisions* fail closed; *non-security features* may degrade gracefully. A classic bug: `catch { return true; }` in an authorization handler. OWASP elevated this to its own category in 2025 (A10, Mishandling of Exceptional Conditions).

**3. Economy of mechanism and complete mediation.** Keep security mechanisms small and centralized enough to review, and check authorization on **every** access, not just the first (no "we checked at the gateway so the service doesn't need to"). Centralize the *mechanism* (one authorization library, one token validation configuration) and distribute the *enforcement* (every service enforces).

**4. Psychological acceptability.** Controls that are painful get bypassed: users write down passwords, developers copy production secrets locally, admins keep standing access. Choose controls that make the secure path the easy path — passkeys instead of complex password rules, managed identity instead of secret distribution, `dotnet user-secrets` and developer Entra identities instead of shared dev keys.

**The economics.** Fixing a security flaw costs more the later it's found — a design-time threat model costs hours; a production breach costs incident response, notification, regulatory fines (GDPR's ceiling is 4% of global turnover), customer churn and engineering months. The precise multipliers often quoted are disputed, but the direction isn't. This is the argument for **shifting left**: threat modeling in design (Part B), secure defaults in templates, static analysis and dependency scanning in CI (Concept 68), and security review as part of architecture review — rather than a penetration test the week before launch.

**The interview-grade sentence:** *"I design secure by default — deny-by-default authorization, private-by-default data stores, safe framework defaults — so insecurity is always an explicit, reviewed choice; security checks fail closed even though non-security features may degrade gracefully; mechanisms are centralized and small enough to review while enforcement happens at every access; and the secure path is made the easy path, because controls that hurt get bypassed. And I push the work left into design and CI, because a threat model costs hours while a breach costs months."*

---
# Part B — Threat modeling as a design tool

## Concept 9 — Why threat model: the four questions

**Threat modeling** is the structured analysis of a design to find what can go wrong, security-wise, *before* it's built (or before the next change ships). It's a design review technique, not a document type. Adam Shostack's formulation — now the basis of the **Threat Modeling Manifesto** (2020) — reduces it to four questions:

1. **What are we working on?** — a model of the system: a data flow diagram (Concept 10), its trust boundaries, its assets.
2. **What can go wrong?** — threat enumeration, usually with STRIDE (Concept 11).
3. **What are we going to do about it?** — mitigations, eliminations, transfers and accepted risks (Concept 14), turned into backlog items.
4. **Did we do a good enough job?** — validation: were the mitigations implemented and tested, did the model match reality, what did we miss?

Why it's worth the time:

- **It finds design flaws, which scanners can't.** SAST and DAST find implementation bugs (an injection, a missing header). They can't tell you that the refund API lets a support agent refund to an arbitrary account, or that a tenant ID is taken from the request body instead of the token. Those are design flaws — OWASP's A06:2025 Insecure Design — and they're the expensive ones.
- **It's cheap at design time.** An hour with a whiteboard can remove a whole vulnerability class (for example by choosing managed identity over a shared key, or the BFF pattern over tokens in the browser).
- **It produces shared understanding.** Engineers, security and product agree on what's being protected, from whom, and what's deliberately accepted.
- **It's evidence.** Many compliance regimes and customer security questionnaires ask for it; Microsoft's SDL has required it for decades.

What it isn't: a 60-page document produced once and filed; an exercise only security specialists can do; a guarantee. The Manifesto's values are worth quoting in spirit: *a culture of finding and fixing design issues over checkbox compliance; people and collaboration over processes and tools; a journey of understanding over a security or privacy snapshot; doing threat modeling over talking about it; continuous refinement over a single delivery.*

**The interview-grade sentence:** *"Threat modeling answers four questions — what are we working on, what can go wrong, what are we going to do about it, and did we do a good enough job — and I run it at design time because it finds design flaws, like a tenant ID taken from the request body or an over-privileged refund path, that no scanner can find, at the cost of an hour rather than an incident. It's a team habit that produces backlog items and documented accepted risks, not a one-off document."*

---

## Concept 10 — Data flow diagrams and trust boundaries

The standard model for question 1 is a **data flow diagram (DFD)**. It's deliberately simpler than a C4 container diagram: it shows how *data* moves, which is what attackers care about. Four element types plus one annotation:

| Element | Symbol (convention) | Meaning | Examples |
|---|---|---|---|
| **External entity** | Rectangle | Something outside your control that sends or receives data | User's browser, partner system, payment provider, Entra ID |
| **Process** | Circle / rounded box | Code you run that transforms data | Orders API, checkout worker, Azure Function |
| **Data store** | Two parallel lines | Data at rest | Azure SQL, Cosmos DB container, Blob container, Key Vault, a queue, a log |
| **Data flow** | Arrow | Data in motion between elements | HTTPS request, Service Bus message, SQL query |
| **Trust boundary** | Dashed line / box | Where trust levels change | Internet ↔ edge; app subnet ↔ data; tenant ↔ tenant; your org ↔ vendor |

A worked DFD for a checkout feature (reused in Worked example 2):

```
                ┌───────────────────── Internet (untrusted) ─────────────────────┐
  [Customer browser] ──(1) HTTPS: cart, cookie──►  
                └────────────────────────────────────────────────────────────────┘
  - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - trust boundary A
  ( Front Door + WAF ) ──(2) HTTPS──► ( Checkout BFF ) ──(3) HTTPS + access token──► ( Orders API )
                                            │                                          │
                                            │(4) OIDC                                  │(5) TDS, Entra token
                                            ▼                                          ▼
                                       [Entra External ID]                        ═══ Orders DB ═══
  - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - trust boundary B
                         ( Orders API ) ──(6) HTTPS, mTLS/API key──► [Payment provider]
                         ( Orders API ) ──(7) AMQP, managed identity──► ═══ Service Bus: orders-events ═══
                                                                            │(8)
                                                                            ▼
                                                                  ( Fulfillment worker ) ──(9)──► [Warehouse system]
```

Rules for useful DFDs:

1. **Start at the right altitude.** One diagram per feature or bounded context (Module 22), not the whole company. If it doesn't fit on a screen, it's too big to threat-model in one session.
2. **Number the flows.** Threats are discussed per flow ("flow 6: the payment callback").
3. **Draw every trust boundary.** Each boundary crossing is a concentration of threats; a DFD without boundaries is just a diagram.
4. **Include the identity provider, the secret store and the logs.** They're often missing and often the target.
5. **Include administrative and operational flows** — the CI/CD pipeline deploying the service, the support tool reading the database, the backup job. Many real breaches go through them.
6. **Annotate assets on stores and flows** — which carry PII, payment data, credentials, tenant data.
7. **Keep it next to the code** (as Mermaid, PlantUML, a Threat Dragon JSON file or a Threagile YAML file) so it changes with the design.

**The interview-grade sentence:** *"I model the feature as a data flow diagram — external entities, processes, data stores and numbered data flows — with every trust boundary drawn and the assets on each store and flow labeled, at the altitude of one feature or bounded context, including the identity provider, the secret store, the logs and the operational flows like CI/CD and support tools, and I keep it as code next to the service so it evolves with the design."*

---

## Concept 11 — STRIDE: six threats, six properties

**STRIDE** was created at Microsoft by Loren Kohnfelder and Praerit Garg (1999). Its value is that each letter is the **violation of one security property**, so walking it methodically covers the whole space:

| Threat | Violates | Question to ask | Examples | Typical controls |
|---|---|---|---|---|
| **S**poofing | **Authentication** | Can someone pretend to be another user, service or system? | Stolen session cookie; forged JWT with `alg: none`; webhook calls with no signature; spoofed `X-Forwarded-For`; DNS hijack | Strong authentication (MFA, passkeys), signed tokens validated properly, mTLS, webhook HMAC signatures, managed identities |
| **T**ampering | **Integrity** | Can someone modify data in transit, at rest, or code? | Changing the price in a client-side cart; altering a message on the queue; modifying a container image; SQL injection writing data | TLS, signatures/HMAC, server-side recomputation, parameterized queries, immutable images with signature verification, append-only stores |
| **R**epudiation | **Non-repudiation (accountability)** | Can someone deny doing something, and could we prove otherwise? | Admin deletes records with no audit; user disputes a payment approval; shared service account makes attribution impossible | Audit logs written by the system (not the actor), per-principal identities, tamper-evident storage, signed approvals |
| **I**nformation disclosure | **Confidentiality** | Can someone read data they shouldn't? | IDOR/BOLA; verbose error pages with stack traces; secrets in logs; public blob container; side channels; over-broad API responses | Authorization on every object, encryption, private endpoints, response filtering, redaction (Module 28), least privilege |
| **D**enial of service | **Availability** | Can someone make the system unavailable or too expensive? | Request floods; expensive GraphQL queries; regex backtracking (ReDoS); unbounded uploads; exhausting a shared rate limit; cost attacks on autoscaling or telemetry | Rate limiting, quotas, request size and complexity limits, timeouts, autoscale ceilings, DDoS protection, bulkheads (Module 13) |
| **E**levation of privilege | **Authorization** | Can someone do something they're not allowed to? | Normal user calling an admin endpoint; mass assignment setting `IsAdmin`; tenant A acting in tenant B; container escape; over-privileged managed identity | Authorization policies at every endpoint, deny by default, function- and object-level checks, least-privilege identities, sandboxing |

Two subtleties that come up in interviews:

- **Spoofing vs elevation of privilege.** Spoofing is *becoming someone else*; elevation is *doing more than you're allowed as yourself* (or than the identity you spoofed is allowed). Many attacks chain them.
- **Repudiation is the most neglected letter.** Teams rarely ask "if a privileged action is disputed, can we prove who did it?" — until an insider incident or a payment dispute. It's where diagnostic logs (sampled, mutable, short-lived) get confused with audit trails (complete, tamper-evident, retained) — Module 28, Concept 40.

**The interview-grade sentence:** *"STRIDE works because each letter is the violation of one property: spoofing breaks authentication, tampering breaks integrity, repudiation breaks accountability, information disclosure breaks confidentiality, denial of service breaks availability and elevation of privilege breaks authorization — so walking all six over a design covers the space, maps each threat to a family of controls, and forces me to ask the neglected question of whether we could prove who performed a disputed privileged action."*

---

## Concept 12 — STRIDE per element and per interaction

Applying STRIDE "to the system" invites hand-waving. Two mechanical variants make it systematic.

**STRIDE per element** — apply only the categories that are meaningful for each DFD element type:

| Element type | S | T | R | I | D | E |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| External entity | ✓ | | ✓ | | | |
| Process | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Data store | | ✓ | ✓* | ✓ | ✓ | |
| Data flow | | ✓ | | ✓ | ✓ | |

\* Repudiation applies to data stores that *are* logs or audit records (can the log be tampered with or deleted to hide an action?).

So the checkout DFD's Orders DB gets tampering, information disclosure, denial of service (and repudiation if it holds the audit table); flow 6 to the payment provider gets tampering, information disclosure and denial of service; the Orders API process gets all six.

**STRIDE per interaction** — the approach Microsoft's Threat Modeling Tool has used since 2014: for every **flow crossing a trust boundary**, enumerate threats against the source, the flow and the destination of that interaction. It produces more threats but stays focused on boundaries, which is where they concentrate. A practical blend: per-interaction for every boundary-crossing flow, per-element for the processes and stores holding the most sensitive assets.

**Running a session** (60–90 minutes for one feature):

1. Ten minutes: walk the DFD; confirm boundaries and assets with the people who built it.
2. Forty minutes: walk elements or interactions, STRIDE letter by letter. Write every threat down, even if it seems mitigated — note the existing mitigation instead of skipping it.
3. Twenty minutes: rate and decide (Concepts 13–14); create backlog items with owners.
4. Afterwards: the threat list lives next to the DFD in the repo.

Phrasing threats well makes them actionable. Use the form *"[actor] can [action] [asset/element] by [method], resulting in [impact]"*: *"An authenticated customer can read another tenant's orders by changing the `orderId` in `GET /orders/{id}`, resulting in cross-tenant data disclosure."* That sentence already suggests its test case.

**Elevation of Privilege** (the card game Microsoft released in 2010, with OWASP's **Cornucopia** as a web-focused cousin) is a good way to get developers generating threats in a session; each card is a concrete threat prompt by STRIDE category.

**The interview-grade sentence:** *"I apply STRIDE mechanically: per element — external entities get spoofing and repudiation, data flows get tampering, disclosure and denial of service, data stores the same plus repudiation when they're logs, and processes all six — and per interaction for every flow that crosses a trust boundary, which is how Microsoft's tool generates threats. Each threat is written as 'actor can action asset by method resulting in impact', so it already implies its mitigation and its test."*

---

## Concept 13 — Rating and prioritizing threats

A session easily produces 40 threats; you can't fix them all at once. Prioritization should be quick, consistent and defensible.

**What not to use: DREAD.** DREAD (Damage, Reproducibility, Exploitability, Affected users, Discoverability, each scored 1–10) was Microsoft's early scheme. Microsoft itself stopped using it because scores were subjective and inconsistent between raters — "Discoverability" in particular rewards security through obscurity. You'll still meet it; know why it fell out of favor.

**What works:**

1. **Likelihood × impact on a small scale** (e.g., Low/Medium/High each), producing a 3×3 risk matrix. Likelihood considers exposure (internet-facing vs internal), attacker skill required, preconditions (authenticated? specific role?), and existing mitigations. Impact considers data sensitivity, number of tenants/users affected, financial and regulatory consequences, and reversibility.
2. **Bug bars** — Microsoft's SDL approach: a pre-agreed table mapping classes of issues to severities ("remote unauthenticated information disclosure of PII across tenants = Critical"; "self-XSS = Low"). Bug bars remove debate during the session.
3. **CVSS** (Common Vulnerability Scoring System, v4.0 since 2023) for *vulnerabilities* — concrete, known flaws, especially in dependencies — not for design threats. It's the language of CVEs, scanners and patch SLAs.
4. **Business-impact framing** for leadership: map the top risks to money, regulatory exposure and customer trust (this is where PASTA, Concept 15, shines).

**A simple risk register** row:

| ID | Threat | Element | STRIDE | Likelihood | Impact | Risk | Response | Owner | Status |
|---|---|---|---|---|---|---|---|---|---|
| T-07 | Customer reads another tenant's order by changing `orderId` | Orders API | I, E | High | High | **Critical** | Mitigate: tenant-scoped query + object-level policy + integration test | Orders team | In progress |
| T-12 | Payment callback forged by attacker | Flow 6 | S, T | Medium | High | High | Mitigate: verify provider HMAC signature, allow-list source IPs, idempotent processing | Payments | Done |
| T-19 | Volumetric flood on checkout | Edge | D | Medium | Medium | Medium | Transfer/mitigate: Front Door + DDoS protection + rate limit per client | Platform | Done |
| T-23 | Insider with DB admin reads card metadata | Orders DB | I | Low | High | Medium | Accept for now; PIM + audit; revisit with Always Encrypted next quarter | CISO | Accepted until Q1 |

**Severity SLAs** close the loop: Critical before release, High within the sprint, Medium within the quarter, Low tracked. Accepted risks get an owner with authority to accept them, and an expiry date.

**The interview-grade sentence:** *"I prioritize threats with likelihood times impact on a small scale or a pre-agreed bug bar — not DREAD, which Microsoft dropped because scores were subjective — reserve CVSS for concrete vulnerabilities like dependency CVEs, and keep a risk register with response, owner, status and fix-by SLAs, where every accepted risk has an owner with the authority to accept it and an expiry date."*

---

## Concept 14 — Responding to threats: the four responses and the STRIDE-to-control map

For each prioritized threat, choose one of four responses:

| Response | Meaning | Example |
|---|---|---|
| **Mitigate** | Add or strengthen a control to reduce likelihood or impact | Add object-level authorization; validate the webhook signature |
| **Eliminate** | Remove the feature, data or component that creates the threat | Don't store card numbers at all — use the provider's tokenization; delete the unused admin endpoint; replace the shared key with managed identity |
| **Transfer** | Move the risk to a party better placed to handle it | Use a PCI-compliant payment provider's hosted fields; buy DDoS protection; use a managed identity provider rather than storing passwords; cyber insurance |
| **Accept** | Consciously live with it, documented and owned | Low-impact, low-likelihood threats; risks whose mitigation costs more than the exposure |

**Elimination is the most underused and most powerful.** The data you don't store can't leak; the secret you don't have can't be stolen; the endpoint you don't expose can't be attacked. Senior candidates reach for elimination first.

**The STRIDE-to-control map on Azure/.NET** — the starting point for "what are we going to do about it?":

| STRIDE | Platform controls (Azure) | Application controls (.NET) |
|---|---|---|
| **S** | Entra ID with MFA/passkeys and Conditional Access; managed identities; workload identity federation; mTLS at Front Door/APIM/App Gateway | `AddMicrosoftIdentityWebApi` / JwtBearer with strict validation (Concept 55); webhook signature verification; antiforgery for cookie apps |
| **T** | TLS everywhere; immutable storage (WORM); signed container images; Azure Policy preventing config drift | Parameterized queries (EF Core); server-side recomputation of prices and totals; HMAC/signatures on messages; optimistic concurrency (Module 19) |
| **R** | Entra sign-in and audit logs; Azure Activity Log; immutable log archive; per-workload identities | Audit trail written transactionally with the change (Module 28 Concept 40); user identity propagated in claims across hops |
| **I** | Private endpoints; disabled public network access; encryption at rest (platform or CMK); Key Vault; RBAC | Object-level authorization; DTO projection instead of entity serialization; redaction; generic error responses (ProblemDetails without stack traces) |
| **D** | Front Door + WAF; DDoS Network/IP Protection; APIM rate limits and quotas; autoscale ceilings | ASP.NET Core rate limiting middleware; request size limits (Kestrel); timeouts; Polly bulkheads (Module 25); `RegexOptions.NonBacktracking` or timeouts |
| **E** | Azure RBAC least privilege; PIM; Conditional Access; container hardening | Authorization policies with deny-by-default fallback; function-level and resource-based checks; avoid mass assignment by binding to explicit DTOs |

**Validating mitigations (question 4).** Each mitigation should come with evidence: a unit or integration test (e.g., "tenant B's token gets 404 on tenant A's order"), a policy in Azure Policy that denies the bad configuration, a pipeline check, or a penetration test finding closed. A mitigation without a test is a hope.

**The interview-grade sentence:** *"For each threat I choose to mitigate, eliminate, transfer or accept — and I reach for elimination first, because data we don't store, secrets we don't have and endpoints we don't expose can't be attacked. Then I map STRIDE to concrete controls on both layers — Entra, managed identities, private endpoints, WAF and policy on the platform; strict token validation, object-level authorization, parameterized queries, rate limiting and transactional audit in the code — and every mitigation gets evidence, ideally an automated test or a policy that denies the bad configuration."*

---

## Concept 15 — Other methods and catalogs, and what each is for

STRIDE is a *threat taxonomy* applied to a *design model*. Other methods answer different questions; a senior engineer knows when to reach for which.

| Method / catalog | What it is | Use it for |
|---|---|---|
| **PASTA** (Process for Attack Simulation and Threat Analysis, 2015) | Seven-stage, risk-centric process starting from business objectives through attack simulation to risk analysis | Crown-jewel systems; outputs leadership understands; aligning security spend to business risk |
| **LINDDUN** (KU Leuven) | Privacy threat taxonomy: Linking, Identifying, Non-repudiation (as a *privacy* threat), Detecting, Data disclosure, Unawareness, Non-compliance | Systems processing personal data; GDPR data protection impact assessments; used alongside STRIDE |
| **Attack trees** (Schneier, 1999) | Root goal ("steal customer data") decomposed into AND/OR subgoals with costs or probabilities | Deep analysis of one high-value goal; explaining why a mitigation cuts many branches |
| **MITRE ATT&CK** | Knowledge base of real adversary tactics and techniques observed in the wild (initial access, persistence, lateral movement…) | Detection engineering, red teaming, checking that your logging would see real techniques |
| **MITRE CAPEC** | Catalog of common attack patterns against software | Making threat brainstorming concrete; mapping threats to known patterns |
| **CWE** | Catalog of weakness types (CWE-79 XSS, CWE-639 authorization bypass through user-controlled key…) | Linking threats and findings to standard weakness IDs; OWASP Top 10 categories are CWE groupings |
| **Kill chain** (Lockheed Martin, 2011) | Stages of an intrusion: reconnaissance → weaponization → delivery → exploitation → installation → command and control → actions on objectives | Thinking about where to break the chain and where to detect |
| **OWASP Top 10 / API Top 10 / LLM and Agentic Top 10s** | Awareness lists of the most common risk categories | Sanity checks, training, prioritizing review attention (Concept 71) |
| **OWASP ASVS 5.0** | ~350 verifiable requirements in 17 chapters, three levels | Turning "be secure" into testable requirements and acceptance criteria |

The common combination in mature organizations: STRIDE per feature inside the team's normal design process; LINDDUN where personal data is involved; an attack tree or PASTA-style analysis for the few crown-jewel assets once a year; ATT&CK to check detection coverage; ASVS to set requirements.

**The interview-grade sentence:** *"STRIDE is my default for feature-level design review, but I pick the method by the question: LINDDUN for privacy threats in personal-data systems, PASTA or attack trees for deep risk analysis of crown-jewel assets in business terms, MITRE ATT&CK to check whether our detection would actually see real adversary techniques, CAPEC and CWE to make threats concrete and standard, and ASVS 5.0 to turn the outcome into verifiable requirements."*

---

## Concept 16 — Threat modeling in practice

The failure mode of threat modeling is not doing it badly — it's doing it once. Practices that make it stick:

**When:**
- At **design time** for every new service, feature with a new trust boundary, new data category (PII, payments, health), new third-party integration, or new authentication flow.
- On **significant change** — a new endpoint class, a new tenant model, moving from internal to internet-facing, adding an AI agent with tools.
- **Periodically** for crown jewels (yearly), and after incidents (what did the model miss?).
- *Not* for every pull request; a lightweight checklist question in the PR template ("does this change cross a trust boundary or touch auth?") routes the ones that need it.

**Who:** the team that builds and runs it, with a security champion facilitating and a security engineer available for the hard cases. Developers know the system; security people know the attacks; product knows what matters. Security champions programs (one trained engineer per team) scale this better than a central team reviewing everything.

**How long:** 60–90 minutes per feature-sized model. If it takes a week, the scope is too big or the process too heavy.

**Artifacts** that live in the repository next to the code:
- the DFD (Mermaid/PlantUML/Threat Dragon/Threagile file);
- the threat list with STRIDE category, rating, response, owner, status and link to the backlog item or test;
- accepted risks with owner and expiry.

**Tools** (all free):

| Tool | Notes |
|---|---|
| **Microsoft Threat Modeling Tool** | Windows desktop; draws DFDs with Azure stencils; generates STRIDE threats per interaction; reports. Good for teams on Windows in Microsoft-centric estates |
| **OWASP Threat Dragon** | Web/desktop, cross-platform, stores models as JSON in the repo; STRIDE, LINDDUN and other rule sets |
| **Threagile** | Threat model as YAML; runs risk rules in CI; produces diagrams and reports — "threat modeling as code" |
| **pytm** (OWASP) | Python DSL describing the system; generates DFDs and threats |
| **Plain Mermaid + a Markdown table** | Often enough; lowest friction |

**Connecting to the rest of the lifecycle:** threats become user stories or acceptance criteria (often mapped to ASVS requirements), tests (Concept 14), Azure Policy rules, and penetration test scope; production incidents feed back into the model. That loop is what Microsoft's **Security Development Lifecycle (SDL)** formalizes: training, requirements, design (threat modeling), implementation (approved tools, static analysis), verification (dynamic analysis, fuzzing, pen tests), release (incident response plan) and response.

**The interview-grade sentence:** *"I make threat modeling a habit rather than a ceremony: a 60-to-90-minute session run by the owning team with a security champion whenever a change adds a trust boundary, a data category, a third party, an auth flow or an AI agent with tools — routed by a PR-template question rather than done for every PR — with the DFD and threat list kept as code in the repo, threats turned into backlog items, tests and policies, and incidents fed back into the model, which is the design step of the SDL done continuously."*

---
# Part C — OAuth 2.0 and OpenID Connect, precisely

## Concept 17 — The delegation problem: why OAuth exists

Before OAuth (RFC 6749, 2012; OAuth 1.0 in 2010), a third-party app that wanted to read your photos from a photo site asked for **your username and password**. That "password anti-pattern" had five fatal properties:

1. The app gets **full access** — everything you can do, not just "read photos."
2. Access is **indefinite** — until you change your password.
3. **Revoking one app** means changing your password and breaking every other app.
4. The app can **store your password** badly; one breach of the app breaches your account.
5. **MFA is impossible** — the app can't answer your second factor.

OAuth solves *delegated authorization*: the user (the **resource owner**) authenticates directly with the service that holds the data (the **authorization server**), approves a limited grant, and the app (the **client**) receives an **access token** — a credential that is **scoped** (only what was approved), **time-limited**, **revocable per app**, and **never contains the password**.

Two clarifications that interviewers probe:

- **OAuth is an authorization framework, not an authentication protocol.** An access token says "the bearer may call this API with these scopes"; it was never designed to tell the *client* who the user is. Using OAuth access tokens as proof of login caused real vulnerabilities (a client accepting an access token issued to a *different* app). **OpenID Connect** (Concept 27) adds authentication properly.
- **OAuth is a framework, not a single protocol.** RFC 6749 defines roles and grant types and leaves many choices open; security comes from the profile you choose — RFC 9700's best current practice, OAuth 2.1, FAPI 2.0 — and from implementing validation correctly.

The family of specifications you should recognize:

| RFC / spec | What it adds |
|---|---|
| RFC 6749 / 6750 | OAuth 2.0 framework; bearer token usage |
| RFC 7636 | **PKCE** |
| RFC 7519 / 7515 / 7517 | **JWT**, JWS, JWK |
| RFC 7662 / 7009 | Token introspection; token revocation |
| RFC 8414 | Authorization server metadata (discovery) |
| RFC 8628 | Device authorization grant |
| RFC 8693 | Token exchange |
| RFC 8705 | Mutual-TLS client authentication and certificate-bound tokens |
| RFC 8707 | Resource indicators |
| RFC 9068 | JWT profile for access tokens |
| RFC 9101 / 9126 / 9396 | JAR (signed requests), **PAR** (pushed authorization requests), **RAR** (rich authorization requests) |
| RFC 9207 | Issuer identification in authorization responses (mix-up defense) |
| RFC 9449 | **DPoP** |
| RFC 9470 | Step-up authentication challenge |
| RFC 9700 (Jan 2025) | **OAuth 2.0 Security Best Current Practice** |
| RFC 9728 | **Protected resource metadata** (used by MCP) |
| draft-ietf-oauth-v2-1 (-15, Mar 2026) | **OAuth 2.1** consolidation |
| OpenID Connect Core 1.0, Discovery, Session/Logout | Authentication layer |
| FAPI 2.0 (final Feb 2025) | High-security profile |

**The interview-grade sentence:** *"OAuth exists to solve delegated authorization without the password anti-pattern: instead of handing an app your password — full, indefinite, unrevocable access with no MFA — you authenticate at the authorization server and the client gets a scoped, short-lived, per-app revocable access token. It's an authorization framework, not authentication — that's what OpenID Connect adds — and its security comes from the profile you adopt, today RFC 9700 and OAuth 2.1, and from validating tokens correctly."*

---

## Concept 18 — The roles and the two channels

| Role | Definition | Examples |
|---|---|---|
| **Resource owner** | The entity that can grant access to a protected resource — usually the end user | You, approving an app to read your calendar |
| **Client** | The application requesting access on the resource owner's behalf (or its own) | A web app, SPA, mobile app, CLI, daemon, AI agent |
| **Authorization server (AS)** | Authenticates the resource owner, obtains consent, and issues tokens | Microsoft Entra ID, Entra External ID, Auth0, Okta, Keycloak, Duende IdentityServer, OpenIddict |
| **Resource server (RS)** | Hosts the protected resource and accepts access tokens | Your ASP.NET Core API; Microsoft Graph; an MCP server |

In OpenID Connect the AS is called the **OpenID Provider (OP)** and the client the **Relying Party (RP)**.

OAuth messages travel over two channels with very different security properties:

| | **Front channel** | **Back channel** |
|---|---|---|
| Path | Through the user's browser: redirects with query strings or fragments, form posts | Direct HTTPS between client server and AS (or RS) |
| Who can see/modify it | The user, browser extensions, browser history, proxies' logs, `Referer` headers, malicious scripts on the page, an attacker who controls a redirect | Only the two endpoints (with TLS) |
| What should travel here | Short-lived, single-use, low-value artifacts: the **authorization code**, `state`, `iss` | **Tokens**, client credentials, refresh tokens |

The whole design of the modern authorization code flow follows from this table: **only a one-time code goes through the front channel; tokens are fetched over the back channel**, and PKCE (Concept 21) ensures a stolen code is useless. The implicit grant, which returned tokens in the URL fragment through the front channel, is gone for exactly this reason (Concept 24).

Endpoints a client meets:

| Endpoint | Channel | Purpose |
|---|---|---|
| `/authorize` | Front | Start the user interaction; returns a code to the redirect URI |
| `/token` | Back | Exchange code, refresh token, client credentials or assertion for tokens |
| `/.well-known/openid-configuration` or `/.well-known/oauth-authorization-server` | Back | Discovery metadata (endpoints, supported algorithms, JWKS URI) |
| `jwks_uri` | Back | Public keys for validating signatures |
| `/par` | Back | Pushed authorization requests (Concept 31) |
| `/introspect`, `/revoke` | Back | Token introspection and revocation |
| `/devicecode` | Back | Device authorization (Concept 24) |
| `/userinfo` | Back | OIDC user claims |
| `end_session_endpoint` | Front | Logout (Concept 28) |

**The interview-grade sentence:** *"OAuth has four roles — resource owner, client, authorization server and resource server — and two channels: the front channel through the browser, which users, extensions, history and referrers can observe, and the back channel directly between servers. Modern flows put only a short-lived, single-use authorization code in the front channel and fetch tokens over the back channel, which is exactly why the implicit grant that returned tokens in the URL fragment was removed."*

---

## Concept 19 — Tokens: access, refresh and ID

Three token types with **different audiences and jobs** — mixing them up is a common and serious mistake.

| | **Access token** | **Refresh token** | **ID token** (OIDC) |
|---|---|---|---|
| Purpose | Authorize calls to an API | Obtain new access tokens without user interaction | Tell the **client** who signed in and how |
| Audience | The **resource server** (`aud` = the API) | The **authorization server** only | The **client** (`aud` = client ID) |
| Format | Opaque or JWT (RFC 9068); **the client must treat it as opaque** | Opaque (usually) | Always a JWT |
| Lifetime | Short: minutes to ~1 hour (Entra: 60–90 min by default) | Long: hours to months; may slide | Short; used once at sign-in |
| Sent to | The API, in `Authorization: Bearer …` | The token endpoint only | Nowhere — the client validates and consumes it |
| If stolen | Attacker calls the API until expiry | Attacker mints access tokens until revoked — **high value** | Limited (shouldn't be accepted anywhere) |

Rules that follow:

1. **Never send an ID token to an API as an access token**, and never accept one in an API. Its audience is the client.
2. **Clients must not parse access tokens** for decisions. The token belongs to the API; its format can change (Entra explicitly warns that access tokens for Microsoft APIs may be encrypted or change format). If the client needs user info, it uses the ID token or the UserInfo endpoint.
3. **APIs validate access tokens** — signature, issuer, audience, lifetime, scope/roles (Concept 20) — or introspect opaque ones.
4. **Refresh tokens are credentials.** Store them like passwords: server-side, encrypted, never in browser storage accessible to JavaScript (Concept 30).

**Bearer semantics.** A plain access token is a **bearer token** (RFC 6750): *possession is authorization*. No proof of who presents it. That's the root of most OAuth risk: tokens leak through logs, URLs, browser storage, proxies and memory dumps, and a leaked bearer token is fully usable. The mitigations are layered: short lifetimes, narrow audiences and scopes, keeping tokens out of logs and URLs (never in query strings), TLS everywhere, and **sender-constraining** high-value tokens with DPoP or mTLS so a stolen token can't be replayed by another party (Concept 31).

**Reference (opaque) vs self-contained (JWT) access tokens:**

| | Opaque + introspection | JWT |
|---|---|---|
| Validation | API calls the AS's introspection endpoint (or a cache of it) | API validates the signature locally with cached JWKS |
| Revocation | Immediate (AS says inactive) | Not until expiry (unless the API checks a revocation list or uses CAE) |
| Latency and AS load | A network call per token (cacheable) | None |
| Privacy | Claims stay at the AS | Claims readable by anyone holding the token (unless encrypted, JWE) |
| Typical use | High-security, revocation-sensitive APIs; gateways | Most APIs, including Entra-protected ones |

**The interview-grade sentence:** *"There are three tokens with three audiences: access tokens are for the API, short-lived and opaque to the client; refresh tokens are for the authorization server only, long-lived and as sensitive as passwords; and ID tokens are for the client, to learn who signed in, and must never be sent to or accepted by an API. Plain access tokens are bearer tokens — possession is authorization — so I keep them short-lived and narrowly scoped, out of URLs, logs and JavaScript-readable storage, and sender-constrain the high-value ones; and I choose JWTs for local validation or opaque tokens with introspection when immediate revocation matters."*

---

## Concept 20 — JWTs and how to validate them

A **JSON Web Token** (RFC 7519) is three base64url-encoded parts joined by dots: `header.payload.signature`. As a **JWS** (RFC 7515) it's *signed*, not encrypted — anyone can read the payload. A **JWE** (RFC 7516) is encrypted.

```json
// Header
{ "alg": "RS256", "typ": "at+jwt", "kid": "Lq2...h8" }

// Payload (access token for a custom API, Entra v2-style claims)
{
  "iss": "https://login.microsoftonline.com/5e1c.../v2.0",   // who issued it
  "aud": "api://orders-api" ,                                 // who it's for (Entra v2: often the API's client ID GUID)
  "sub": "AAAAAAAAAAAAAAAAAAAAAIkzqF...",                     // subject (pairwise per app in Entra)
  "oid": "6a6c...-...",                                       // Entra object ID of the user — stable across apps
  "tid": "5e1c...-...",                                       // tenant ID
  "azp": "0c5f...-...",                                       // authorized party: the client app that requested it
  "scp": "Orders.Read Orders.Write",                          // delegated scopes (space-separated)
  "roles": ["Orders.Admin"],                                  // app roles
  "iat": 1791273600, "nbf": 1791273600, "exp": 1791278100,    // issued at, not before, expires
  "jti": "1b9d...",                                           // unique token ID
  "ver": "2.0"
}
```

**Validation is a checklist, every time, in this order of importance:**

1. **Signature** — verify with a key from the issuer's **JWKS** (selected by `kid`), fetched from the discovery document and cached with periodic refresh to handle key rotation.
2. **Algorithm allow-list** — accept only the algorithms you expect (e.g., RS256, ES256, PS256). **Never** accept `alg: none`; never let the token's header choose an HMAC algorithm when you expect RSA (the classic "algorithm confusion" attack, where the public key is used as an HMAC secret).
3. **Issuer (`iss`)** — exactly matches the expected issuer(s). For multi-tenant apps, the issuer contains the tenant ID and must be validated against `tid` (Concept 36).
4. **Audience (`aud`)** — the token is for *this* API. Skipping audience validation means a token issued for any API from the same issuer works against yours — the **confused deputy / token substitution** problem.
5. **Lifetime** — `exp` in the future, `nbf` in the past, with small clock skew (ASP.NET Core defaults to 5 minutes; many teams reduce it).
6. **Token type** — where the issuer sets `typ: at+jwt` (RFC 9068), check it to avoid accepting ID tokens as access tokens.
7. **Authorization claims** — `scp` for delegated calls, `roles` for app-only calls (and app roles assigned to users), required per endpoint (Concepts 26, 56).
8. **Optional business checks** — tenant allow-list, `azp`/`appid` allow-list (which clients may call), `acr`/`amr` for step-up requirements.

Common implementation traps:

- **Decoding instead of validating.** `new JwtSecurityTokenHandler().ReadJwtToken(token)` *reads* a token; it doesn't validate anything. Code that reads claims from an unvalidated token is an authentication bypass.
- **Disabling validation "temporarily"** (`ValidateAudience = false`, `ValidateIssuer = false`, `RequireSignedTokens = false`) in development and shipping it.
- **Static key configuration** that breaks (or gets "fixed" by disabling validation) when the issuer rotates keys. Always use metadata discovery.
- **Trusting `kid` or `jku`/`x5u` headers to point at arbitrary key URLs.** Keys come only from the configured issuer's JWKS.
- **Putting sensitive data in JWT payloads** — they're readable.

In .NET, `Microsoft.IdentityModel.JsonWebTokens.JsonWebTokenHandler` (the default in ASP.NET Core's JwtBearer since .NET 8, faster than the older `JwtSecurityTokenHandler`) does all of this when configured correctly (Concept 55).

**The interview-grade sentence:** *"A JWT is a signed, readable header, payload and signature, and validating one is a fixed checklist: verify the signature with the issuer's JWKS selected by kid and refreshed for rotation, enforce an algorithm allow-list so alg-none and RSA-to-HMAC confusion fail, check issuer exactly — tenant-aware for multi-tenant — and audience so tokens for other APIs are rejected, check expiry and not-before with small skew and the at+jwt type where present, then the scopes or roles the endpoint requires. Reading claims from a decoded but unvalidated token, or disabling audience validation, is an authentication bypass."*

---

## Concept 21 — The authorization code flow with PKCE, step by step

This is the one interactive flow for **every** client type in OAuth 2.1 — web apps, SPAs (via a BFF ideally), mobile and desktop apps.

**PKCE** (Proof Key for Code Exchange, RFC 7636, pronounced "pixie") was invented for mobile apps whose redirect URIs could be intercepted by other apps; RFC 9700 and OAuth 2.1 now require it for all clients because it also defeats **authorization code injection**.

The PKCE math:

```
code_verifier  = high-entropy random string, 43–128 chars from [A-Z a-z 0-9 - . _ ~]
                 (e.g., 32 random bytes → base64url → 43 chars)
code_challenge = BASE64URL( SHA-256( ASCII(code_verifier) ) )
method         = S256      (never "plain" unless the client truly can't hash)
```

The flow:

```
 User/Browser             Client (app)                 Authorization Server            Resource Server
     │                        │                                │                              │
 (1) │  click "Sign in" ─────►│ generate code_verifier,        │                              │
     │                        │ code_challenge, state, nonce   │                              │
 (2) │◄─ 302 to /authorize?response_type=code&client_id=…&redirect_uri=…&scope=openid orders.read
     │        &state=…&nonce=…&code_challenge=…&code_challenge_method=S256 ───────────────────►│
 (3) │◄──────────────── authenticate user (password+MFA / passkey), consent ──────────────────►│
 (4) │◄─ 302 to redirect_uri?code=SplxlOBeZQQYbYS6WxSbIA&state=…&iss=… ───────────────────────│
 (5) │── GET redirect_uri?code=…&state=… ─►│ check state; check iss                          │
 (6) │                        │── POST /token  grant_type=authorization_code&code=…           │
     │                        │     &redirect_uri=…&code_verifier=…  (+ client authentication)│
     │                        │                        ──────────────►│ verify code, redirect_uri,
     │                        │                                       │ SHA256(verifier)==challenge,
     │                        │                                       │ client authentication
 (7) │                        │◄── { access_token, id_token, refresh_token, expires_in } ─────│
 (8) │                        │ validate ID token (sig, iss, aud, exp, nonce)                 │
 (9) │                        │── GET /orders   Authorization: Bearer <access_token> ───────────────────────────────►│
     │                        │◄───────────────────────────────────────── 200 [...] ──────────────────────────────────│
```

What each protection defends against:

| Parameter / check | Defends against |
|---|---|
| **`state`** (random, bound to the user's session) | **CSRF on the redirect endpoint** — an attacker making your browser complete *their* login (login CSRF) or inject their code. PKCE also defeats code injection, but `state` remains good practice and is required by many libraries |
| **PKCE `code_challenge` / `code_verifier`** | **Code interception and code injection** — a stolen code is useless without the verifier, which never left the client |
| **Exact `redirect_uri` matching** at the AS | **Open redirector / code theft** via attacker-controlled redirect URIs (wildcards and prefix matching caused many real breaches) |
| **`nonce`** in the ID token | **ID token replay / injection** into a different session |
| **`iss` in the response** (RFC 9207) | **Mix-up attacks** when a client talks to several authorization servers |
| **Client authentication at `/token`** (confidential clients) | Use of a stolen code by anyone who isn't the client |
| **Single-use, short-lived code** (seconds to ~10 minutes) | Replay; RFC 9700 says ASs should revoke tokens issued from a code that's used twice |
| **Back-channel token delivery** | Token leakage through browser history, referrers and extensions |

**What libraries do for you.** In ASP.NET Core, `AddOpenIdConnect` / `AddMicrosoftIdentityWebApp` implement steps 2–8 (state, nonce, PKCE — `UsePkce` defaults to true since .NET 7 — code exchange, ID token validation) and store the result in an encrypted cookie. MSAL does it for desktop, mobile and SPA clients. You should never hand-roll this flow.

**The interview-grade sentence:** *"The authorization code flow with PKCE is the one interactive flow in OAuth 2.1: the client sends the user to /authorize with a SHA-256 code challenge, state and nonce; the AS authenticates the user and redirects back with a short-lived single-use code and its issuer; the client checks state and issuer, then exchanges the code over the back channel with the code verifier and its client authentication; and it validates the ID token including the nonce. PKCE makes a stolen or injected code useless, exact redirect URI matching stops code theft through open redirects, state stops login CSRF, nonce stops ID token replay and the iss parameter stops mix-up attacks."*

---

## Concept 22 — Client types and client authentication

OAuth distinguishes clients by whether they can keep a secret:

| | **Confidential client** | **Public client** |
|---|---|---|
| Definition | Runs where its credentials can be kept secret from users | Runs on a device the user controls; anything shipped is extractable |
| Examples | Server-side web app, BFF, API calling another API, daemon/background service | SPA (JavaScript in the browser), mobile app, desktop app, CLI |
| Authenticates at `/token`? | **Yes** | No (it has no secret worth the name) — PKCE protects the code instead |
| Can use client credentials flow? | Yes | No |
| Refresh token protection | Bound to the client's credentials | Rotation or sender-constraining required (RFC 9700) |

**Client authentication methods**, from weakest to strongest:

| Method | How it works | Notes |
|---|---|---|
| `client_secret_basic` / `client_secret_post` | Shared secret sent with the token request | Common, simplest, a shared secret with all of Concept 7's problems; Entra caps secret lifetime (max 2 years; many orgs enforce less via policy) |
| `client_secret_jwt` | Client signs a JWT assertion with HMAC using the secret | Rarely used; still a shared secret |
| **`private_key_jwt`** | Client signs a JWT assertion (`client_assertion`) with its **private key**; the AS verifies with the registered public key/certificate | Asymmetric — the AS never holds a secret; private key can live in Key Vault/HSM and sign remotely. Entra calls this "certificate credentials" |
| **`tls_client_auth` / `self_signed_tls_client_auth`** (RFC 8705) | Client authenticates with a client certificate in mutual TLS | Strong; also enables certificate-bound tokens; operationally heavier |
| **Federated assertion** (`client_assertion` = a token from a *trusted external issuer*) | The client presents a short-lived token issued by another IdP (GitHub OIDC, Kubernetes, a managed identity) that the AS has been configured to trust | **No credential stored at all** — Entra workload identity federation (Concept 38) |

A best-practice ladder for an Entra confidential client: **managed identity** (when the client *is* an Azure workload calling Azure resources — no app registration credential at all) → **federated identity credential** (managed identity as FIC, GitHub/Kubernetes federation) → **certificate in Key Vault** used for `private_key_jwt` → **client secret** only as a last resort, stored in Key Vault, short-lived and rotated.

**Dynamic and metadata-based registration.** Classic OAuth requires pre-registering every client. **Dynamic Client Registration** (RFC 7591) lets clients register themselves — useful but risky (spam registrations, phishing apps). The newer **Client ID Metadata Documents** draft, where the client ID is an HTTPS URL hosting the client's metadata, is what the MCP specification now recommends for agents connecting to servers they've never seen.

**The interview-grade sentence:** *"Confidential clients — server-side web apps, BFFs, APIs and daemons — can hold credentials and must authenticate at the token endpoint; public clients — SPAs, mobile, desktop and CLIs — can't, so PKCE protects their codes and their refresh tokens must be rotated or sender-constrained. For confidential clients I climb the ladder from client secrets to private_key_jwt with a Key Vault certificate, to mTLS, to federated assertions where nothing is stored — and on Azure, managed identities or managed-identity federated credentials remove the app credential entirely."*

---

## Concept 23 — Client credentials: machine to machine

When there's **no user** — a nightly job calling an API, a service calling another service on its own authority — the client authenticates as itself and receives a token representing *the application*:

```http
POST /oauth2/v2.0/token HTTP/1.1
Host: login.microsoftonline.com
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=0c5f...                      
&scope=api://orders-api/.default          ← Entra: ".default" = all app permissions granted to this client for that resource
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=eyJhbGciOiJSUzI1NiIsImtpZCI6...   ← private_key_jwt or a federated token
```

The resulting token has no user (`sub` and `oid` identify the service principal), carries **`roles`** (application permissions / app roles assigned to the client), and has no `scp` claim.

Design points:

1. **Authorization uses app roles.** The API defines app roles like `Orders.Read.All`; an administrator assigns (consents) them to the calling application; the API requires the role (Concept 35). A common bug is accepting *any* token from your tenant — any app in the tenant that can get a token for your API's audience could then call it. On Entra, either require app roles or set the API's service principal to **require assignment**, and validate `roles`.
2. **Application permissions are powerful.** "Read all orders" with no user context means no per-user restriction — the client can read everything the permission covers. Grant them only to workloads that truly need tenant-wide access; prefer delegated flows (Concept 29) when a user context exists.
3. **Managed identity is client credentials without the credential.** On Azure, a managed identity requests a token for a resource from the platform's local endpoint; under the hood it's an app-only token for the identity's service principal (Concept 37).
4. **Cache the token.** Requesting a new token per call adds latency and can hit AS throttling. MSAL and Azure.Identity cache app tokens in memory and refresh before expiry; make the credential/MSAL client a **singleton** (the same lesson as `HttpClient` and `CosmosClient` in earlier modules).

**The interview-grade sentence:** *"Client credentials is for machine-to-machine calls with no user: the client authenticates as itself — ideally with a certificate assertion, a federated token or as a managed identity — and gets an app-only token carrying roles, not scopes. The API must require specific app roles, or require assignment, rather than accepting any token from the tenant, application permissions should be granted sparingly because they carry no per-user restriction, and the token client is a singleton that caches and refreshes tokens."*

---

## Concept 24 — The device flow, and the grants that were removed

**Device authorization grant** (RFC 8628) — for clients with no browser or limited input: smart TVs, IoT devices, CLIs over SSH.

```
1. Device → AS:  POST /devicecode  client_id=…&scope=…
   AS → Device:  { device_code, user_code: "WDJB-MJHT", verification_uri: "https://microsoft.com/devicelogin", interval: 5, expires_in: 900 }
2. Device shows: "Go to microsoft.com/devicelogin and enter WDJB-MJHT"
3. User, on a phone or laptop, signs in (with MFA) and approves
4. Device polls POST /token grant_type=urn:ietf:params:oauth:grant-type:device_code&device_code=… until it gets tokens
```

Its risk is **device code phishing**: an attacker starts a device flow and sends the victim the code with a convincing message ("enter this code to view the document"); the victim signs in and the attacker's device receives the tokens. This has been used in real campaigns against Microsoft 365 users. Mitigations: Conditional Access **blocking device code flow** except for users and apps that need it (Entra supports this as an authentication-flows condition), user training, and preferring flows with browser redirects where possible. In your own clients, use device flow only when there's genuinely no browser.

**The removed grants — and why:**

| Grant | How it worked | Why it's gone |
|---|---|---|
| **Implicit** (`response_type=token`) | Access token returned directly in the redirect URL fragment, for SPAs before CORS was universal | Tokens in the front channel leak through history, referrers, extensions and injected scripts; no refresh token, so apps used hidden iframes; no client authentication or PKCE-like binding; access token injection attacks. Replaced by **code + PKCE**. RFC 9700 says clients should not use it; OAuth 2.1 omits it |
| **Resource owner password credentials (ROPC)** | Client collects the user's username and password and sends them to `/token` | It *is* the password anti-pattern OAuth was invented to remove; incompatible with MFA, passkeys, federation and Conditional Access; trains users to type passwords into apps. RFC 9700: must not be used; OAuth 2.1 omits it; Microsoft recommends against it and mandatory MFA breaks it in practice |

When someone says "but ROPC is easier for our test automation": use a test tenant with a confidential client and client credentials for service-level tests, or federated test identities; for UI tests, automate the real browser flow with test accounts exempted via Conditional Access *in a test tenant only*.

**The interview-grade sentence:** *"For browserless devices and CLIs I'd use the device authorization grant — show a user code, let the user approve on another device, poll for tokens — while blocking device code flow by Conditional Access for everyone who doesn't need it, because device code phishing is a real attack. The implicit grant is gone because it put tokens in the front channel, and the password grant is gone because it is the very password anti-pattern OAuth exists to remove and can't do MFA; code with PKCE replaces both."*

---

## Concept 25 — Refresh tokens: long-lived and high-value

Access tokens are short-lived so that leaks expire quickly; refresh tokens exist so that users don't have to sign in every hour. That makes the refresh token the **most valuable credential in a session** — a stolen one lets an attacker mint access tokens for as long as it remains valid.

```http
POST /token
grant_type=refresh_token&refresh_token=0.AV4A…&client_id=…&scope=…   (+ client authentication for confidential clients)
→ { access_token, refresh_token (possibly new), expires_in }
```

**Protections, from RFC 9700 and OAuth 2.1:**

1. **Confidential clients**: refresh tokens are bound to the client's credentials — a thief also needs the client's secret, certificate or federated identity. Store them server-side, encrypted (e.g., MSAL's token cache in a distributed cache, encrypted with Data Protection — Concept 57).
2. **Public clients**: the AS must use **refresh token rotation** or **sender-constraining**:
   - **Rotation**: every refresh returns a new refresh token and invalidates the old one. If an *old* refresh token is ever presented again, that's evidence of theft (either the attacker or the legitimate client is replaying), so the AS **revokes the whole token family**. This is **reuse detection**.
   - **Sender-constraining** with DPoP or mTLS (Concept 31): the refresh token is bound to a key only the legitimate client holds.
3. **Lifetimes**: absolute maximum lifetime plus inactivity timeout. Entra's defaults: refresh tokens for SPAs are limited to **24 hours**; for other clients up to **90 days** with sliding renewal, subject to Conditional Access sign-in frequency and revocation events (Concept 40).
4. **Revocation**: password reset, user disable, admin "revoke sessions," and explicit logout (`/revoke`, RFC 7009) should invalidate refresh tokens.
5. **Scope**: refresh tokens can only obtain tokens for scopes the user consented to; requesting broader scopes requires interaction.

**Operational traps:**

- **Concurrency and rotation**: two parallel requests in the same client both trying to refresh with the same rotating token — the second looks like reuse and kills the session. Serialize refreshes per user/session (MSAL handles this; hand-rolled clients often don't).
- **Storing refresh tokens in browser storage** — in `localStorage`, any XSS steals a long-lived credential. In a SPA, prefer the BFF pattern so the browser never holds one (Concept 30).
- **Logging token responses** — the refresh token ends up in Application Insights forever (Module 28's redaction concept applies).

**The interview-grade sentence:** *"A refresh token is the highest-value credential in a session, so confidential clients keep it server-side and encrypted, bound to their own credentials, while public clients need refresh token rotation with reuse detection — replaying an old token revokes the whole family — or DPoP or mTLS binding. I'd also rely on bounded lifetimes, like Entra's 24 hours for SPAs, revocation on password reset and sign-out, serialized refreshes so rotation doesn't kill sessions under concurrency, and never storing or logging refresh tokens where script or telemetry can read them."*

---

## Concept 26 — Scopes, roles, claims and audience

Four concepts that interact and are constantly confused:

| Concept | Answers | Lives in | Who decides | Example |
|---|---|---|---|---|
| **Audience** (`aud`) | *Which API is this token for?* | Access token | The client's request (resource/scope) and the AS | `api://orders-api` |
| **Scope** (`scp`) | *What has the user allowed this client to do on their behalf?* | Delegated access tokens | User (or admin) consent | `Orders.Read` |
| **Role** (`roles`) | *What is this principal (user or app) allowed to do in this API?* | Access tokens (and ID tokens for user roles) | Administrators assign roles | `Orders.Admin`, `Orders.Read.All` |
| **Claims** | *Facts about the subject and the authentication* | Tokens | The issuer | `oid`, `tid`, `email`, `acr`, `amr`, `groups` |

The key insight: **delegated access is the intersection of what the user may do and what the client was allowed to do.** A scope doesn't grant the *user* anything; it limits the *client*. If Alice is an orders admin but the reporting app was only granted `Orders.Read`, the reporting app can only read Alice's orders even though Alice herself could delete them. The API must therefore check **both** the scope (client permission) **and** the user's own authorization (role, ownership, tenant).

**Scope design:**

- **Coarse enough to understand at consent, fine enough to limit damage**: `Orders.Read`, `Orders.Write`, `Payments.Refund` — not one `api.full_access`, and not 200 micro-scopes.
- **Resource-oriented** names (`<Resource>.<Action>[.<Constraint>]`), consistent across APIs.
- **Sensitive actions get their own scope**, so a compromised low-privilege client can't perform them.
- **Scopes aren't business authorization.** "Can approve invoices over €10,000" is a role or policy evaluated with data, not a scope.

**Audience design:**

- **One audience per API** (per independently deployed resource server). A token for the orders API must be rejected by the payments API (Concept 20). Shared "platform" audiences across many services turn one stolen token into access to everything.
- **Resource indicators** (RFC 8707) let a client request a token for a specific resource explicitly; Entra derives the audience from the scope's resource prefix (`api://orders-api/Orders.Read`).

**Groups vs app roles in Entra**: group claims are tenant-specific, can overflow (beyond a limit, Entra emits an overage indicator instead of the list), and couple your authorization to directory structure. **App roles** are defined by your app, appear in `roles`, and map naturally to policies — prefer them, assigning groups *to* roles where convenient.

**The interview-grade sentence:** *"Audience says which API a token is for, scopes say what the user let this client do, roles say what the principal may do in this API, and claims are facts about the subject and how they authenticated. Delegated access is the intersection of the client's scopes and the user's own rights, so the API checks both — and I design one audience per API so tokens can't be replayed across services, resource-oriented scopes with separate ones for sensitive actions, and app roles rather than raw group claims for authorization."*

---

## Concept 27 — OpenID Connect: authentication on top of OAuth

**OpenID Connect (OIDC, 2014)** adds an identity layer to OAuth 2.0. The client requests the `openid` scope (plus `profile`, `email`, `offline_access` as needed) in the authorization code flow and receives an **ID token**: a signed JWT, addressed to the client, describing the authentication event.

ID token claims that matter:

| Claim | Meaning | Validate / use |
|---|---|---|
| `iss` | Issuer | Must equal the expected OP (tenant-aware) |
| `aud` | The client ID | Must equal *your* client ID |
| `sub` | Subject identifier, unique per issuer (and in Entra, per app — pairwise) | Stable user key for this app |
| `exp`, `iat` | Lifetime | Standard checks |
| `nonce` | Echo of the value the client sent | Must match the value bound to this browser session |
| `auth_time` | When the user actually authenticated | For "re-authenticate if older than X" |
| `acr`, `amr` | Authentication context class and methods (e.g., `mfa`, `pwd`, `fido`) | Step-up decisions |
| `at_hash`, `c_hash` | Hashes binding the ID token to the access token / code | Validated by libraries in hybrid flows |
| `oid`, `tid` (Entra) | Object and tenant ID | **Use `oid` + `tid` as the stable user key across apps in Entra**; `email` and `preferred_username` are mutable and must never be the primary key |

**Discovery and keys.** Every OP publishes a discovery document at `/.well-known/openid-configuration` listing its endpoints, supported flows and algorithms, and its `jwks_uri`. Clients and APIs read it at startup and refresh periodically; signing keys rotate (Entra rotates its keys periodically and can roll them in emergencies), so **hard-coding keys or certificates is an outage waiting to happen**. ASP.NET Core's `ConfigurationManager` handles this automatically.

**UserInfo endpoint** returns claims for an access token (with the `openid` scope); useful when you don't want large ID tokens.

**Identity vs account linking.** Treat `(iss, sub)` — or in Entra `(tid, oid)` — as the identity key. Linking accounts by email across providers is a well-known account-takeover vector (an attacker registers the victim's email at a provider that doesn't verify it). If you must link, require verified email claims from providers you trust and re-authentication.

**SAML** still exists in enterprise SSO (XML assertions, POST bindings). For new applications, OIDC is the default; for enterprise customers who insist on SAML, an IdP like Entra can broker it.

**The interview-grade sentence:** *"OpenID Connect layers authentication onto OAuth: requesting the openid scope in the code flow returns an ID token addressed to the client, which I validate for issuer, audience equal to my client ID, lifetime and the nonce bound to the browser session, and I use acr, amr and auth_time for step-up. Keys come from the discovery document's JWKS and rotate, so they're never hard-coded; and identities are keyed on issuer plus subject — in Entra, tenant ID plus object ID — never on email, because email-based account linking is a classic account-takeover vector."*

---

## Concept 28 — Sessions and logout

After an OIDC sign-in, a web app typically **doesn't use the ID token as its session**. It creates its own **session cookie** (in ASP.NET Core, an encrypted authentication cookie via Data Protection — Concept 59) containing the user's claims and, if it calls APIs, a reference to tokens stored server-side. There are now *several* sessions:

```
 Browser ──cookie──► Your app session (your cookie)       lifetime you choose (e.g., 8 h sliding)
 Browser ──cookie──► IdP session (Entra/External ID)      lifetime set by the IdP / Conditional Access sign-in frequency
                     Refresh token (server-side)          IdP's refresh lifetime
                     Access tokens (server-side cache)    ~1 h each
```

**Session design decisions:**

1. **Lifetime and sliding expiration**: shorter for sensitive apps (banking: 15 minutes idle), longer for low-risk ones. Absolute maximum plus idle timeout.
2. **Cookie flags**: `HttpOnly` (no JavaScript access), `Secure`, `SameSite=Lax` or `Strict`, and the **`__Host-` prefix** (forces `Secure`, path `/`, no `Domain` — prevents subdomain cookie injection).
3. **Cookie size**: claims-heavy cookies exceed browser limits (ASP.NET Core chunks them); keep claims minimal or use a server-side session store (`ITicketStore`).
4. **Re-authentication for sensitive actions** — check `auth_time`/`acr` and trigger step-up (RFC 9470 defines a standard challenge for APIs: `WWW-Authenticate: Bearer error="insufficient_user_authentication", acr_values="…"`).

**Logout is genuinely hard** because there are multiple sessions in multiple places:

| Mechanism | How | Limits |
|---|---|---|
| **Local logout** | Delete your app's cookie | IdP session remains; next sign-in is silent (SSO) |
| **RP-initiated logout** | Redirect to the OP's `end_session_endpoint` with `id_token_hint` and `post_logout_redirect_uri` | Ends the IdP session; other apps' sessions may remain |
| **Front-channel logout** | OP loads each RP's logout URL in hidden iframes | Broken by third-party cookie restrictions in modern browsers |
| **Back-channel logout** | OP sends a signed **logout token** server-to-server to each RP's endpoint; RP kills the matching session (by `sid`) | Reliable, but RPs must index sessions by `sid` and be reachable |
| **Token revocation** | Revoke refresh tokens (RFC 7009) | Outstanding access tokens remain valid until expiry (or CAE) |

A sound design for a high-value app: local cookie deletion + RP-initiated logout + back-channel logout support + server-side session store so sessions can be killed centrally + refresh token revocation + short access-token lifetimes (or CAE) for the residual window.

**The interview-grade sentence:** *"After OIDC sign-in the app runs its own session — an HttpOnly, Secure, SameSite, __Host- prefixed cookie with idle and absolute timeouts and minimal claims — separate from the IdP's session and the server-side tokens, and sensitive actions trigger step-up based on auth_time and acr. Logout must end all of them: delete the cookie, redirect to the provider's end-session endpoint, support back-channel logout keyed by session ID because front-channel iframes break under third-party cookie restrictions, revoke refresh tokens, and accept that outstanding access tokens live until expiry unless we use continuous access evaluation."*

---

## Concept 29 — Delegation chains: on-behalf-of and token exchange

In a distributed system a user's request often crosses several services: browser → BFF → Orders API → Inventory API. Each hop needs a token **for its downstream audience**, and ideally the downstream service still knows **which user** the request is for. Options:

| Pattern | How | Pros | Cons |
|---|---|---|---|
| **Forward the incoming token** | Orders API sends the user's token (aud = orders-api) to Inventory | Simple | **Wrong audience** — Inventory must then accept tokens not issued for it, which defeats audience validation; any service holding a token can call everything |
| **App-only token** (client credentials) | Orders API calls Inventory as itself | Simple; clear service identity | User context lost — Inventory can't authorize or audit per user; requires broad app permissions |
| **On-Behalf-Of (OBO)** | Orders API exchanges the incoming user token for a new token for Inventory, still representing the user | Correct audience per hop; user identity preserved; scopes limited per hop | A token request per hop (cached); middle tier must be a confidential client; Entra-specific grant (`urn:ietf:params:oauth:grant-type:jwt-bearer` with `requested_token_use=on_behalf_of`) |
| **Token Exchange (RFC 8693)** | Standard version of the same idea, with `subject_token` and optional `actor_token`; the result can carry an **`act` claim** showing the delegation chain ("Orders API acting for Alice") | Standard, expresses delegation explicitly | Support varies by IdP |
| **Internal signed context** | Edge validates the user token, then services pass a short-lived internally signed token or mTLS-authenticated header | Fast; works with service mesh | Custom; every service must trust the internal issuer; risky if headers can be injected from outside |

**OBO in Entra** (Concept 57 shows the .NET code):

```
Browser/BFF ──token(aud=orders-api, scp=Orders.Read, sub=Alice)──► Orders API
Orders API  ──POST /token grant_type=jwt-bearer, assertion=<incoming token>, scope=api://inventory-api/Inventory.Read,
              requested_token_use=on_behalf_of (+ Orders API's client credentials)──► Entra
Entra       ──token(aud=inventory-api, scp=Inventory.Read, sub=Alice, azp=orders-api)──► Orders API ──► Inventory API
```

Design rules:

1. **Every hop validates a token issued for itself.** No service should accept tokens with someone else's audience.
2. **Preserve the user where the downstream authorizes or audits per user**; use app-only tokens for genuinely service-level calls (cache refresh, background jobs).
3. **Downscope at every hop** — request only the scopes the next service needs.
4. **Keep chains short.** Each OBO hop adds a token request (cached per user and audience, but cold-path latency and AS dependency matter — Module 13's dependency availability arithmetic applies to the IdP too).
5. **Asynchronous work loses the user token** (it expires). For background processing triggered by a user, carry the *user's identity as data* (an `initiatedBy` field), have the worker act with its own app identity, and authorize at enqueue time — don't stash user tokens in messages.

**The interview-grade sentence:** *"Across service hops I never forward the user's token to a service it wasn't issued for, because that defeats audience validation; instead each middle tier uses on-behalf-of — or RFC 8693 token exchange where supported — to swap the incoming token for a downscoped token for the next audience that still represents the user, while genuinely service-level calls use app-only tokens. Every hop validates a token issued for itself, chains stay short because each hop depends on the IdP, and asynchronous work carries the user's identity as data and runs under the worker's own identity rather than storing user tokens in messages."*

---

## Concept 30 — Browser-based apps and the Backend-for-Frontend

A SPA is a **public client running in a hostile environment**: any XSS, malicious browser extension or compromised third-party script runs with the page's privileges. Where it keeps tokens decides what an attacker gets.

| Token storage in the browser | Readable by injected script? | Notes |
|---|---|---|
| `localStorage` / `sessionStorage` | **Yes** | XSS exfiltrates tokens, including refresh tokens, to be used from the attacker's machine for hours or days |
| In-memory JavaScript variable | **Yes** (script in the same context can reach it or hook `fetch`) | Better than storage (no persistence) but not a defense against XSS |
| Web Worker / service worker isolation | Harder | Reduces exposure; complex |
| **HttpOnly cookie to your own backend** | **No** | Script can *use* the session (by making requests) while XSS is active, but can't **steal** a reusable credential |

The IETF's **"OAuth 2.0 for Browser-Based Applications"** best-practice draft describes three architectures, in decreasing order of security:

1. **Backend-for-Frontend (BFF)** — a server-side component (in .NET: an ASP.NET Core host serving the SPA, or a YARP-based proxy) is a **confidential client**: it runs the authorization code flow, keeps access and refresh tokens server-side, and gives the browser only an HttpOnly session cookie. The SPA calls the BFF (`/api/...`); the BFF attaches the access token and proxies to APIs.
2. **Token-mediating backend** — the backend obtains tokens and hands *access tokens* (but not refresh tokens) to the browser.
3. **Browser-based OAuth client** — the SPA itself runs code + PKCE (e.g., MSAL.js) with refresh token rotation; tokens live in the browser. Acceptable for lower-risk apps; requires strong XSS defenses.

```
        BFF pattern
 ┌──────────┐ cookie (HttpOnly, Secure, SameSite, __Host-) ┌───────────────────┐ Bearer token ┌───────────┐
 │  SPA      │ ─────────────── /api/orders ───────────────►│  BFF (ASP.NET Core)│─────────────►│ Orders API │
 │ (no tokens│ ◄────────────── JSON ───────────────────────│  confidential client│◄─────────────│            │
 └──────────┘                                              │  token cache (server)│             └───────────┘
                                                           └─────────┬──────────┘
                                                                     │ OIDC code + PKCE, refresh
                                                                     ▼
                                                                 Entra ID
```

**CSRF returns** when you use cookies: a malicious site can cause the browser to send your cookie. Defenses in a BFF: `SameSite=Lax/Strict` cookies, requiring a **custom header** on API calls (e.g., `X-CSRF: 1` — cross-site requests can't set custom headers without a CORS preflight, which your BFF won't approve), antiforgery tokens for form posts, and strict CORS (Concept 60).

**XSS is still the enemy.** The BFF limits XSS to "act as the user while the page is open" instead of "steal long-lived credentials," but XSS must still be prevented: framework output encoding (Razor, Blazor, React escape by default), a strict **Content Security Policy**, Subresource Integrity for third-party scripts, and no `innerHTML` with untrusted data.

**The interview-grade sentence:** *"Tokens in localStorage, or anywhere JavaScript can read them, turn any XSS into theft of reusable — often long-lived — credentials, so for anything sensitive I use the Backend-for-Frontend pattern: an ASP.NET Core BFF is the confidential OIDC client, keeps access and refresh tokens server-side and gives the SPA only an HttpOnly, Secure, SameSite, __Host- session cookie, proxying API calls with the token attached. That brings CSRF back, which I handle with SameSite plus a required custom header and antiforgery for forms, and XSS still has to be prevented with output encoding and a strict CSP, because the BFF only limits what XSS can do."*

---

## Concept 31 — Sender-constrained tokens and high-security profiles

For high-value APIs (payments, health, admin), bearer semantics aren't enough. The tools:

**DPoP — Demonstrating Proof of Possession (RFC 9449, 2023).** The client generates a key pair (in the browser via WebCrypto with a non-extractable key, or in an app's secure storage). Every token request and every API call carries a **`DPoP` header**: a short JWT signed with the private key, containing `htm` (HTTP method), `htu` (URL), `iat`, a unique `jti`, and for API calls `ath` (hash of the access token). The AS binds the issued token to the key's thumbprint (`cnf.jkt` claim). The API verifies that the proof is signed by the bound key, fresh, for this method and URL, and not replayed. **A stolen token is useless without the private key.** Works at the application layer, so it suits SPAs and mobile apps where mTLS is impractical.

**mTLS certificate-bound tokens (RFC 8705).** The client authenticates to the AS with a client certificate; the token carries `cnf.x5t#S256` (the certificate thumbprint); the API checks that the TLS client certificate presented on the call matches. Strong and simple for server-to-server; awkward through TLS-terminating proxies (the proxy must forward the client certificate securely) and in browsers.

**Microsoft's position.** Entra supports proof-of-possession in specific Microsoft scenarios (for example token protection for sign-in sessions on supported platforms, and mTLS-bound tokens for some confidential-client and managed-identity scenarios, which Microsoft.Identity.Web 4.14 surfaces for federated credentials) rather than general-purpose DPoP for your own APIs. Verify current support before designing on it; if you need DPoP end to end, an AS like Duende IdentityServer, Keycloak or Auth0 supports it.

**PAR — Pushed Authorization Requests (RFC 9126).** Instead of putting all authorization parameters in the front-channel URL, the client POSTs them to the AS's `/par` endpoint (authenticated) and receives a `request_uri` to send in the redirect. Benefits: parameters can't be tampered with or observed in the browser; the AS authenticates the client *before* the user interaction; long requests (RAR) fit.

**RAR — Rich Authorization Requests (RFC 9396).** Replaces coarse scope strings with structured `authorization_details` — *"payment of €45.00 to IBAN X, once"* — so consent and tokens describe exactly the transaction. Essential for open banking and agent-initiated actions.

**JAR — JWT-Secured Authorization Requests (RFC 9101).** Authorization parameters as a signed (and optionally encrypted) JWT, giving integrity and non-repudiation of the request.

**FAPI 2.0 Security Profile (OpenID Foundation, final February 2025)** packages the strong options into one interoperable profile used by open banking regimes: authorization code with PKCE (S256), **PAR required**, confidential clients with `private_key_jwt` or mTLS, **sender-constrained access tokens (DPoP or mTLS)**, `iss` in responses, short-lived codes, and no long-lived bearer tokens. If someone asks "how would you design OAuth for a payments API?", "FAPI 2.0" is a strong answer.

**The interview-grade sentence:** *"For high-value APIs I don't rely on bearer tokens: DPoP binds tokens to a client-held key by requiring a signed per-request proof with the method, URL and token hash, and mTLS binds them to the client certificate — either way a stolen token can't be replayed. I'd add PAR so authorization parameters go over an authenticated back channel instead of the browser, RAR so consent describes the exact transaction, and in regulated finance adopt the FAPI 2.0 Security Profile, final since February 2025, which mandates PKCE, PAR, strong client authentication and sender-constrained tokens — while checking what my identity provider actually supports, since Entra's proof-of-possession support is scenario-specific."*

---

## Concept 32 — OAuth 2.1, RFC 9700 and the attacks they exist to stop

**RFC 9700** (OAuth 2.0 Security Best Current Practice, January 2025) is the distilled lesson of a decade of OAuth attacks; **OAuth 2.1** (draft -15) folds it, PKCE and the bearer/browser guidance into one document. The headline requirements:

| Requirement | Attack it stops |
|---|---|
| **PKCE for all authorization code clients** (S256) | Authorization code interception and **code injection** (attacker inserts a stolen code into the victim's session or vice versa) |
| **Exact redirect URI matching** (no wildcards, no prefix match; localhost port flexibility only for native apps) | **Code/token theft via open redirects** and attacker-controlled redirect URIs |
| **No implicit grant; no tokens in URLs** | Token leakage via history, `Referer`, logs; **access token injection** |
| **No password grant** | Credential exposure; MFA bypass |
| **Refresh tokens for public clients rotated or sender-constrained** | Refresh token theft and replay |
| **Issuer identification (`iss` parameter, RFC 9207) or distinct redirect URIs per AS** | **Mix-up attacks** — a malicious AS tricks a multi-AS client into sending a code meant for an honest AS to the attacker |
| **Audience-restricted access tokens** (resource indicators) | **Token replay across resource servers** — a malicious or compromised RS reusing a token it received |
| **Least-privilege scopes** | Over-privileged leaked tokens |
| **Sender-constrained tokens recommended** (DPoP/mTLS) | Replay of leaked access tokens |
| **No open redirectors at client or AS** | Phishing and code exfiltration via `redirect_uri`/`post_logout_redirect_uri` |
| **CSRF protection** (PKCE, `state`, or OIDC `nonce`) | Login CSRF and session swapping |
| **Clickjacking protection on the authorization page** (`frame-ancestors`/`X-Frame-Options`) | Tricking users into approving consent in a hidden frame |

Other attack classes an architect should be able to name:

- **Consent phishing / illicit consent grants** — a malicious multi-tenant app requests broad permissions (`Mail.Read`, `Files.ReadWrite.All`); users approve; the attacker gets persistent API access without passwords. Mitigations in Entra: restrict user consent to verified publishers and low-risk permissions, admin consent workflow, app governance and periodic review of granted permissions.
- **Token theft from endpoints** — malware stealing browser cookies and tokens ("pass-the-cookie"), now common against cloud tenants. Mitigations: phishing-resistant MFA doesn't stop it alone; token protection/binding, short sessions, CAE, device compliance in Conditional Access.
- **Audience confusion in multi-tenant apps** — accepting tokens from *any* Entra tenant because the issuer check was disabled (Concept 36).
- **JWT validation flaws** — `alg:none`, algorithm confusion, missing audience checks (Concept 20).
- **SSRF via OIDC metadata** — servers fetching `jwks_uri`, `logo_uri` or `request_uri` from attacker-supplied URLs (dynamic registration and CIMD implementations must restrict fetches).

**Agents and MCP — OAuth's newest client type.** The **Model Context Protocol** authorization spec makes an MCP server an **OAuth 2.1 resource server** that **must** publish **RFC 9728 protected resource metadata** (`/.well-known/oauth-protected-resource`, advertised in the `WWW-Authenticate` header of a 401) pointing to its authorization server(s); clients use code + PKCE, send **RFC 8707 resource indicators** so tokens are audience-bound to that MCP server, and (since 2025-11-25) **should** register via **Client ID Metadata Documents**. The two classic agent mistakes map directly to earlier concepts: **token passthrough** (an MCP server forwarding the user's token to downstream APIs — the audience problem of Concept 29; the spec forbids it) and **confused deputy** (a server using its own broad credentials on behalf of whatever the agent asks — Concept 44).

**The interview-grade sentence:** *"RFC 9700, published in January 2025, and the OAuth 2.1 draft codify a decade of attacks: PKCE for every code flow against code interception and injection, exact redirect URI matching against open-redirect token theft, no implicit or password grants, rotated or sender-constrained refresh tokens for public clients, issuer identification against mix-up, audience-restricted tokens against replay across resource servers, and least-privilege scopes. Beyond the spec I'd watch for consent phishing, cookie and token theft from endpoints, disabled issuer checks in multi-tenant apps, JWT validation flaws and SSRF through metadata URLs — and with AI agents, MCP now profiles all of this with RFC 9728 resource metadata and resource indicators, and explicitly forbids token passthrough."*

---
# Part D — Microsoft Entra ID

## Concept 33 — The Entra map: tenants and the three audiences

**Microsoft Entra** is the family name (since 2023) for Microsoft's identity products; **Microsoft Entra ID** is what used to be Azure Active Directory. Its fundamental unit is the **tenant**: a dedicated directory instance with its own users, groups, applications, policies and a globally unique **tenant ID** (GUID). Every Azure subscription trusts exactly one tenant for authentication.

Three audiences, three models:

| Audience | Entra capability | Who the users are | Typical design |
|---|---|---|---|
| **Workforce** | Entra ID (workforce tenant) | Employees and contractors, often synced from on-premises AD via Entra Connect / Cloud Sync | SSO to SaaS and internal apps; Conditional Access; PIM; device management with Intune |
| **Partners / business guests** | **B2B collaboration** (external users as guests in your workforce tenant) and **B2B direct connect** / cross-tenant access settings | Users from other organizations, authenticating at *their* home tenant (or with email OTP / social for non-Entra orgs) | Invite guests to specific apps; cross-tenant trust settings decide whether you accept their MFA and device claims |
| **Customers / consumers** | **Entra External ID** (external tenant configuration — the CIAM product, GA September 2024) | Your app's customers, with local accounts (email + password/OTP/passkey) or social IdPs | Customer sign-up/sign-in user flows, branding, custom domains, native authentication SDKs (Concept 43) |

**Azure AD B2C's status** (a common interview trap): B2C has been **closed to new customers since May 1, 2025**; **B2C Premium P2 was discontinued March 15, 2026** (risk-based Identity Protection features went with it); Microsoft says it will **support existing B2C tenants until at least May 2030**. New CIAM work goes on **External ID**; existing B2C estates should plan a migration (custom XML policies don't port directly).

**Tenancy design decisions an architect owns:**

1. **One workforce tenant per organization** is the norm; multiple tenants multiply admin overhead and break SSO. Separate tenants are justified for strong isolation (a regulated subsidiary), test environments with risky configuration (a **test tenant** for experimenting with Conditional Access and app registrations), or customer identity (External ID).
2. **Multi-tenant SaaS on Entra**: your app is registered once in your tenant as **multi-tenant**; each customer organization consents, creating a **service principal** in *their* tenant (Concept 34). Their users sign in with their own corporate identities and their admins control access — the standard B2B SaaS pattern.
3. **Licensing tiers matter for design**: Entra ID Free (basic SSO and MFA via security defaults), **P1** (Conditional Access, dynamic groups, hybrid features), **P2** (Identity Protection risk-based policies, **PIM**), plus **Entra ID Governance** (access reviews, entitlement management, lifecycle workflows) and **Workload ID Premium** (Conditional Access and risk for workload identities).

**The interview-grade sentence:** *"A tenant is Entra's isolation unit — users, apps, policies, one tenant per subscription trust — and I design for three audiences: the workforce tenant with Conditional Access and PIM for employees, B2B guests and cross-tenant access settings for partners who authenticate at their home tenant, and Entra External ID for customers. Azure AD B2C has been closed to new customers since May 2025, lost its P2 tier in March 2026 and is supported for existing tenants until at least May 2030, so new customer identity goes on External ID; and for B2B SaaS I register one multi-tenant app that each customer organization consents to."*

---

## Concept 34 — App registrations and service principals

Two objects represent an application in Entra, and confusing them causes most "why can't my app get a token" tickets.

| | **Application object** (app registration) | **Service principal** (enterprise application) |
|---|---|---|
| What it is | The **global definition** of the app: identifier (client ID / `appId`), redirect URIs, credentials, exposed scopes and app roles, required permissions, token configuration | The app's **instance in a specific tenant**: the security principal that gets consented permissions, role assignments, user assignments and Conditional Access policies |
| How many | One, in the home tenant | One per tenant where the app is used (including the home tenant) |
| Where you see it | Entra admin center → App registrations | Entra admin center → Enterprise applications |
| Managed identities | Have **no** app registration you manage — only a service principal | — |

Key settings on an app registration, with the security reasoning:

- **Supported account types**: single tenant, multi-tenant (any Entra org), multi-tenant + personal Microsoft accounts. Pick the narrowest; multi-tenant apps must implement tenant-aware issuer validation and tenant onboarding (Concept 36, 63).
- **Redirect URIs**: exact, HTTPS (except `http://localhost` for development), per platform type (web, SPA, public client/native). SPA-type redirect URIs enable CORS on the token endpoint for browser code + PKCE. Remove stale ones — an abandoned redirect URI on a domain you no longer own is a code-theft vector (subdomain takeover).
- **Credentials**: prefer **none** (managed identity as FIC, federated credentials) → certificates → secrets (short, in Key Vault). Use app management policies to restrict secret lifetimes or block secrets tenant-wide.
- **Expose an API**: Application ID URI (`api://orders-api` or a verified domain URI), **scopes** for delegated access, **app roles** for application permissions and user role assignment.
- **Token configuration**: optional claims, group claims, access token version (`requestedAccessTokenVersion` = 2 recommended for new APIs).
- **Owners**: app owners can add credentials — an app owner is effectively someone who can impersonate the app. Keep ownership tight and audited.

On the **service principal** (enterprise app) side:

- **Assignment required** = only users/groups (or, for APIs, client apps with app roles) explicitly assigned can get tokens — a powerful, underused control for internal apps and APIs.
- **Permissions and consent** (Concept 35) are granted here, per tenant.
- **Conditional Access** targets enterprise apps.
- **Sign-in logs** show service principal sign-ins — the first place to look when investigating a leaked credential.

A frequently exploited weakness: **service principals with privileged Graph permissions** (`Application.ReadWrite.All`, `RoleManagement.ReadWrite.Directory`) and credentials added by an attacker — a path to tenant takeover seen in major real-world incidents. Review high-privilege app permissions and credential additions regularly; alert on new credentials on privileged apps.

**The interview-grade sentence:** *"An app registration is the global definition of an application — client ID, redirect URIs, credentials, exposed scopes and app roles — while a service principal is its instance in each tenant, where permissions are consented, roles and users are assigned and Conditional Access applies; managed identities have only the service principal. I keep account types narrow, redirect URIs exact and pruned, credentials secretless or certificate-based, owners few because owners can add credentials, turn on assignment required for internal apps and APIs, and watch privileged Graph permissions and new credentials on service principals, which is a known tenant-takeover path."*

---

## Concept 35 — Permissions in Entra: delegated vs application, scopes vs roles, consent

| | **Delegated permissions** | **Application permissions** |
|---|---|---|
| Who is acting | The app **on behalf of a signed-in user** | The app **as itself**, no user |
| Effective access | Intersection of the permission and the user's own rights | Everything the permission covers, tenant-wide |
| Flow | Authorization code (+ OBO downstream) | Client credentials / managed identity |
| Token claim | **`scp`** (space-separated scopes) | **`roles`** (app roles assigned to the app) |
| Consent | User consent (if allowed by tenant policy) or admin consent | **Always admin consent** |
| Defined in the API as | "Scopes" under *Expose an API* | "App roles" with *allowed member types* = Applications |

**App roles** can also be assigned to **users and groups** (allowed member types = Users/Groups); then they appear in the user's tokens in `roles`. That's the cleanest way to do role-based authorization for your app's users (Concept 26).

**Consent:**

- **User consent** — a user approves delegated permissions for themselves. Tenant policy controls whether users may consent at all; Microsoft's recommended setting is to allow it only for **verified publishers** and **low-impact permissions**, with an **admin consent workflow** for requests beyond that. This is the main defense against **consent phishing** (Concept 32).
- **Admin consent** — an administrator grants permissions for the whole tenant; required for application permissions and high-impact delegated permissions.
- **Incremental and dynamic consent** — v2.0 endpoint clients can request additional scopes when needed rather than everything up front.
- **`.default` scope** — `api://orders-api/.default` means "all permissions statically configured on the app registration for this resource" — required for the client credentials flow and useful for OBO.

**Designing your API's permission model:**

1. Define **delegated scopes** for user-context calls (`Orders.Read`, `Orders.Write`) and **application roles** for service-to-service calls (`Orders.Read.All`, `Orders.Process`).
2. In the API, write policies that accept *either* the right scope (delegated) *or* the right app role (app-only) per endpoint, and **distinguish** the two cases when the semantics differ: delegated calls are limited to the user's own data; app-only calls may be tenant-wide.
3. Validate that **app-only tokens are app-only** (no `scp`, `idtyp` = `app` if you enable that optional claim) — so a delegated token can't satisfy an app-only policy by accident.
4. For Microsoft Graph, request the **least privileged permission** that works (`User.Read` not `User.Read.All`; `Sites.Selected` instead of `Sites.ReadWrite.All` for SharePoint) — Graph permissions are some of the most abused in incident reports.

**The interview-grade sentence:** *"Entra separates delegated permissions — the app acting for a signed-in user, limited to the intersection with the user's rights, carried as scp — from application permissions — the app acting as itself tenant-wide, carried as roles and always requiring admin consent. My APIs expose delegated scopes for user calls and app roles for service calls, policies distinguish the two because app-only access is tenant-wide, user consent is restricted to verified publishers and low-impact permissions with an admin consent workflow against consent phishing, and Graph permissions are always the least privileged that work, like Sites.Selected instead of tenant-wide write."*

---

## Concept 36 — Entra tokens in detail: versions, issuers and validation traps

**Token versions.** Entra issues **v1.0** and **v2.0** access tokens. The version is determined by the **resource (API) app's manifest** — `requestedAccessTokenVersion` (formerly `accessTokenAcceptedVersion`) — *not* by whether the client used the v1 or v2 endpoint. Set it to **2** for new APIs.

| | v1.0 access token | v2.0 access token |
|---|---|---|
| `iss` | `https://sts.windows.net/{tid}/` | `https://login.microsoftonline.com/{tid}/v2.0` |
| `aud` | App ID URI (e.g., `api://orders-api`) or client ID | API's **client ID** (GUID) |
| Client app claim | `appid` | `azp` |
| Name claims | `upn`, `unique_name` | `preferred_username`, `name` |

Your validation must accept the audience form(s) you actually receive — Microsoft.Identity.Web accepts both the client ID and the App ID URI for you.

**Identity claims to key on:** `tid` + `oid` (object ID) — stable, unique, immutable for a user or service principal across all apps in the tenant. `sub` is **pairwise** (different per app) — fine as a per-app key, useless for cross-app correlation. `preferred_username`, `email`, `upn` are **mutable** and, for `email`, potentially **unverified** — never use them for authorization or as primary keys (a well-known class of account-takeover issues with multi-tenant apps keyed on email claims).

**Multi-tenant issuer validation** — the trap that has produced real cross-tenant breaches:

- A multi-tenant app uses the `/organizations` or `/common` authority; tokens arrive with issuers that contain **each customer's tenant ID**.
- Lazy implementations set `ValidateIssuer = false` → the API accepts tokens from **any** Entra tenant on earth — including a tenant an attacker creates in five minutes.
- Correct validation: the issuer must match the **tenant-templated pattern** *and* the `{tid}` in the issuer must equal the token's `tid` claim (Microsoft.Identity.Web's `AadIssuerValidator` does this), **and** the tenant must be one of **your onboarded customers** (an allow-list check in your code or a database lookup) — a token from a genuine but *un-onboarded* tenant is still not a customer.

**Lifetimes:** Entra access tokens default to a randomized **60–90 minutes**; with **continuous access evaluation** (Concept 40), CAE-capable clients can receive tokens valid up to **28 hours** because revocation events are pushed instead of waiting for expiry. ID tokens ~1 hour. Refresh tokens: 24 hours for SPAs, up to 90 days otherwise. Configurable token lifetime policies exist for access/ID tokens but are rarely the right lever — Conditional Access **sign-in frequency** is the supported way to force re-authentication.

**Other details worth knowing:**

- **Group overage**: if a user is in more groups than fit (the limit for JWTs is 200), Entra omits the `groups` claim and signals overage — apps must call Graph or (better) use app roles.
- **`wids`** carries directory role template IDs (Global Administrator etc.) — don't use it for your app's authorization.
- **Optional claims** (`idtyp`, `acct`, `xms_cc` for client capabilities like CAE) are opt-in via token configuration.
- **Access tokens for Microsoft first-party APIs** (Graph) are not meant to be validated by your code; they may be encrypted or change format.
- **Signing keys** come from `https://login.microsoftonline.com/{tenant}/discovery/v2.0/keys`; apps with custom signing keys (claims mapping) use an app-specific JWKS (`?appid=`).

**The interview-grade sentence:** *"Entra's access token version is set by the API's manifest, not the endpoint — v2 for new APIs, where the issuer is login.microsoftonline.com/{tid}/v2.0 and the audience is the API's client ID — and I key identities on tenant ID plus object ID, never on mutable email or UPN claims. For multi-tenant APIs I never disable issuer validation: the issuer must match the tenant template with its tid equal to the token's tid, and the tenant must be one we've actually onboarded, because otherwise any attacker-created tenant gets in. And I remember tokens last 60 to 90 minutes by default, up to 28 hours with CAE, and group claims overflow at 200, which is one more reason to use app roles."*

---

## Concept 37 — Managed identities

A **managed identity** is a service principal whose **credentials are managed entirely by Azure**: no secret or certificate exists that you can see, copy or leak. The workload asks the platform for a token; the platform authenticates the workload by *where it's running*.

| | **System-assigned** | **User-assigned** |
|---|---|---|
| Lifecycle | Created with and deleted with the resource | Standalone Azure resource; lives independently |
| Sharing | One resource only | Can be attached to many resources |
| Pre-provisioning | Exists only after the resource is created | Create it and grant roles **before** deploying the workload — no "deploy, then wait for role assignment" race |
| Blue/green, scale-out, re-creation | New identity (new principal ID) each time the resource is re-created → role assignments must be recreated | Stable identity across re-deployments |
| Typical use | Simple single-resource apps | Most production workloads; AKS workload identity; identities used as federated credentials |

**How a token is obtained.** On a VM, the workload calls the **Instance Metadata Service** (IMDS) at `http://169.254.169.254/metadata/identity/oauth2/token?resource=...` — a link-local address reachable only from inside the VM. On App Service, Functions and Container Apps, the platform injects `IDENTITY_ENDPOINT` and `IDENTITY_HEADER` environment variables for a local token service. The Azure SDKs (`ManagedIdentityCredential`) and MSAL handle all of this; your code never sees a credential.

**Design guidance:**

1. **One identity per workload per environment** (Concept 4). Sharing a user-assigned identity across unrelated services merges their blast radius. Sharing across replicas of the *same* service is correct.
2. **Prefer user-assigned for production**: stable principal IDs, pre-authorized before deployment, cleaner infrastructure code. When a resource has several user-assigned identities, specify which one by client ID (`ManagedIdentityId.FromUserAssignedClientId(...)` in Azure.Identity).
3. **Grant data-plane roles at the narrowest scope** — `Key Vault Secrets User` on one vault, `Storage Blob Data Contributor` on one container, `Azure Service Bus Data Sender` on one queue (Concept 42).
4. **Expect propagation delays.** New role assignments can take minutes to apply; and because the platform **caches managed identity tokens** (Microsoft documents caching of up to 24 hours), **removing** a role or group membership may not take effect until the cached token expires. Design revocation-sensitive access with that in mind (and prefer direct role assignments over group membership for workloads).
5. **SSRF is the main threat.** Anything that can make your workload issue arbitrary HTTP requests can try to reach IMDS and steal a token. IMDS requires a `Metadata: true` header and rejects requests with `X-Forwarded-For`, which blocks naive SSRF, but code that lets users control URLs *and* headers must be locked down (Concept 60).
6. **Managed identities can't leave Azure.** For workloads elsewhere (GitHub runners, on-premises, other clouds, Kubernetes outside AKS), use workload identity federation (Concept 38) or Azure Arc.

```csharp
// Production: deterministic — this exact user-assigned identity, nothing else.
TokenCredential credential = new ManagedIdentityCredential(
    ManagedIdentityId.FromUserAssignedClientId(builder.Configuration["Identity:ClientId"]!));

var blobs = new BlobServiceClient(new Uri("https://contosoorders.blob.core.windows.net"), credential);
```

**The interview-grade sentence:** *"A managed identity is a service principal whose credentials Azure manages, so there's nothing to leak: the workload gets tokens from IMDS or the platform's local identity endpoint. I prefer user-assigned identities in production — stable, pre-authorized before deployment, specified by client ID — one per workload per environment with data-plane roles at the narrowest scope, and I design for propagation: new assignments take minutes and the platform caches tokens for up to a day, so revocations aren't instant. The main threat is SSRF reaching IMDS, and outside Azure the equivalent is workload identity federation."*

---

## Concept 38 — Workload identity federation

**Workload identity federation** lets an Entra app registration or user-assigned managed identity **trust tokens issued by an external OIDC issuer** for a specific subject — so a workload running *outside* Entra's direct control can get Entra tokens **without any stored secret**. You configure a **federated identity credential (FIC)**: `issuer` + `subject` (+ `audience`, usually `api://AzureADTokenExchange`). At runtime the workload presents the external token as a `client_assertion`; Entra validates it against the issuer's published keys and the configured subject, and issues an Entra token.

```
GitHub Actions job (repo contoso/orders, environment prod)
   │ 1. request OIDC token from GitHub: iss=https://token.actions.githubusercontent.com,
   │    sub=repo:contoso/orders:environment:prod, aud=api://AzureADTokenExchange
   ▼
Entra token endpoint ── 2. client_assertion=<GitHub token>, client_id=<app or UAMI client ID>
   │ 3. checks FIC: issuer and subject match exactly; signature valid via GitHub's JWKS
   ▼
Entra access token for ARM / Key Vault / Storage  ── 4. deploy
```

Common issuers and their subject formats:

| Workload | Issuer | Subject |
|---|---|---|
| **GitHub Actions** | `https://token.actions.githubusercontent.com` | `repo:org/repo:environment:prod`, `repo:org/repo:ref:refs/heads/main`, `repo:org/repo:pull_request` |
| **Azure DevOps** service connections | Azure DevOps' issuer for your organization | The service connection identifier |
| **AKS workload identity** | The cluster's **OIDC issuer URL** | `system:serviceaccount:<namespace>:<service-account>` |
| **Other Kubernetes** (on-premises, EKS, GKE) | The cluster's OIDC issuer | Same pattern |
| **AWS / GCP workloads** | Their identity tokens via OIDC | Role/service account identity |
| **Managed identity as FIC** (GA May 2025) | Entra itself (the managed identity's tenant issuer) | The managed identity's object ID |

**AKS workload identity** (which replaced the deprecated pod-managed identity): the service account is annotated with the identity's client ID; the mutating webhook projects a short-lived service account token into the pod and sets `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_FEDERATED_TOKEN_FILE`; `WorkloadIdentityCredential` exchanges it.

**Managed identity as FIC** solves a specific problem: an **app registration** (needed for multi-tenant scenarios, for exposing APIs, or for calling resources in *other* tenants) traditionally needed a secret or certificate. Now the app trusts a user-assigned managed identity attached to the workload; the workload gets an MI token for the exchange audience and presents it as the app's client assertion — **cross-tenant access with no secret anywhere**.

**Security properties and pitfalls:**

1. **The subject is the whole security boundary.** `repo:contoso/orders:ref:refs/heads/main` lets *anyone who can push to main* deploy; `repo:contoso/orders:pull_request` lets *any PR author* (including forks, depending on settings) get the identity — never federate production identities to PR subjects. Use **GitHub environments with required reviewers** and federate production to `environment:prod`.
2. **Narrow the identity's roles** as you would any other — a deployment identity scoped to its own resource group, not the subscription.
3. **Limits**: up to 20 FICs per app or user-assigned identity; exact subject matching (flexible/wildcard matching was in preview at last check — verify before relying on it).
4. **Trust only issuers you control or explicitly trust**; the external issuer's compromise is your compromise.

**The interview-grade sentence:** *"Workload identity federation lets an app registration or user-assigned managed identity trust an external issuer's token for an exact subject — GitHub Actions, Azure DevOps, AKS or any Kubernetes OIDC issuer, AWS or GCP, and since May 2025 a managed identity itself — so CI pipelines and workloads outside Azure get Entra tokens with no stored secret, and multi-tenant apps can drop their client secrets. The subject is the whole security boundary, so production identities federate only to protected environments with required reviewers, never to pull-request subjects, and keep least-privilege roles."*

---

## Concept 39 — Conditional Access and phishing-resistant authentication

**Conditional Access (CA)** is Entra's policy engine — the Zero Trust "verify explicitly" decision point. Each policy is *if these signals, then these controls*:

| Assignments (signals) | Conditions | Grant controls | Session controls |
|---|---|---|---|
| Users, groups, roles, guests, **workload identities** (with Workload ID Premium) | Sign-in risk and user risk (P2), device platform, **device compliance/hybrid join**, location and named networks, client apps, **authentication flows** (e.g., block device code flow), insider risk | Block; require MFA; require **authentication strength** (e.g., phishing-resistant); require compliant device; require approved app; require password change; terms of use | **Sign-in frequency**, persistent browser session, app-enforced restrictions, **token protection**, CAE strictness |

Policies apply to **target resources** (enterprise apps, including your own APIs and apps, and user actions such as registering security info). Evaluation happens at **token issuance**, so CA enforces at sign-in and token refresh — and with CAE, near-real-time for supporting resources (Concept 40).

**A baseline every tenant should have** (Microsoft publishes these as templates):

1. **Require MFA for all users** (and the Azure management apps are already covered by Microsoft's mandatory MFA).
2. **Require phishing-resistant MFA for administrators** — via an **authentication strength** policy.
3. **Block legacy authentication** (protocols that can't do MFA: basic auth for IMAP/POP/SMTP, old Office clients).
4. **Require compliant or hybrid-joined devices** for sensitive apps or admin portals.
5. **Risk-based policies** (P2): high sign-in risk → require MFA; high user risk → require secure password change or block.
6. **Block device code flow** except for the users and devices that need it.
7. **Workload identity policies** (Workload ID Premium): restrict service principals to known IP ranges; block risky service principals.
8. **Report-only first**, then enforce; always **exclude break-glass accounts** (Concept 41) and test in a non-production tenant.

**Phishing resistance — what it means.** One-time codes (SMS, voice, TOTP apps) and push approvals can be **phished in real time** by adversary-in-the-middle proxies (Evilginx-style kits relay the login and steal the session cookie) or worn down by **MFA fatigue** spam. **Phishing-resistant** methods cryptographically bind the authentication to the legitimate origin, so a proxy on a look-alike domain gets nothing usable:

| Method | Phishing-resistant | Notes |
|---|---|---|
| **Passkeys (FIDO2/WebAuthn)** — device-bound (security keys, Authenticator app passkeys) or **synced** (GA in Entra in March 2026, with passkey profiles) | ✅ | The origin is part of the signed challenge; private key never leaves the authenticator (or the sync fabric) |
| **Windows Hello for Business / platform SSO on macOS** | ✅ | Device-bound credentials |
| **Certificate-based authentication** (smart cards, derived credentials) | ✅ | Common in government |
| Authenticator push with **number matching** | Partially (resists fatigue, not AiTM proxies) | Better than plain push |
| TOTP codes, SMS, voice | ❌ | Phishable; SMS also vulnerable to SIM swap |

**Note what phishing-resistant MFA does *not* fix:** theft of the session *after* sign-in (cookie/token theft by malware on the device). That needs device compliance, token protection/binding, short sessions and CAE.

**The interview-grade sentence:** *"Conditional Access is Entra's policy decision point: signals like user, role, device compliance, location, risk and authentication flow, mapped to controls like block, MFA, a phishing-resistant authentication strength, compliant device or sign-in frequency, evaluated at token issuance. My baseline is MFA for everyone, phishing-resistant strength for admins, legacy authentication blocked, compliant devices for sensitive apps, risk-based policies, device code flow blocked except where needed and location policies for workload identities — deployed report-only first with break-glass accounts excluded. Passkeys, including synced passkeys that went GA in March 2026, resist adversary-in-the-middle phishing because the origin is in the signed challenge; OTP and push don't — and none of it stops stolen session tokens, which needs device and token-binding controls."*

---

## Concept 40 — Token lifetime, revocation and continuous access evaluation

The fundamental tension of JWTs: **self-contained tokens are fast because the API doesn't call the IdP, but that also means the API can't know when the token should stop working.** If a user is disabled at 10:00, their access token issued at 09:55 keeps working until ~11:00.

Ways to shrink that window:

| Approach | Revocation latency | Cost |
|---|---|---|
| Short access token lifetime (e.g., 5–10 minutes) | Minutes | More refreshes → more IdP load and latency; Entra's default is 60–90 minutes |
| Opaque tokens + introspection (Concept 19) | Immediate (modulo introspection cache) | A network call per request; IdP becomes a hard runtime dependency |
| Revocation lists / deny lists in the API (by `jti`, `sub`, `sid`) | Seconds | You build and distribute the list |
| Session-bound checks at your BFF/gateway | Seconds | Works only where you control the session |
| **Continuous Access Evaluation (CAE)** | Near real time for supported resources | Client and resource must support it |

**CAE in Entra** is based on the OpenID **Shared Signals** / CAEP ideas. The IdP and **CAE-capable resource providers** (Exchange Online, SharePoint Online, Teams, Microsoft Graph) maintain a conversation: on **critical events** — user disabled or deleted, password changed or reset, MFA enabled, admin explicitly revokes refresh tokens, high user risk detected — and on **Conditional Access location policy** changes (IP no longer allowed), the resource rejects the existing token with a **claims challenge** (`401` with `WWW-Authenticate: ... error="insufficient_claims", claims="..."`), and the CAE-capable client goes back to Entra for a new token, which is then evaluated against current policy. Because revocation is pushed, CAE-capable clients get *longer-lived* tokens (up to 28 hours), which improves resilience during IdP outages.

For **your own APIs**, the practical takeaways:

1. **Be a good CAE client** when calling Graph: declare the `cp1` client capability (MSAL/Microsoft.Identity.Web option `ClientCapabilities`) and handle claims challenges by re-acquiring tokens with the returned claims (Microsoft.Identity.Web does this for `IDownstreamApi`).
2. **As a resource server**, CAE events aren't automatically delivered to arbitrary custom APIs; you still rely on short lifetimes, your own session/revocation checks for high-risk operations, and step-up (Concept 28). Check current Entra documentation for custom-resource support before designing around it.
3. **Admin "revoke sessions"** (revoke refresh tokens) is the standard incident action for a compromised user; for a compromised **service principal**, remove/rotate its credentials and role assignments, and remember cached managed identity tokens (Concept 37).

**The interview-grade sentence:** *"Self-contained tokens trade revocability for speed: a disabled user's Entra access token keeps working for up to 60 to 90 minutes. I shrink that window with short lifetimes where it's worth the extra refreshes, introspection or deny lists for high-risk operations, and session checks at the BFF; and with Entra I use Continuous Access Evaluation, where CAE-capable resources like Graph reject tokens with a claims challenge on critical events such as account disablement, password reset or a location policy change — so my clients declare the cp1 capability and handle claims challenges — while for my own APIs I don't assume CAE events arrive and design revocation-sensitive paths explicitly."*

---

## Concept 41 — Privileged access: Entra roles, PIM, break-glass and mandatory MFA

Two separate permission systems govern an Azure estate, and both must be secured:

| | **Microsoft Entra roles** (directory roles) | **Azure RBAC roles** |
|---|---|---|
| Govern | The **tenant**: users, groups, apps, policies, licenses | **Azure resources**: subscriptions, resource groups, resources and their data planes |
| Examples | Global Administrator, Privileged Role Administrator, Application Administrator, Conditional Access Administrator, Security Reader | Owner, Contributor, User Access Administrator, Key Vault Secrets User, Storage Blob Data Reader |
| Scope | Tenant (or administrative units, app-specific scopes) | Management group → subscription → resource group → resource |
| Escalation path | Global Admin can **elevate** to User Access Administrator on all subscriptions (`elevateAccess`) — so tenant admin implies Azure admin | Owner/User Access Administrator can grant any role, including to themselves |

**Privileged Identity Management (PIM)** (Entra ID P2 / Governance) turns standing privilege into **just-in-time** privilege for both systems:

- Users are **eligible** for a role rather than **active**; they **activate** it for a bounded time (e.g., 1–8 hours) with justification, **MFA** (phishing-resistant via authentication context), optional **approval**, and a ticket number.
- Activations are **audited** and can trigger alerts; **access reviews** periodically re-certify eligibility.
- **PIM for groups** extends the same model to group membership (e.g., a group that holds production Owner).

**Break-glass (emergency access) accounts:**

- **Two** cloud-only accounts (on the `*.onmicrosoft.com` domain, not federated, not synced) with Global Administrator, for when Conditional Access or federation misconfiguration locks everyone out.
- **Strong, phishing-resistant credentials** — FIDO2 security keys stored physically securely (Microsoft's mandatory MFA applies to them too, so passwords alone no longer work for Azure management).
- Excluded from most CA policies (but not from monitoring); **alert on every sign-in**; test quarterly.

**Mandatory Azure MFA:** Microsoft now enforces MFA for Azure portal and admin centers (Phase 1, from October 2024) and for **create, update and delete operations through ARM from any client** — CLI, PowerShell, SDKs, REST, IaC (Phase 2, from **October 1, 2025**). **Workload identities aren't affected.** Consequence for architecture: **automation must run as managed identities, federated workload identities or service principals — never as user accounts with passwords.**

**Further hardening:**

- **Privileged Access Workstations** or dedicated admin browser profiles/devices for tenant and production administration.
- **Separate admin accounts** from daily-use accounts (no email or browsing as Global Admin).
- **Limit Global Administrators** (Microsoft recommends fewer than five); use least-privileged Entra roles (e.g., Application Administrator only for app owners).
- **Protect Tier 0**: Entra Connect servers, federation servers, PIM, CA administrators — compromise of these is compromise of the tenant.
- **Deployment stacks / resource locks** and **deny assignments** to protect critical resources from accidental or malicious deletion.

**The interview-grade sentence:** *"An Azure estate has two privilege systems — Entra directory roles governing the tenant and Azure RBAC governing resources, linked because a Global Administrator can elevate to User Access Administrator everywhere — and I put both behind PIM so admins are eligible rather than active, activating time-bound roles with phishing-resistant MFA, justification and approval, all audited and periodically reviewed. Two cloud-only break-glass accounts with FIDO2 keys are excluded from Conditional Access but alerted on every sign-in, Global Admins stay under five, and since Microsoft's Phase 2 MFA enforcement from October 2025 covers every ARM write from any client, all automation runs as managed or federated workload identities, never as users."*

---

## Concept 42 — Azure RBAC in depth

**The model**: a **role assignment** = **security principal** (user, group, service principal, managed identity) + **role definition** (a set of allowed `Actions`/`NotActions` for the control plane and `DataActions`/`NotDataActions` for the data plane) + **scope** (management group, subscription, resource group, or resource). Assignments **inherit downward**. Azure RBAC is **additive** (union of all assignments) except for **deny assignments**.

**Control plane vs data plane** — the distinction that most often makes or breaks least privilege:

| | Control plane | Data plane |
|---|---|---|
| Operations | Create/configure/delete resources: `Microsoft.Storage/storageAccounts/write`, `listKeys/action` | Read/write the data inside: `.../blobServices/containers/blobs/read` |
| Endpoint | `management.azure.com` (ARM) | Service endpoint (`*.blob.core.windows.net`, `*.vault.azure.net`, `*.servicebus.windows.net`) |
| Example roles | Owner, Contributor, Reader, Storage Account Contributor | Storage Blob Data Reader/Contributor, Key Vault Secrets User, Azure Service Bus Data Sender/Receiver, Cosmos DB Built-in Data Contributor (Cosmos uses its own data-plane RBAC role assignments) |
| Applications need | Almost never | Yes — the specific data actions |

The trap: **`Contributor` (control plane) on a storage account can call `listKeys` and get the account keys — full data access.** Giving an app `Contributor` "so it can read blobs" is both over-privileged and leads to keys. Give the app a **data-plane role** and **disable shared-key access** on the account (Concept 66).

**Built-in roles worth knowing for applications:**

| Service | Least-privilege roles |
|---|---|
| Storage | Storage Blob Data Reader / Contributor; Storage Queue Data Message Sender / Processor; Storage Table Data Reader/Contributor |
| Key Vault (RBAC model) | Key Vault Secrets User (read secret values), Key Vault Crypto User / Crypto Service Encryption User (key operations, CMK wrap/unwrap), Key Vault Certificate User, Key Vault Reader (metadata only), Secrets Officer / Crypto Officer / Certificates Officer (management of contents) |
| Service Bus / Event Hubs | Azure Service Bus Data Sender / Receiver / Owner; Azure Event Hubs Data Sender / Receiver |
| App Configuration | App Configuration Data Reader / Owner |
| Azure SQL | Not Azure RBAC for data: create **contained database users from Entra identities** (`CREATE USER [orders-api] FROM EXTERNAL PROVIDER`) and grant SQL permissions |
| Azure Monitor ingestion | Monitoring Metrics Publisher (for Entra-authenticated ingestion) |

**Custom roles**: define exactly the actions needed when no built-in role fits (e.g., "restart App Service, read logs, nothing else" for an on-call role). Keep them few; version them in code.

**Conditions (Azure ABAC)**: role assignments can carry **conditions** — e.g., Storage Blob Data Reader *only for blobs with index tag `tenant = contoso`*, or only for a container path prefix. Also used to **constrain delegation**: the **Role Based Access Control Administrator** role with a condition limiting *which* roles it can assign, so a team can grant `Storage Blob Data Reader` to its own identities without being able to grant Owner.

**Deny assignments** block actions regardless of role assignments; you can't create them directly — they come from deployment stacks (deny settings), managed applications and similar platform features.

**Operational practices:** assign roles to **groups** for humans and to **managed identities directly** for workloads; define all assignments in Bicep/Terraform; review with Entra access reviews and Defender for Cloud recommendations; watch for privileged roles (Owner, User Access Administrator, RBAC Administrator, Contributor) at broad scopes.

**The interview-grade sentence:** *"An Azure role assignment is principal plus role definition plus scope, inherited downward and additive except for deny assignments — and the decisive distinction is control plane versus data plane: an app reading blobs gets Storage Blob Data Reader on its container, never Contributor, which can list the account keys and read everything. I use built-in data roles like Key Vault Secrets User and Service Bus Data Sender at the narrowest scope, Entra contained users for Azure SQL, custom roles sparingly, ABAC conditions for tag- or path-scoped access and for constraining which roles a team may delegate, and everything in infrastructure code with periodic reviews."*

---

## Concept 43 — Customer identity (CIAM): Entra External ID and the alternatives

Customer identity has different priorities from workforce identity: **conversion** (sign-up friction costs revenue), **scale** (millions of users), **branding** (your look, your domain), **self-service** (account recovery without a help desk), **privacy and consent** (GDPR), and **fraud** (bots, fake accounts, credential stuffing).

**Microsoft Entra External ID** (external tenant configuration) provides:

- Sign-up/sign-in **user flows** with email + password, **email OTP**, **passkeys**, and **social IdPs** (Google, Facebook, Apple) or any OIDC/SAML provider;
- **Branding** and **custom URL domains**;
- **Native authentication** SDKs/APIs for fully in-app (non-browser) sign-in experiences on mobile and web — with email/SMS OTP MFA GA in early 2026 — at the cost of losing some browser-based protections;
- **Custom authentication extensions** (call your REST API during sign-up/token issuance to validate data or enrich claims);
- **Conditional Access and MFA**, fraud protection integrations, **just-in-time password migration** from legacy stores (GA in 2026);
- Standard OIDC/OAuth endpoints, so ASP.NET Core and MSAL apps integrate the same way as with workforce Entra.

**The landscape of alternatives** you should be able to compare:

| Option | Model | Strengths | Trade-offs |
|---|---|---|---|
| **Entra External ID** | Managed SaaS (Microsoft) | Azure-native, Microsoft compliance footprint, Entra admin and CA model, per-MAU pricing | Fewer deep-customization hooks than B2C custom policies; Microsoft-centric |
| **Auth0 (Okta Customer Identity)** | Managed SaaS | Rich extensibility (Actions), broad protocol support, strong developer experience | Cost at scale; another vendor |
| **Duende IdentityServer** | Self-hosted .NET library (commercial license above revenue/usage thresholds) | Full control, standards-complete (DPoP, PAR, FAPI), .NET-native | You operate it — patching, key management, availability, security reviews |
| **OpenIddict** | Self-hosted .NET library (Apache 2.0) | Free, flexible, standards-compliant server/client/validation stacks | Lower-level; you build UI and operations |
| **Keycloak** | Self-hosted Java server (CNCF) | Feature-rich, free | JVM ops; customization in Java |
| **ASP.NET Core Identity alone** | A user store and cookie auth in your app | Simple for a single monolithic app | You own passwords, MFA, recovery, breach response — and it isn't an OAuth server (Concept 61) |

**Decision heuristic:** **buy** (a managed CIAM) unless you have a specific reason — regulatory hosting constraints, deep protocol requirements, multi-product SSO you must control, or cost at very large scale — and the team to operate an identity provider as a critical, internet-facing, security-sensitive service. Running your own authorization server means owning signing key rotation, token endpoint availability (every sign-in depends on it), brute-force protection, account recovery fraud, and keeping up with standards — the classic hidden cost (Concept 73).

**The interview-grade sentence:** *"Customer identity optimizes for conversion, scale, branding, self-service recovery, privacy and fraud resistance, and on Azure I'd default to Entra External ID — user flows with email OTP, passwords, passkeys and social providers, custom domains, native authentication where in-app sign-in matters, custom authentication extensions and Conditional Access. I'd self-host Duende IdentityServer, OpenIddict or Keycloak only with a concrete reason like hosting constraints or protocol needs such as FAPI-grade DPoP and PAR, and a team ready to run an internet-facing identity provider, because owning key rotation, availability and account-recovery fraud is a large hidden cost."*

---

## Concept 44 — Agents as principals: identity and authority for AI

AI agents — software that plans, calls tools and takes actions — are a new kind of principal with an uncomfortable property: **their behavior is steered by text they read**, including text written by attackers (indirect **prompt injection**). The OWASP **Top 10 for Agentic Applications** (December 2025) catalogs the resulting risks — agent goal hijacking, tool misuse, identity and privilege abuse, agentic supply chain vulnerabilities, memory poisoning, cascading failures across agents — and the **2026 Top 10 for LLM Applications** (August 2026) keeps prompt injection at the top.

Security architecture for agents reuses everything in this module, with sharper emphasis:

1. **Give the agent an identity, not borrowed credentials.** Microsoft's answer is **Entra Agent ID** — agent identities (created from agent identity *blueprints*) that appear in the directory, sign in, get permissions and can be targeted by **Conditional Access and ID Protection for agents** (service plans rolling out from July 2026 with Agent 365 / Microsoft 365 E7). Many Agent ID features have been preview; check status. Outside Microsoft's ecosystem, the principle is the same: a workload identity per agent.
2. **Decide the authority model explicitly**:
   - **Delegated** — the agent acts *on behalf of* a user with a token whose scopes limit it to what that user consented to (OAuth code flow; MCP's model). The agent can never exceed the user.
   - **Autonomous** — the agent acts as itself with its own app permissions; these must be *very* narrow because there's no user to bound them.
3. **Least privilege per tool.** Each tool the agent can call is an API; scope tokens to that tool's audience (MCP's resource indicators), use read-only permissions where possible, and require **human approval** for irreversible or high-impact actions (payments, deletions, sending external messages) — RAR-style structured authorization (Concept 31) fits naturally.
4. **Never pass through tokens.** An MCP server or tool must obtain its own token for downstream APIs (OBO or its own identity), not forward the token it received (Concept 32).
5. **Treat all model input as untrusted data** — documents, web pages, emails and tool outputs may contain instructions. Prompt injection has no complete fix (there's no "parameterized query" for natural language), so the defense is architectural: limit what a hijacked agent *can* do (blast radius), separate privileged actions behind deterministic checks and approvals, and filter outputs that could exfiltrate data (e.g., rendering untrusted URLs).
6. **Audit like a privileged user.** Every agent action logged with the agent identity, the delegating user, the tool, parameters and outcome; anomaly detection on agent behavior.
7. **Threat-model the agent** with STRIDE plus the agentic list: spoofed tool servers, tampered memory, repudiation of agent actions, disclosure via tool outputs, cost-exhaustion DoS through runaway loops, and elevation through over-scoped tools.

**The interview-grade sentence:** *"I treat AI agents as a new principal type whose behavior can be steered by attacker-written text, so they get their own identities — Entra Agent ID in the Microsoft world — and an explicit authority model: delegated tokens bounded by what the user consented to, or autonomous app permissions kept very narrow. Each tool gets audience-bound, least-privilege tokens with no passthrough, irreversible actions require human approval, all model input is untrusted because prompt injection has no complete fix, and every action is audited with the agent, the delegating user and the tool — which is how I'd address OWASP's agentic risks like goal hijacking and tool misuse."*

---
# Part E — Secrets, keys and certificates: Azure Key Vault

## Concept 45 — Secrets, keys and certificates are different things

They're often lumped together as "secrets," but they have different natures, threats and lifecycles:

| | **Secret** | **Key** | **Certificate** |
|---|---|---|---|
| What it is | An opaque value your code needs to **read**: API key, password, connection string, webhook signing secret | Cryptographic key material used to **perform operations**: sign, verify, encrypt, decrypt, wrap, unwrap | An X.509 certificate binding a public key to an identity, plus its private key and a lifecycle policy |
| How it's used | Retrieved and used by the app | Ideally **never retrieved** — the app sends data to the vault/HSM, which performs the operation (`CryptographyClient`) | TLS server/client authentication, code signing, `private_key_jwt` client assertions, document signing |
| Main threat | Disclosure (copying the value) | Misuse of the key (signing or decrypting on an attacker's behalf) and, if exportable, disclosure | Private key disclosure; **expiry outages**; mis-issuance |
| Lifecycle | Rotation (manual or scripted) and expiry | Rotation with versioning; old versions retained for decrypt/verify | Issuance, **renewal** (now every ≤200 days for public TLS), revocation |
| In Key Vault | Secret object (up to 25 KB) | Key object (RSA, EC; symmetric AES keys in Managed HSM), software- or HSM-protected | Certificate object = key + secret (the PFX/PEM) + policy and metadata |

The design consequences:

1. **Prefer keys you use over secrets you read.** A secret has to leave the vault to be useful; an HSM-protected key never does. Example: instead of storing a client secret, store a *certificate* in Key Vault and have the identity library sign the client assertion **in the vault** — the private key never enters your process.
2. **Prefer no secret at all** (Concept 7). Key Vault is where *irreducible* secrets live, not the first answer.
3. **Certificates are an operational problem as much as a security one** — most certificate incidents are outages from expiry, not breaches (Concept 52).

**Why a vault rather than configuration or environment variables:**

| Property | `appsettings.json` / env vars / CI variables | Key Vault |
|---|---|---|
| Access control | Anyone who can read the config, the container image, the App Service settings blade or the pipeline | Per-identity RBAC on the data plane; can be per-secret |
| Audit | None | Every read logged (diagnostic logs to Log Analytics) |
| Rotation | Redeploy | New version; apps pick it up via references or reload |
| Exposure in images, repos, crash dumps | High | Values are fetched at runtime |
| HSM protection, non-exportable keys | No | Yes (Premium, Managed HSM) |

**The interview-grade sentence:** *"Secrets, keys and certificates are different: secrets are values the app must read, so their threat is copying; keys are used for operations and ideally never leave an HSM, so their threat is misuse; certificates bind keys to identities with a lifecycle whose most common failure is an expiry outage. I prefer eliminating secrets, then using keys in place — for example a Key Vault certificate signing client assertions without ever exporting the private key — and keep irreducible secrets in Key Vault for per-identity access control, audit and rotation rather than in configuration, images or pipeline variables."*

---

## Concept 46 — Key Vault tiers and HSM options

| Option | Tenancy | Key protection | Validation (per current Microsoft/partner docs) | Use it for |
|---|---|---|---|---|
| **Key Vault Standard** | Multi-tenant PaaS | Software-protected keys | FIPS 140-2 Level 1 (software) | Application secrets, certificates, software keys — the default |
| **Key Vault Premium** | Multi-tenant PaaS | **HSM-protected keys** (shared HSM fleet) available alongside software keys | HSMs validated FIPS 140-3 Level 3 (older pages say 140-2 L2/L3 — check) | CMK for Azure services, signing keys, compliance requiring HSM-backed keys |
| **Key Vault Managed HSM** | **Single-tenant** PaaS pool | HSM-only; customer-controlled **security domain**; local RBAC; symmetric AES keys supported | FIPS 140-3 Level 3 | Regulated workloads needing a dedicated HSM with Azure service integration (CMK for Storage, SQL, Disks); high throughput key operations |
| **Azure Cloud HSM** | **Single-tenant IaaS** HSM cluster (3 HSMs, synchronized) | Full customer administrative control; PKCS#11, CNG/KSP, JCE, OpenSSL interfaces | FIPS 140-3 Level 3 | Lift-and-shift of apps that talk to HSMs directly (PKI CAs, TDE with on-prem patterns, custom crypto), migrating from Dedicated HSM or on-prem HSMs; **no** native Azure-service CMK integration |
| ~~Azure Dedicated HSM~~ | Single-tenant appliances (Thales Luna) | — | — | **Retired for new customers**; existing customers supported until **July 31, 2028**; migrate to Cloud HSM or Managed HSM |
| **Azure Payment HSM** | Single-tenant | Payment-specific HSMs | PCI PTS HSM | Payment processing (PIN, card) — specialized |

How to choose:

1. **Default to Key Vault Standard** for application secrets and certificates. Most apps never need more.
2. **Premium** when you need HSM-protected keys (for example customer-managed keys or signing keys) without operating anything.
3. **Managed HSM** when compliance requires single-tenant, customer-controlled HSMs *and* you want PaaS integration. Remember the **security domain**: you download it at activation (protected by a quorum of your RSA keys); lose it and a disaster may be unrecoverable — it's a real operational responsibility.
4. **Cloud HSM** when applications need raw HSM interfaces (PKCS#11) and full HSM administration.

**Vault topology**: create **one vault per application per environment (per region if needed)** rather than a shared enterprise vault. Reasons: blast radius (Concept 4), simpler RBAC, independent throttling limits (Concept 49), clean deletion with the app. Vaults are cheap; per-operation pricing dominates.

**The interview-grade sentence:** *"I default to Key Vault Standard for application secrets and certificates, use Premium when I need HSM-protected keys like customer-managed or signing keys, Managed HSM when compliance demands a single-tenant, customer-controlled HSM that still integrates with Azure services — accepting the security-domain responsibility — and Azure Cloud HSM, the successor to the retired Dedicated HSM, for applications that need raw PKCS#11 access. And I deploy one vault per application per environment, for blast radius, simpler RBAC and independent throttling."*

---

## Concept 47 — Key Vault access control: RBAC is now the default

Key Vault has two planes:

- **Control plane** (`management.azure.com`): create/delete vaults, configure networking, set the access model — authorized with **Azure RBAC** always.
- **Data plane** (`https://<vault>.vault.azure.net`): read secrets, use keys, manage certificates — authorized by **one of two models**:

| | **Vault access policies** (legacy) | **Azure RBAC** (recommended; default for new vaults) |
|---|---|---|
| Granularity | Per vault; permission sets per principal (get, list, set… for secrets/keys/certs) | Per vault **or per individual secret/key/certificate**; built-in roles |
| Management | Vault-specific configuration | Same tooling as all Azure access: role assignments, PIM, Conditional Access for admins, Azure Policy, access reviews |
| Privilege-escalation risk | **High**: anyone with `Contributor` (or `Microsoft.KeyVault/vaults/write`) on the vault can **add an access policy granting themselves full data access** | Granting data access requires role-assignment permissions (Owner / User Access Administrator / RBAC Administrator), which are rarer and PIM-protected |
| Limits | 1,024 policies per vault | Subscription role-assignment limits |
| PIM / JIT for data access | No | Yes |

**What changed in 2026**: with **control-plane API version 2026-02-01**, **new vaults default to Azure RBAC** (`enableRbacAuthorization = true`) — matching what the portal already did. **All control-plane API versions before 2026-02-01 retire on February 27, 2027**, so IaC templates, SDKs and scripts must be updated; if you *must* create a legacy-model vault with the new API, you set `enableRbacAuthorization: false` explicitly. Existing vaults keep their model. Azure Policy has built-in definitions to **audit or deny** vaults not using RBAC.

**Least-privilege role choices:**

| Who | Role | Why |
|---|---|---|
| App that reads a secret | **Key Vault Secrets User**, scoped to the **secret** (or the vault if it's dedicated to the app) | Read values only; can't list or change other secrets if scoped per secret |
| App or Azure service using a key for CMK | **Key Vault Crypto Service Encryption User** | Wrap/unwrap/get only |
| App that signs with a key | **Key Vault Crypto User** | Cryptographic operations without key management |
| App that needs a certificate's private key (e.g., TLS) | **Key Vault Secrets User** on the certificate's backing secret, or **Key Vault Certificate User** | Certificates' private keys are exposed via the secret |
| Pipeline that writes secrets | **Key Vault Secrets Officer**, scoped narrowly, via federated identity | Manage secrets, not keys or access |
| Security team | **Key Vault Reader** (metadata) / Key Vault Administrator via PIM | Audit without reading values; JIT for break-fix |
| Delegated access management | **Key Vault Data Access Administrator** with a condition limiting assignable roles | Teams grant data roles without becoming Owners |

**Migration from access policies to RBAC**: inventory policies → map each to built-in roles → create role assignments (consider per-secret scope) → switch the vault's permission model (a brief window where access may break; do it per vault in a maintenance slot) → remove reliance on `Contributor` for data access.

**The interview-grade sentence:** *"Key Vault's control plane is always Azure RBAC, and its data plane is now RBAC by default for new vaults created with API version 2026-02-01 — older control-plane API versions retire on February 27, 2027, so templates and SDKs need updating. RBAC matters because with legacy access policies anyone with Contributor on the vault can grant themselves data access, while RBAC needs role-assignment rights that are rare and PIM-protected. So apps get Key Vault Secrets User scoped to their secrets, CMK services get Crypto Service Encryption User, pipelines get a narrowly scoped Secrets Officer through federation, and humans get Reader plus just-in-time administration."*

---

## Concept 48 — Key Vault networking: private endpoints, firewall and perimeters

Identity is the primary control; network restriction is a strong second layer (Concept 3) — particularly against **stolen tokens** used from outside your network.

| Option | How it works | Notes |
|---|---|---|
| **Public endpoint, all networks** | Default; reachable from anywhere; only identity protects it | Acceptable only for low-sensitivity vaults or early dev |
| **Firewall: selected networks / IPs** | Allow specific public IP ranges and VNet subnets (service endpoints) | "Allow trusted Microsoft services" exception lets certain Azure services (e.g., for CMK) bypass the firewall |
| **Private endpoint + public network access disabled** | Vault gets a private IP in your VNet; DNS (`privatelink.vaultcore.azure.net`) resolves the vault name to that IP; the public endpoint refuses traffic | The standard production pattern; requires private DNS done right (Concept 64) |
| **Network security perimeter (NSP)** — GA | A logical boundary around PaaS resources (Key Vault, Storage, Event Hubs, Service Bus, Azure Monitor, AI Search…); resources in the perimeter can talk to each other; inbound and outbound public access are **denied by default** with explicit access rules; **access logs** | Solves "PaaS-to-PaaS" traffic that private endpoints don't cover (e.g., Storage reaching Key Vault for CMK, Azure Monitor exports) and **outbound exfiltration** from PaaS; has a learning (transition) mode before enforcement |

Practical guidance:

1. **Production vaults: private endpoint + public network access disabled**, or an NSP in enforced mode — often both (private endpoints for your workloads, NSP for PaaS-to-PaaS and exfiltration control).
2. **Mind the deployment path.** Once public access is off, pipelines must deploy secrets from inside the network (self-hosted or VNet-injected runners, managed DevOps pools) or through deployment approaches that don't need data-plane access from the internet.
3. **Mind the developer path.** Developers can't read production secrets from laptops — that's a feature; give them their own dev vaults.
4. **Diagnostic logs** (`AuditEvent` category) to Log Analytics: who read which secret, from which IP, with which identity — the evidence trail for a leak investigation.

**The interview-grade sentence:** *"Network controls on Key Vault are a second layer behind identity, mainly so a stolen token is useless from outside: production vaults get a private endpoint with public network access disabled — which only works with private DNS done right — and increasingly a network security perimeter, GA now, which denies public inbound and outbound by default, covers PaaS-to-PaaS traffic like Storage reaching Key Vault for CMK, and logs access. Then deployment and developer paths have to be designed for a vault that's unreachable from the internet, and audit logs go to Log Analytics for investigations."*

---

## Concept 49 — Recovery, availability and limits

**Deletion protection:**

- **Soft delete** is always on for Key Vault: deleted vaults and objects are recoverable for a retention period of **7–90 days** (default 90). The *name* stays reserved during retention — a common surprise when IaC tries to recreate a vault with the same name.
- **Purge protection**: when enabled, nobody (including subscription owners) can permanently purge a soft-deleted vault or object until retention ends. **Once enabled it can't be disabled.** It's **required** when the vault holds **customer-managed keys** for Azure services — losing a CMK means losing the data it protects. Enable it on any vault whose loss would be catastrophic (and accept that the name stays locked for the retention period after deletion).
- **Backup/restore** of individual secrets/keys/certificates exists (restorable only within the same geography and subscription); **Managed HSM** has full backups. Treat backups as sensitive as the keys.

**Availability:** Key Vault is replicated within the region and to the **paired region** for read access during a regional outage (fails over to read-only); in regions without a pair, behavior differs — check. Managed HSM supports multi-region replication. Your app shouldn't depend on Key Vault being reachable *per request* anyway:

**Throttling and caching.** Key Vault has **service limits** per vault per region (on the order of thousands of transactions per 10 seconds for secrets and software keys, lower for HSM key operations — check the current limits page). An app that reads a secret **on every request** will be throttled (`429`) at scale and adds latency and a hard runtime dependency. Rules:

1. **Read secrets at startup and cache them** in memory, with periodic refresh (minutes to hours) to pick up rotations (Concept 53).
2. **Don't read secrets per request**; don't create a new `SecretClient` per call (singleton, like `HttpClient`).
3. For **key operations at high volume** (signing every request, envelope encryption per object), cache unwrapped DEKs briefly in memory where acceptable, or use Managed HSM for throughput.
4. **Retry with backoff** on 429 (the Azure SDK does this by default — Module 25's discipline applies).
5. **Separate vaults** per app (Concept 46) so one noisy app doesn't throttle another.

**The interview-grade sentence:** *"Key Vault always has soft delete with seven to ninety days of retention, and I enable purge protection — which can't be undone — on any vault whose loss would be catastrophic and always when it holds customer-managed keys, accepting that the name stays reserved after deletion. For availability and throttling, the app reads secrets at startup and caches them with periodic refresh rather than per request, uses singleton clients with backoff on 429s, caches unwrapped data keys briefly for high-volume crypto, and gets its own vault so it can't be throttled by a neighbor."*

---

## Concept 50 — Envelope encryption and customer-managed keys

Encrypting large data directly with an HSM key is slow (every byte through the HSM) and makes rotation expensive (re-encrypt everything). **Envelope encryption** solves both:

```
 DEK (data encryption key): random AES-256 key per object/blob/partition, generated locally
 KEK (key encryption key):  RSA or AES key in Key Vault / Managed HSM, never leaves the HSM

 Encrypt:  ciphertext = AES-GCM(DEK, plaintext)
           wrappedDEK = KeyVault.WrapKey(KEK, DEK)          ← one HSM call per DEK
           store { ciphertext, wrappedDEK, kekId+version }

 Decrypt:  DEK = KeyVault.UnwrapKey(KEK version, wrappedDEK)
           plaintext = AES-GCM-Decrypt(DEK, ciphertext)

 Rotate KEK: create KEK v2; new DEKs wrapped with v2; optionally re-wrap old DEKs (cheap: no data re-encryption);
             keep v1 enabled for unwrap until all DEKs are re-wrapped.
```

This is exactly how Azure services implement encryption at rest with **customer-managed keys (CMK)**: Storage, SQL TDE, Cosmos DB, Managed Disks, Service Bus Premium and others hold their own DEKs and call your KEK in Key Vault (via a managed identity with **Key Vault Crypto Service Encryption User**) to wrap/unwrap them.

**Platform-managed keys (PMK) vs customer-managed keys (CMK):**

| | Platform-managed (default) | Customer-managed |
|---|---|---|
| Who controls the KEK | Microsoft | You, in your Key Vault/Managed HSM |
| Data encrypted at rest | Yes — always, by default | Yes |
| What CMK adds | — | **Revocation** (disable the key → the service can't decrypt → data becomes inaccessible: a "kill switch"), **your** rotation policy and audit of key use, separation of duties, compliance requirements |
| Costs | None | Key Vault/HSM operations, operational risk (lose the key = lose the data; key unavailable = service outage), purge protection required |

**When CMK is worth it:** regulatory or contractual requirements; customers who demand control (common in B2B SaaS — "bring your own key" per tenant); separation of duties between data administrators and key administrators. **When it isn't:** most workloads. CMK doesn't protect against an attacker who has compromised your application (the app decrypts transparently); it protects against specific provider-side and administrative scenarios. Saying this precisely is a senior signal.

**Rotation**: Key Vault supports **automatic key rotation** via a **rotation policy** (e.g., rotate every 12 months, notify 30 days before expiry via Event Grid). Many Azure services support **versionless key URIs** and pick up the new version automatically.

**Application-level envelope encryption** for especially sensitive fields (national IDs, health data) uses the same pattern in your code — `CryptographyClient.WrapKeyAsync`/`UnwrapKeyAsync` with `AesGcm` locally — or **Always Encrypted** in Azure SQL, where column encryption keys are wrapped by a column master key in Key Vault and the database never sees plaintext.

**The interview-grade sentence:** *"Envelope encryption encrypts data locally with a per-object AES data key and wraps that key with a key-encryption key that never leaves Key Vault or the HSM, so there's one HSM call per data key and rotating the KEK means re-wrapping keys, not re-encrypting data — which is exactly how Azure services implement customer-managed keys. Data is encrypted at rest by default with platform-managed keys; CMK adds a revocation kill switch, my own rotation and audit and separation of duties, at the cost of operational risk and mandatory purge protection, so I use it for regulatory or per-tenant bring-your-own-key needs, knowing it doesn't protect against a compromised application."*

---

## Concept 51 — Secret rotation

Rotation limits how long a leaked secret stays useful and proves you *can* rotate during an incident. The order of preference:

1. **Eliminate the secret** (managed identity, federation, keyless access) — no rotation needed (Concept 7).
2. **Use credentials the platform rotates** — App Service managed certificates, Key Vault–integrated CA certificates with auto-renew, managed identities.
3. **Automate rotation** for what remains.
4. **Manual rotation with a calendar** — a last resort that will eventually fail.

**The dual-credential pattern** — rotation without downtime. Most providers allow two valid credentials at once (Storage account key1/key2, Entra app with two secrets/certificates, most API providers with two keys):

```
t0: apps use credential A (both A and B valid)
t1: regenerate B; write B to Key Vault as the new current version
t2: apps reload configuration and switch to B (cache refresh interval elapses)
t3: verify no usage of A (provider metrics / sign-in logs)
t4: regenerate (invalidate) A
next cycle: swap roles
```

**Event-driven automation on Azure:**

- Set an **expiration date** on every secret.
- Key Vault emits **Event Grid** events — `SecretNearExpiry` (30 days before), `SecretExpired`, plus certificate and key equivalents.
- An **Azure Function** (with a managed identity allowed to manage the target credential and write the secret) handles `SecretNearExpiry`: generates the new credential at the provider, writes a new secret version, and later revokes the old one.
- Monitor: alert if a secret is within N days of expiry with no new version — a failed rotation must page someone *before* it's an outage.

**Consumption must be rotation-aware** (Concept 53): apps that read a secret only at startup need a restart to pick up the new value; prefer periodic reload (`ReloadInterval`), platform references that refresh (App Service Key Vault references refresh on a schedule), or retry-on-auth-failure that re-reads the secret.

**Rotation during an incident** is the real test. When a secret leaks: rotate **immediately** (dual-credential makes this a fast, safe operation), review the provider's logs for misuse during the exposure window, then fix the root cause (how did it leak?) and, ideally, eliminate the secret (Worked example 5).

**The interview-grade sentence:** *"My rotation preference is to eliminate the secret, then to use platform-rotated credentials, then to automate: every secret has an expiry, Key Vault's SecretNearExpiry event on Event Grid triggers a Function that creates the new credential at the provider and writes a new version, using the dual-credential pattern — two valid keys, switch, verify, revoke the old one — so rotation never causes downtime. Consumers reload periodically or re-read on authentication failure, a rotation that hasn't happened near expiry pages someone, and the same machinery is what lets me rotate within minutes when a secret leaks."*

---

## Concept 52 — Certificates and their shrinking lifetimes

**The timeline (CA/Browser Forum Ballot SC-081v3, adopted April 2025) for publicly trusted TLS certificates:**

| Issued on or after | Maximum validity | Domain-validation data reuse |
|---|---|---|
| Before March 15, 2026 | 398 days | 398 days |
| **March 15, 2026 (in effect now)** | **200 days** | 200 days |
| March 15, 2027 | 100 days | 100 days |
| March 15, 2029 | **47 days** | **10 days** |

The industry's explicit intent is to **force automation**: with 47-day certificates, a team renewing manually would renew roughly eight times a year per certificate. Private CAs (your internal PKI, mTLS between services) aren't bound by these rules, but the same operational lesson applies. **Certificate pinning** of leaf certificates becomes untenable — pin to CA public keys if at all, and prefer not pinning.

**Automation options on Azure:**

| Option | What it automates | Notes |
|---|---|---|
| **App Service managed certificates** | Free certificates for custom domains, auto-renewed | Limited (no wildcard for some cases, not exportable) |
| **Front Door / Application Gateway managed certificates** | Edge TLS, auto-renewed | Edge only |
| **Key Vault certificates with an integrated CA** (DigiCert, GlobalSign) | Issuance and **auto-renewal** per certificate policy (e.g., renew at 80% of lifetime), with Event Grid notifications | Services consuming from Key Vault (App Service, Application Gateway, Front Door, AKS via CSI) pick up new versions — verify each consumer's refresh behavior |
| **ACME** (Let's Encrypt, others) via tooling such as cert-manager on AKS or a Function | Issuance and renewal over the ACME protocol | Standard for Kubernetes ingress |
| **Key Vault with a non-integrated CA** | CSR generation in the vault (private key never leaves); you complete issuance | Manual steps — automate with the CA's API |

**Operational rules:**

1. **Inventory every certificate** — public TLS, client certificates for `private_key_jwt`, mTLS, signing, SAML signing certificates in enterprise apps. Expiry outages almost always involve a certificate nobody knew about.
2. **Alert at multiple thresholds** (e.g., 30, 14, 7 days) on every certificate *and* on renewal job failures.
3. **Make consumers reload** — a renewed certificate in Key Vault doesn't help an app that loaded the old one at startup and never restarts. Use versionless references, platform bindings that sync, or periodic reload; Kestrel can be configured with a certificate selector that reloads.
4. **Keep private keys non-exportable** where possible; for `private_key_jwt` client authentication, let MSAL/Microsoft.Identity.Web **sign with the Key Vault key** rather than downloading the PFX.
5. **Plan for revocation and mis-issuance** — CAA DNS records restrict which CAs may issue for your domains; Certificate Transparency monitoring alerts you to unexpected certificates for your names.

**The interview-grade sentence:** *"Public TLS certificates are capped at 200 days since March 15, 2026, falling to 100 days in March 2027 and 47 days in March 2029 with ten-day domain-validation reuse, which makes manual renewal untenable and leaf pinning a bad idea. So every certificate is inventoried and automated — App Service or Front Door managed certificates, Key Vault certificates with an integrated CA and auto-renew policies, or ACME via cert-manager on AKS — with expiry and renewal-failure alerts, consumers that reload new versions instead of only reading at startup, non-exportable keys signed in the vault, and CAA records plus Certificate Transparency monitoring against mis-issuance."*

---

## Concept 53 — Consuming Key Vault from .NET

Four consumption patterns, from most to least decoupled:

**1. Platform Key Vault references** — the app reads normal configuration; the platform resolves the secret with the app's managed identity.

- **App Service / Functions**: an app setting with the value `@Microsoft.KeyVault(SecretUri=https://kv-orders-prod.vault.azure.net/secrets/PaymentApiKey/)` — **versionless** URIs pick up new versions (App Service re-fetches periodically, documented as within 24 hours, and on restart or configuration change).
- **Container Apps**: secrets defined as Key Vault references with a managed identity, then mapped to environment variables.
- **AKS**: the **Secrets Store CSI driver** with the Azure Key Vault provider mounts secrets as files (optionally synced to Kubernetes secrets), authenticated with workload identity; rotation polling is configurable.
- **Azure App Configuration Key Vault references**: App Configuration stores a *pointer*; the .NET App Configuration provider resolves it **client-side** with the app's own credential (App Configuration never sees the secret value).

Advantages: no Key Vault code in the app; secrets appear as configuration. Disadvantage: refresh timing is the platform's, not yours.

**2. The configuration provider** — `Azure.Extensions.AspNetCore.Configuration.Secrets`:

```csharp
var credential = builder.Environment.IsDevelopment()
    ? new AzureCliCredential()                                           // developer's own identity, dev vault
    : new ManagedIdentityCredential(ManagedIdentityId.FromUserAssignedClientId(builder.Configuration["Identity:ClientId"]!));

builder.Configuration.AddAzureKeyVault(
    new Uri(builder.Configuration["KeyVault:Uri"]!),
    credential,
    new AzureKeyVaultConfigurationOptions
    {
        ReloadInterval = TimeSpan.FromMinutes(30),        // pick up rotated versions without restart
        // Manager = new PrefixKeyVaultSecretManager("OrdersApi")  // optional: load only this app's secrets
    });

// Secret named "PaymentProvider--ApiKey" becomes configuration key "PaymentProvider:ApiKey".
builder.Services.Configure<PaymentProviderOptions>(builder.Configuration.GetSection("PaymentProvider"));
```

Consume via `IOptionsMonitor<T>` (not `IOptions<T>`) so reloaded values flow into the app. Note that the provider loads **all** secrets the identity can list unless you filter with a `KeyVaultSecretManager` — another reason for per-app vaults.

**3. The SDK clients directly** — `SecretClient`, `KeyClient`, `CertificateClient`, **`CryptographyClient`** (from `Azure.Security.KeyVault.*`):

```csharp
// Register once; clients are thread-safe and should be singletons.
builder.Services.AddAzureClients(clients =>
{
    clients.UseCredential(credential);
    clients.AddSecretClient(new Uri(builder.Configuration["KeyVault:Uri"]!));
});

// Sign a payload with a non-exportable HSM key — the private key never enters the process.
var crypto = new CryptographyClient(new Uri("https://kv-orders-prod.vault.azure.net/keys/receipt-signing"), credential);
byte[] digest = SHA256.HashData(payload);
SignResult signature = await crypto.SignAsync(SignatureAlgorithm.PS256, digest, ct);
```

Use these for keys (sign/verify, wrap/unwrap) and for dynamic secrets the app manages; wrap secret reads in a small cache with expiry.

**4. Identity libraries reading credentials from Key Vault** — Microsoft.Identity.Web can load a client certificate **from Key Vault** by reference (`"SourceType": "KeyVault"`), or better, use **signed assertions** and **managed identity as FIC** so there's no certificate to load.

**Rules across all patterns:**

- **Production uses a deterministic credential** (`ManagedIdentityCredential` / `WorkloadIdentityCredential`), not `DefaultAzureCredential`'s probing chain; if you keep `DefaultAzureCredential` for convenience, set **`AZURE_TOKEN_CREDENTIALS=prod`** so developer-tool credentials can never be picked up in production (Concept 58).
- **Developers use their own Entra identity** (`AzureCliCredential`, `VisualStudioCredential`) against **dev vaults**, plus `dotnet user-secrets` for purely local values — never copies of production secrets.
- **Never log configuration dumps** (Module 28 — `IConfiguration` debug views print every secret).
- **Fail fast at startup** if required secrets are missing, with a clear (secret-free) error.

**The interview-grade sentence:** *"I consume Key Vault in the most decoupled way that fits: platform references — App Service and Functions Key Vault references with versionless URIs, Container Apps secret references, the CSI driver on AKS, or App Configuration references resolved client-side — or the ASP.NET Core configuration provider with a reload interval and IOptionsMonitor so rotations flow without restarts, and the SDK's CryptographyClient for signing and wrapping with keys that never leave the vault. Production uses a deterministic managed-identity credential, developers use their own identities against dev vaults, clients are singletons, and configuration is never dumped to logs."*

---
# Part F — Making .NET do it right

## Concept 54 — The ASP.NET Core authentication pipeline

Module 18 placed `UseAuthentication()` and `UseAuthorization()` in the middleware pipeline; here's what they actually do.

**Schemes and handlers.** An **authentication scheme** is a name plus a **handler** plus options: `Cookies` (`CookieAuthenticationHandler`), `Bearer` (`JwtBearerHandler`), `OpenIdConnect` (`OpenIdConnectHandler`), certificate, negotiate, or custom. Handlers implement up to five actions:

| Action | Meaning | Cookie handler | JwtBearer handler | OIDC handler |
|---|---|---|---|---|
| **Authenticate** | Build a `ClaimsPrincipal` from the request | Decrypt and validate the cookie | Validate the bearer token | — (delegates to the sign-in scheme) |
| **Challenge** | Respond when an unauthenticated user hits a protected resource | Redirect to login (browser pages) or 401 (API endpoints in .NET 10) | `401` with `WWW-Authenticate: Bearer` | Redirect to the IdP's `/authorize` |
| **Forbid** | Respond when an authenticated user lacks permission | Redirect to access-denied page (or 403 for APIs) | `403` | — |
| **SignIn** | Persist a principal | Write the cookie | — | — |
| **SignOut** | Remove it | Delete the cookie | — | Redirect to end-session endpoint |

**The flow for a request:**

1. `UseAuthentication()` runs the **default authenticate scheme** and sets `HttpContext.User` (an anonymous principal if nothing validated — authentication middleware **never rejects** requests by itself).
2. Routing selects the endpoint; its metadata (`[Authorize]`, `RequireAuthorization()`, policies, schemes) is available.
3. `UseAuthorization()` evaluates the endpoint's policies; on failure it calls **Challenge** (not authenticated) or **Forbid** (authenticated but not authorized) on the relevant scheme(s).

**Typical configurations:**

```csharp
// API only: bearer tokens.
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"));   // Concept 55

// Web app or BFF: cookie session + OIDC sign-in.
builder.Services.AddAuthentication(OpenIdConnectDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApp(builder.Configuration.GetSection("AzureAd"));    // cookie is the sign-in scheme

// Mixed app: pick schemes per endpoint with policies or [Authorize(AuthenticationSchemes = ...)],
// or a "policy scheme" that forwards to Bearer when an Authorization header is present.
```

**.NET 10 behavior change**: for **known API endpoints** — `[ApiController]` controllers, minimal APIs reading or writing JSON, endpoints returning `TypedResults`, SignalR — **cookie authentication now returns 401/403 instead of redirecting to the login page**. Previously only `XMLHttpRequest`s got status codes; `fetch` calls got a 302 to an HTML login page. Mixed apps and SPAs whose JavaScript followed redirects must now handle 401/403 explicitly. .NET 10 also added **authentication and authorization metrics** (built-in meters you can export through OpenTelemetry — Module 28) for sign-ins, challenges, forbids and authorization outcomes.

**Pitfalls:**

- **Middleware order**: `UseAuthentication` must come before `UseAuthorization`, both after `UseRouting` (implicit in minimal hosting) and before endpoints; CORS before authentication for preflights.
- **Multiple schemes without a clear default** produce confusing challenges (an API returning an OIDC redirect).
- **Custom authentication handlers** that "accept" requests on validation errors — authentication failures must yield no principal, not a partial one.
- **API keys as authentication**: if you must accept them (partners, webhooks), implement a proper handler that maps the key to a principal with claims, compare in constant time against **hashed** stored keys, scope keys to specific operations, and rate-limit — and offer OAuth client credentials as the better option.

**The interview-grade sentence:** *"In ASP.NET Core an authentication scheme is a handler with authenticate, challenge, forbid, sign-in and sign-out behaviors: UseAuthentication only builds HttpContext.User from the default scheme and never rejects a request, while UseAuthorization evaluates the endpoint's policies and calls challenge for unauthenticated or forbid for unauthorized callers. APIs use the JwtBearer handler, web apps and BFFs a cookie plus OpenID Connect, and since .NET 10 cookie authentication returns 401 and 403 rather than login redirects for API endpoints, which SPA code must now handle — and .NET 10 also emits authentication and authorization metrics I'd export through OpenTelemetry."*

---

## Concept 55 — Validating JWT bearer tokens in ASP.NET Core

**With Microsoft.Identity.Web** (Entra and External ID) — the recommended path, because it configures issuer validation for single- and multi-tenant apps, accepts both audience forms, handles key rotation, and adds token acquisition for downstream calls:

```jsonc
// appsettings.json
"AzureAd": {
  "Instance": "https://login.microsoftonline.com/",
  "TenantId": "5e1c....",                 // or "organizations" for multi-tenant (then add tenant allow-listing)
  "ClientId": "0c5f....",                 // the API's own client ID
  "Audience": "api://orders-api"          // optional; client ID is accepted by default
}
```

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"));

builder.Services.Configure<JwtBearerOptions>(JwtBearerDefaults.AuthenticationScheme, o =>
{
    o.MapInboundClaims = false;                                 // keep "scp", "roles", "oid" as-is
    o.TokenValidationParameters.ClockSkew = TimeSpan.FromMinutes(1);
    o.TokenValidationParameters.ValidAlgorithms = [SecurityAlgorithms.RsaSha256];
});
```

**With plain JwtBearer** (any OIDC-compliant issuer — Duende, OpenIddict, Auth0, Keycloak):

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(o =>
    {
        o.Authority = "https://id.contoso.com";       // discovery + JWKS with automatic key refresh
        o.Audience  = "orders-api";                   // REQUIRED — never leave audience validation off
        o.MapInboundClaims = false;
        o.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,                    // defaults are true; stated for reviewers
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            RequireSignedTokens = true,
            RequireExpirationTime = true,
            ValidAlgorithms = [SecurityAlgorithms.RsaSha256, SecurityAlgorithms.EcdsaSha256],
            ValidTypes = ["at+jwt"],                  // RFC 9068 issuers; Entra access tokens use "JWT", so don't set this for Entra
            ClockSkew = TimeSpan.FromMinutes(1),
            NameClaimType = "name",
            RoleClaimType = "roles"
        };
        o.RefreshOnIssuerKeyNotFound = true;          // re-fetch JWKS on unknown kid (rotation)
    });
```

What to notice:

1. **`MapInboundClaims = false`** stops the legacy mapping of JWT claim names to long WS-* URIs (`sub` → `http://schemas.xmlsoap.org/.../nameidentifier`). Without it, `User.FindFirst("scp")` silently returns null and someone "fixes" the policy by removing it.
2. **`JsonWebTokenHandler`** is the default validator since .NET 8 (faster, async, uses `TokenValidationResult`); custom code that relied on `JwtSecurityToken` types in events must adapt.
3. **Never set `ValidateAudience = false` or `ValidateIssuer = false`** to "make it work." Find out what the token actually contains (the claims, the version — Concept 36) and configure accordingly.
4. **Multi-tenant**: with `TenantId = "organizations"`, Microsoft.Identity.Web validates the issuer *format* against `tid`; **you must still allow-list onboarded tenants** — e.g., in `OnTokenValidated`, reject if `tid` isn't a known customer, or in an authorization policy.
5. **Events** (`OnTokenValidated`, `OnAuthenticationFailed`, `OnChallenge`) are for enrichment and logging — log failures *without* the token itself (Module 28's redaction rules).
6. **Don't call the IdP per request**; JWKS are cached by `ConfigurationManager` (default refresh 24 hours, with an automatic refresh on an unknown `kid` at most every few minutes).

**The interview-grade sentence:** *"For Entra I validate bearer tokens with AddMicrosoftIdentityWebApi, which handles tenant-aware issuers, both audience forms and key rotation; for other issuers, JwtBearer with the authority for discovery and an explicit audience, issuer, lifetime and signing-key validation, an algorithm allow-list, the at+jwt type where the issuer sets it, small clock skew and MapInboundClaims off so scp and roles keep their names. I never disable audience or issuer validation to make something work, I still allow-list tenants for multi-tenant APIs, and I log validation failures without the token."*

---

## Concept 56 — Authorization in ASP.NET Core

ASP.NET Core authorization is **policy-based**: a **policy** is a set of **requirements**; each requirement is evaluated by one or more **handlers**; the policy succeeds when every requirement is satisfied and no handler fails it explicitly.

**1. Deny by default.** Make every endpoint require an authenticated user unless explicitly opted out:

```csharp
builder.Services.AddAuthorizationBuilder()
    .SetFallbackPolicy(new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build())                                         // applies to endpoints with no [Authorize]/[AllowAnonymous]
    .AddPolicy("orders:read", p => p
        .RequireScopeOrAppPermission(                     // Microsoft.Identity.Web helper
            allowedScopeValues: ["Orders.Read"],
            allowedAppPermissionValues: ["Orders.Read.All"]))
    .AddPolicy("orders:refund", p => p
        .RequireAuthenticatedUser()
        .RequireRole("Orders.Refunder")                   // app role assigned to the user
        .RequireClaim("scp", "Orders.Write"));            // AND the client was granted the scope

app.MapHealthChecks("/health/live").AllowAnonymous();    // explicit, reviewable exceptions
```

**2. Function-level authorization** — attach policies to endpoints, never rely on hiding URLs:

```csharp
var orders = app.MapGroup("/orders").RequireAuthorization("orders:read");
orders.MapPost("/{id:guid}/refund", RefundAsync).RequireAuthorization("orders:refund");
```

**3. Resource-based (object-level) authorization** — the defense against **BOLA/IDOR**, OWASP API1:2023. Policies on endpoints can't know which *object* is being accessed; check after loading it:

```csharp
public sealed class SameTenantRequirement : IAuthorizationRequirement;

public sealed class OrderTenantHandler : AuthorizationHandler<SameTenantRequirement, Order>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context, SameTenantRequirement requirement, Order order)
    {
        string? tid = context.User.FindFirst("tid")?.Value;               // from the validated token, never from the request
        string? oid = context.User.FindFirst("oid")?.Value;

        bool sameTenant = tid is not null && order.TenantId == tid;
        bool ownerOrAdmin = order.CustomerObjectId == oid || context.User.IsInRole("Orders.Admin");

        if (sameTenant && ownerOrAdmin) context.Succeed(requirement);
        return Task.CompletedTask;                                         // not calling Succeed = deny
    }
}

static async Task<IResult> GetOrderAsync(Guid id, OrdersDb db, IAuthorizationService authz, ClaimsPrincipal user)
{
    Order? order = await db.Orders.FindAsync(id);
    if (order is null) return TypedResults.NotFound();

    AuthorizationResult result = await authz.AuthorizeAsync(user, order, new SameTenantRequirement());
    return result.Succeeded ? TypedResults.Ok(order.ToDto()) : TypedResults.NotFound();   // 404, not 403: don't confirm existence
}
```

Better still for multi-tenant systems, **make cross-tenant access structurally impossible** with tenant-scoped queries (EF Core global query filters, SQL row-level security — Concept 63) *and* keep the resource-based check as a second layer.

**4. Property-level authorization** (API3:2023) — return **DTOs**, never entities (no accidental `PasswordHash`, `InternalNotes`, `TenantId` leakage), and **bind input to explicit DTOs** to prevent **mass assignment** (a client setting `IsAdmin: true` or `Price: 0` because the entity was model-bound).

**5. Patterns and pitfalls:**

- **Handlers must not throw on missing data to "deny"** — just don't succeed; and **never `context.Succeed` in a `catch`** (fail closed — Concept 8).
- **Avoid role explosion**: if you have `CanEditOrdersInRegionEastUnder1000`, you need attributes/policies evaluated against data (ABAC), not more roles. For complex relationship-based permissions (documents shared with groups, hierarchical orgs), consider a dedicated authorization service in the **Zanzibar/ReBAC** style (OpenFGA, SpiceDB) or a policy engine (OPA/Cedar) — but only when the model really needs it.
- **Authorization must be enforced server-side** in every service (complete mediation) — the gateway or BFF check is a first layer, not the only one.
- **Test authorization** as a matrix: for each endpoint × role × tenant, the expected status — integration tests with `WebApplicationFactory` and test tokens (Module 18) catch regressions that code review misses.

**The interview-grade sentence:** *"I make ASP.NET Core authorization deny-by-default with a fallback policy requiring an authenticated user and explicit AllowAnonymous exceptions, attach named policies to endpoint groups for function-level checks — scope or app role, using Microsoft.Identity.Web's helpers — and do object-level checks with resource-based authorization after loading the entity, taking tenant and user from the validated token and returning 404 rather than 403 to avoid confirming existence. Then I go further: tenant-scoped queries make cross-tenant access structurally impossible, DTOs prevent property-level leaks and mass assignment, handlers fail closed, and an endpoint-by-role-by-tenant test matrix guards against regressions."*

---

## Concept 57 — Calling downstream APIs from .NET

A web app or API that calls another protected API needs to **acquire tokens** correctly: right flow, right audience, cached, refreshed, and with CAE/claims-challenge handling. **Microsoft.Identity.Web** (on top of MSAL.NET) does this:

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"))
        .EnableTokenAcquisitionToCallDownstreamApi()               // this API becomes a confidential client
            .AddDownstreamApi("Inventory", builder.Configuration.GetSection("Inventory"))
            .AddDistributedTokenCaches();                          // token cache in IDistributedCache (Redis)

builder.Services.AddStackExchangeRedisCache(o => o.Configuration = builder.Configuration["Redis:Connection"]);
```

```jsonc
"AzureAd": {
  "Instance": "https://login.microsoftonline.com/",
  "TenantId": "5e1c....",
  "ClientId": "0c5f....",
  "ClientCredentials": [
    { "SourceType": "SignedAssertionFromManagedIdentity",          // managed identity as FIC — no secret, no certificate
      "ManagedIdentityClientId": "8d2a...." }
  ],
  "ClientCapabilities": [ "cp1" ]                                   // CAE-capable client
},
"Inventory": {
  "BaseUrl": "https://inventory.internal.contoso.com/",
  "Scopes": [ "api://inventory-api/Inventory.Read" ]
}
```

```csharp
// On behalf of the signed-in user (OBO — Concept 29):
app.MapGet("/orders/{id:guid}/availability", async (Guid id, IDownstreamApi api, CancellationToken ct) =>
{
    var stock = await api.GetForUserAsync<StockDto>("Inventory",
        o => o.RelativePath = $"stock/{id}", cancellationToken: ct);
    return TypedResults.Ok(stock);
}).RequireAuthorization("orders:read");

// As the application itself (client credentials), e.g., from a background job:
var report = await api.GetForAppAsync<ReportDto>("Inventory", o => o.RelativePath = "reports/daily");
```

What this gives you, and the decisions behind it:

1. **The right flow per call**: `GetForUserAsync` performs **OBO** with the incoming token; `GetForAppAsync` uses **client credentials**. Choose consciously (Concept 29).
2. **Token caching**: MSAL caches per user and per audience. For APIs and multi-instance web apps, use a **distributed** cache (Redis/SQL) — an in-memory cache per instance works but causes extra token requests and, for OBO, larger memory. Partition and size it; **encrypt** cache entries (Microsoft.Identity.Web's distributed cache adapter supports encryption with Data Protection — enable it, since entries contain refresh tokens). The 4.14 release added options such as partitioning the app token cache by audience.
3. **Credentials**: the `ClientCredentials` list is tried in order; prefer **`SignedAssertionFromManagedIdentity`** (managed identity as FIC), then a **Key Vault certificate**, and only then a secret.
4. **Claims challenges** (CAE, Conditional Access step-up) from downstream APIs are surfaced so the app can re-acquire with the required claims or bounce the user to interactive sign-in.
5. **Resilience**: token acquisition is a dependency on Entra (Module 13); MSAL refreshes proactively and Entra's token-issuance resilience features help, but cold caches after a deployment create bursts — warm caches, don't create MSAL clients per request, and add timeouts.
6. **Plain `HttpClient`**: if you don't use `IDownstreamApi`, add a `DelegatingHandler` that calls `ITokenAcquisition.GetAccessTokenForUserAsync/ForAppAsync` and attaches the header — never cache tokens yourself in static fields.

**For Azure resources** (Storage, Service Bus, Key Vault) you don't use this at all — you pass a `TokenCredential` to the Azure SDK client (Concept 58). **For Microsoft Graph**, `Microsoft.Identity.Web.GraphServiceClient` wires a Graph SDK client to the same token acquisition.

**The interview-grade sentence:** *"To call downstream APIs I use Microsoft.Identity.Web's token acquisition on MSAL: EnableTokenAcquisitionToCallDownstreamApi with AddDownstreamApi, GetForUserAsync for on-behalf-of and GetForAppAsync for app-only calls, a distributed and encrypted token cache for multi-instance services because it holds refresh tokens, credentials ordered from managed identity as a federated credential to Key Vault certificates with secrets last, the cp1 capability so CAE claims challenges are handled, and long-lived clients so token acquisition doesn't become a per-request dependency on Entra. Azure resources don't use this path at all — they take a TokenCredential."*

---

## Concept 58 — Azure SDK credentials and keyless services

Every Azure SDK client accepts a **`TokenCredential`** from **Azure.Identity**. The credential obtains Entra tokens for the service's audience (e.g., `https://storage.azure.com/.default`) and the SDK caches and refreshes them.

| Credential | Use |
|---|---|
| `ManagedIdentityCredential` | **Production on Azure** (App Service, Functions, Container Apps, VMs, AKS with managed identity) |
| `WorkloadIdentityCredential` | **AKS workload identity** / other federated Kubernetes |
| `ClientAssertionCredential` | Custom federation — supply a function returning an external token |
| `ClientCertificateCredential` / `ClientSecretCredential` | Workloads outside Azure without federation (last resort) |
| `AzureCliCredential`, `AzureDeveloperCliCredential`, `VisualStudioCredential`, `AzurePowerShellCredential` | **Local development** with the developer's own identity |
| `InteractiveBrowserCredential`, `DeviceCodeCredential` | Desktop tools and CLIs |
| `ChainedTokenCredential` | An explicit, ordered chain you choose |
| `DefaultAzureCredential` | An opinionated chain (environment → workload identity → managed identity → developer tools …) |

**`DefaultAzureCredential` in production — the nuance interviewers probe.** Microsoft's own guidance now says it's suitable for **early development**, not production, because:

- **Unpredictable**: it probes credentials in order; an unexpected environment variable or a present developer tool on a build agent changes *which identity* the app uses.
- **Slow failure**: probing failed credentials adds latency, especially on cold start (IMDS timeouts off-Azure).
- **Hard to debug**: "which credential authenticated?" requires logging.

Better production patterns:

```csharp
// 1. Deterministic per environment
TokenCredential credential = builder.Environment.IsProduction()
    ? new ManagedIdentityCredential(ManagedIdentityId.FromUserAssignedClientId(cfg["Identity:ClientId"]!))
    : new ChainedTokenCredential(new AzureCliCredential(), new VisualStudioCredential());

// 2. Or keep DefaultAzureCredential but constrain it with AZURE_TOKEN_CREDENTIALS (Azure.Identity 1.14+):
//    AZURE_TOKEN_CREDENTIALS=prod                     → only Environment, WorkloadIdentity, ManagedIdentity
//    AZURE_TOKEN_CREDENTIALS=ManagedIdentityCredential → exactly one credential (1.15+)
//    and use the constructor overload that *requires* the variable, so a missing setting fails fast.

builder.Services.AddAzureClients(c =>
{
    c.UseCredential(credential);                                       // one credential for all clients
    c.AddBlobServiceClient(new Uri(cfg["Storage:BlobEndpoint"]!));
    c.AddServiceBusClientWithNamespace(cfg["ServiceBus:Namespace"]!);   // "contoso-prod.servicebus.windows.net"
});
```

**Keyless services — remove the secret, then remove the possibility of the secret.** Using Entra auth isn't enough if the service still *accepts* keys; anyone who finds a key can bypass your identity controls. Disable local/shared-key authentication:

| Service | Setting | Effect |
|---|---|---|
| Storage | `allowSharedKeyAccess: false` | Account keys and account-key SAS stop working; use Entra and **user delegation SAS** |
| Service Bus / Event Hubs | `disableLocalAuth: true` | SAS keys and connection strings with keys rejected |
| Cosmos DB | `disableLocalAuth: true` | Primary/secondary keys rejected; data-plane RBAC only |
| Azure SQL | **Microsoft Entra-only authentication** | SQL logins disabled; use contained Entra users and `Authentication=Active Directory Managed Identity` in the connection string |
| App Configuration | `disableLocalAuth: true` | Access keys rejected |
| Azure OpenAI / AI Services | `disableLocalAuth: true` | API keys rejected |
| Application Insights | `DisableLocalAuth` | Ingestion requires Entra (Module 28) |
| Redis (Azure Managed Redis / Cache for Redis) | Entra ID authentication; disable access keys | Access keys rejected |

Enforce with **Azure Policy** (built-in definitions exist for most of these: "should have local authentication methods disabled") so new resources can't regress.

```csharp
// Azure SQL with a user-assigned managed identity — no password anywhere.
var cs = "Server=tcp:contoso-sql.database.windows.net;Database=orders;" +
         "Authentication=Active Directory Managed Identity;User Id=8d2a....;Encrypt=True;";
```

**The interview-grade sentence:** *"Every Azure SDK client takes a TokenCredential, and in production I use a deterministic one — ManagedIdentityCredential with the user-assigned client ID, or WorkloadIdentityCredential on AKS — because Microsoft now positions DefaultAzureCredential as a development convenience whose probing makes the chosen identity unpredictable; if a team keeps it, AZURE_TOKEN_CREDENTIALS=prod restricts it to deployed-service credentials. Then I remove the possibility of keys, not just their use: shared-key access off on Storage, local auth off on Service Bus, Event Hubs, Cosmos, App Configuration and AI services, Entra-only authentication on Azure SQL — enforced by Azure Policy so nothing regresses."*

---

## Concept 59 — ASP.NET Core Data Protection

**Data Protection** is the subsystem that protects data the app gives to clients and expects back unmodified and unreadable: **authentication cookies**, **antiforgery tokens**, **TempData**, OIDC **state**/correlation cookies, and anything you protect with `IDataProtector`. It provides authenticated encryption (AES-256-CBC + HMAC-SHA256 by default) with a **key ring** that it rotates automatically (default key lifetime **90 days**; old keys are kept to decrypt).

Why it's an architecture concern:

1. **Multiple instances must share the key ring.** By default keys are stored locally (file system or, on App Service, a synced folder); in containers they're **ephemeral** — every restart or new replica creates new keys, so cookies issued by pod A fail on pod B, and users are logged out on every deploy. Persist the key ring centrally.
2. **The key ring is a high-value secret.** Anyone who has it can forge authentication cookies — i.e., impersonate any user. Protect it **at rest**.
3. **Application isolation.** Apps sharing a key ring and the same application name can decrypt each other's payloads. Set an explicit application name: the same value for instances of one app (and for apps that intentionally share cookies), different values otherwise.

```csharp
builder.Services.AddDataProtection()
    .SetApplicationName("contoso-orders-web")
    .PersistKeysToAzureBlobStorage(new Uri("https://contosokeys.blob.core.windows.net/dataprotection/keys.xml"), credential)
    .ProtectKeysWithAzureKeyVault(new Uri("https://kv-orders-prod.vault.azure.net/keys/dataprotection"), credential);
// packages: Azure.Extensions.AspNetCore.DataProtection.Blobs, Azure.Extensions.AspNetCore.DataProtection.Keys
```

Guidance:

- **Don't use Data Protection for long-term storage encryption** (e.g., database columns). Keys rotate and expire for *protection*, and revoking a key makes payloads unreadable by design. For data at rest, use envelope encryption with Key Vault (Concept 50) or Always Encrypted.
- **Use purpose strings** with `IDataProtectionProvider.CreateProtector("Contoso.Orders.ResetLink.v1")` so payloads for one purpose can't be replayed for another; use `ToTimeLimitedDataProtector()` for expiring tokens (password reset links, email confirmation).
- **Revoking keys** (`IKeyManager.RevokeAllKeys`) invalidates all cookies — an incident-response lever if the key ring may be compromised.

**The interview-grade sentence:** *"ASP.NET Core Data Protection encrypts and authenticates what the app round-trips through clients — auth cookies, antiforgery tokens, OIDC state — with a key ring that rotates every 90 days by default. With more than one instance or in containers, I persist the key ring centrally, for example to Blob Storage, encrypt it at rest with a Key Vault key, and set an explicit application name for isolation, because whoever holds the key ring can forge any user's cookie. I use purpose strings and time-limited protectors for things like reset links, and never use Data Protection for long-term data encryption, which is Key Vault envelope encryption's job."*

---

## Concept 60 — Injection, XSS, CSRF, CORS and SSRF

The classic web vulnerability classes, with what .NET does by default and what you still must do:

| Class | Mechanism | .NET defaults that help | What you still must do |
|---|---|---|---|
| **SQL injection** (A05:2025) | Untrusted input concatenated into queries | EF Core LINQ and `FromSql($"...")` / `ExecuteSql($"...")` **parameterize interpolated values**; Dapper parameters | Never use `FromSqlRaw`/`ExecuteSqlRaw` with concatenated input; whitelist dynamic identifiers (sort columns) against known names; least-privilege DB users |
| **Other injection** | OS commands, LDAP, XPath, template, log injection, NoSQL query operators | `Process.Start` with `ArgumentList` (no shell); structured logging templates (Module 28) | Avoid shells; validate; encode for the sink |
| **XSS** (cross-site scripting) | Untrusted data rendered as HTML/JS | Razor and Blazor **HTML-encode by default**; JSON serialization escapes | Never `@Html.Raw`/`MarkupString` with untrusted data; sanitize rich HTML with an allow-list sanitizer; **Content Security Policy** with nonces; avoid `innerHTML` in SPAs |
| **CSRF** | Browser sends cookies automatically on cross-site requests | **Antiforgery** built into Razor Pages/MVC forms and Blazor; `SameSite=Lax` default for auth cookies | Validate antiforgery for cookie-authenticated state-changing endpoints (minimal APIs need `[ValidateAntiForgeryToken]`/`RequireAntiforgery` patterns or a custom-header check — Concept 30); bearer-token APIs aren't CSRF-prone because browsers don't attach bearer tokens automatically |
| **CORS misconfiguration** | Over-permissive cross-origin access | CORS **off** by default; `AllowAnyOrigin` + `AllowCredentials` **throws** | Explicit origin allow-list; never reflect the `Origin` header; credentials only for your own front-ends. CORS is *not* an authorization control — it only governs browsers |
| **SSRF** (now inside A01:2025) | Server fetches a URL the attacker controls → reaches internal services, **IMDS (managed identity tokens)**, cloud metadata | None automatic | Allow-list destinations; resolve DNS and **reject private, loopback, link-local (169.254.0.0/16) and metadata addresses** — check *after* resolution and on every redirect (or disable redirects); a `SocketsHttpHandler.ConnectCallback` can enforce the IP check at connect time; egress firewall rules (Concept 64) |
| **Path traversal / unsafe file handling** | `../` in file names; executable uploads | `Path.GetFullPath` helps; `IFormFile` streams | Generate server-side names; store uploads in Blob Storage, not the web root; validate type by content; scan; size limits |
| **Deserialization attacks** (A08) | Polymorphic deserialization instantiating attacker-chosen types | `System.Text.Json` doesn't do unsafe polymorphism by default; **`BinaryFormatter` is removed** (.NET 9) | Never enable type-name handling (`TypeNameHandling.All` in Newtonsoft) on untrusted input; use explicit polymorphism allow-lists |
| **Open redirects** | `returnUrl` sends users to attacker sites | `LocalRedirect` / `Url.IsLocalUrl` | Use them for every user-supplied redirect target |
| **ReDoS** | Catastrophic regex backtracking | `RegexOptions.NonBacktracking` (.NET 7+); match timeouts | Use them on untrusted input |
| **Header and transport hardening** | Downgrade, clickjacking, MIME sniffing | `UseHsts`, `UseHttpsRedirection` in templates | Add CSP, `X-Content-Type-Options: nosniff`, `frame-ancestors`/`X-Frame-Options`, `Referrer-Policy`; trust `X-Forwarded-*` only from known proxies (`ForwardedHeadersOptions.KnownProxies/KnownNetworks`) |
| **Error leakage** (A10:2025) | Stack traces and internals in responses | `UseExceptionHandler` + ProblemDetails; developer page only in Development | Generic errors to clients, details to logs (without secrets); fail closed on security exceptions |

**Rate limiting and resource consumption** (API4:2023, Unrestricted Resource Consumption): ASP.NET Core's **rate limiting middleware** (`AddRateLimiter` — fixed window, sliding window, token bucket, concurrency limiters, partitioned per user/tenant/IP) plus Kestrel limits (`MaxRequestBodySize`, request header limits), pagination caps, query complexity limits for GraphQL/OData, and timeouts. Edge rate limiting (Front Door/APIM) handles volume; application limits handle per-tenant fairness and expensive operations (Module 6's overload protection).

**The interview-grade sentence:** *"The .NET defaults stop a lot — EF Core parameterizes interpolated SQL, Razor and Blazor encode output, antiforgery and SameSite cookies counter CSRF, CORS is off and refuses any-origin with credentials, BinaryFormatter is gone — and my job is not to undo them and to cover the rest: no raw SQL concatenation, a strict CSP, antiforgery or a custom-header check on cookie-authenticated minimal APIs, explicit CORS origins, SSRF defenses that resolve DNS and block private, link-local and metadata addresses on every redirect because IMDS hands out managed identity tokens, LocalRedirect for return URLs, non-backtracking regexes, hardened headers with trusted proxies only, generic error responses and rate limiting partitioned per tenant."*

---

## Concept 61 — Passwords, ASP.NET Core Identity and passkeys

**First question: should you hold credentials at all?** Every password you store is a liability: hashing, breach response, credential stuffing defense, account recovery (the most abused flow), MFA, lockout, compliance. For most systems, **delegating authentication to an IdP** (Entra, External ID, Auth0…) via OIDC is safer and cheaper (Concept 43). ASP.NET Core Identity is appropriate for self-contained apps where an external IdP is truly not an option, or as the user store *behind* a self-hosted authorization server (Duende/OpenIddict).

**What ASP.NET Core Identity gives you** — and the details worth knowing:

- **Password hashing**: PBKDF2 with **HMAC-SHA512, 100,000 iterations**, 128-bit salt, 256-bit subkey (the V3 format default since .NET 7). Hashes upgrade transparently on next login when parameters change. Never write your own; never use fast hashes (SHA-256) or encryption for passwords. (Argon2id is OWASP's first recommendation; PBKDF2 at high iteration counts remains acceptable and FIPS-friendly.)
- **Lockout** after failed attempts (configurable), **security stamps** to invalidate sessions on password change, **two-factor** (TOTP authenticator apps, recovery codes), email confirmation and password reset tokens via Data Protection (Concept 59).
- **Identity API endpoints** (`MapIdentityApi`, .NET 8+) for SPA/mobile scenarios — note they issue Identity's own bearer tokens, *not* standards-based OAuth tokens; suitable for a first-party app, not as an OAuth server.

**Passkeys in .NET 10.** ASP.NET Core Identity now has **built-in WebAuthn passkey registration and authentication**:

- Requires the Identity **schema version 3** (`IdentitySchemaVersions.Version3`) and an EF Core migration adding the passkey table.
- Configured via `IdentityPasskeyOptions` — most importantly **`ServerDomain`** (the WebAuthn **relying party ID**); set it explicitly rather than inferring from the `Host` header, which weak host validation could abuse.
- The **Blazor Web App** template with Individual Accounts includes the passkey UI; other app types wire the endpoints and a small JavaScript helper.
- Deliberately scoped to **authentication**; **attestation validation** (proving the authenticator's make/model, needed in some regulated environments) isn't included — use a library such as `fido2-net-lib` if you need it.
- Requires HTTPS (secure context) and modern browsers.

**Design guidance for any credential system** (NIST SP 800-63B is the reference):

1. **Prefer passkeys**, offer them first, and allow passwordless accounts.
2. If passwords exist: **minimum length (≥ 8, better 12–15+), no composition rules, no periodic forced changes, check against breached-password lists** (e.g., the k-anonymity "Pwned Passwords" range API), allow paste and password managers.
3. **MFA** (phishing-resistant where possible); avoid SMS as a primary factor.
4. **Account recovery is an attack surface** — treat it with the same rigor as sign-in (rate limits, notifications, no security questions).
5. **Credential stuffing defense**: rate limits per account and per IP, bot detection, lockout with care (lockout can be a DoS vector), breach-password checks, notifications of new sign-ins.
6. **Generic responses**: "invalid username or password," identical timing — don't reveal which accounts exist (user enumeration).

**The interview-grade sentence:** *"My first question is whether to hold credentials at all — delegating to an IdP like Entra External ID removes hashing, stuffing, recovery and MFA from my threat model. If the app must own them, ASP.NET Core Identity hashes with PBKDF2-HMAC-SHA512 at 100,000 iterations, handles lockout, security stamps and TOTP, and since .NET 10 supports WebAuthn passkeys natively with schema version 3 and an explicitly configured relying-party domain, though without attestation validation. I'd lead with passkeys, follow NIST 800-63B for any passwords — length over composition, breached-password checks, no forced rotation — and treat account recovery and enumeration as part of the attack surface."*

---

## Concept 62 — Cryptography in .NET

**The cardinal rule: don't invent cryptography, and don't assemble primitives when a higher-level API or a managed service does the job.** Most crypto bugs are composition bugs — reused nonces, unauthenticated encryption, wrong modes, timing leaks — not broken algorithms.

**Choose by goal:**

| Goal | Use | Avoid |
|---|---|---|
| Encrypt data at rest in your app | Envelope encryption with Key Vault (Concept 50); locally **`AesGcm`** (AEAD) with a unique 96-bit nonce per encryption | ECB; CBC without a MAC; reusing a nonce with the same key (catastrophic for GCM); home-made "encryption" |
| Protect round-tripped tokens (cookies, links) | **Data Protection** (Concept 59) | Raw AES with hand-made formats |
| Integrity / authenticity with a shared key | **`HMACSHA256.HashData(key, data)`**; compare with **`CryptographicOperations.FixedTimeEquals`** | Plain hashes as MACs; `==` / `SequenceEqual` comparisons (timing leaks) |
| Signatures | RSA-PSS or ECDSA (P-256) via `RSA`/`ECDsa`, or **Key Vault keys** via `CryptographyClient` so private keys stay in HSMs | RSA PKCS#1 v1.5 for new designs where PSS is available; exporting private keys |
| Random values (tokens, keys, nonces, IDs) | **`RandomNumberGenerator.GetBytes` / `GetInt32` / `GetHexString`** | `System.Random`; `Guid.NewGuid()` as a *secret* (it isn't designed to be unpredictable) |
| Password hashing | Identity's hasher, or **`Rfc2898DeriveBytes.Pbkdf2`** with SHA-512 and ≥100k iterations (or Argon2id via a vetted library) | SHA-256/MD5 of passwords; encryption of passwords |
| Hashing for non-secret integrity | SHA-256/SHA-512 (`SHA256.HashData`) | MD5, SHA-1 for anything security-relevant |
| TLS | Let the platform negotiate (TLS 1.2+, 1.3 preferred); don't pin `SslProtocols` to old versions | Custom certificate validation callbacks that return `true` |

```csharp
// AES-GCM done correctly (when you genuinely need local authenticated encryption).
public static byte[] Encrypt(ReadOnlySpan<byte> key, ReadOnlySpan<byte> plaintext, ReadOnlySpan<byte> associatedData)
{
    Span<byte> nonce = stackalloc byte[AesGcm.NonceByteSizes.MaxSize];      // 12 bytes
    RandomNumberGenerator.Fill(nonce);                                       // unique per encryption

    byte[] output = new byte[nonce.Length + plaintext.Length + AesGcm.TagByteSizes.MaxSize];
    Span<byte> cipher = output.AsSpan(nonce.Length, plaintext.Length);
    Span<byte> tag    = output.AsSpan(nonce.Length + plaintext.Length);

    using var aes = new AesGcm(key, AesGcm.TagByteSizes.MaxSize);           // 16-byte tag
    aes.Encrypt(nonce, plaintext, cipher, tag, associatedData);              // associatedData binds context (e.g., tenant ID)
    nonce.CopyTo(output);
    return output;                                                           // nonce || ciphertext || tag
}
```

(With random nonces, rotate the key long before ~2³² encryptions under one key; envelope encryption with a fresh DEK per object avoids the question entirely.)

**Post-quantum cryptography.** A sufficiently large quantum computer would break RSA and elliptic-curve cryptography (Shor's algorithm); symmetric crypto and hashes need only larger sizes (Grover). The near-term risk is **"harvest now, decrypt later"** — traffic recorded today decrypted in the future — which matters for data with a long confidentiality lifetime. NIST standardized **ML-KEM** (FIPS 203, key encapsulation), **ML-DSA** (FIPS 204, signatures) and **SLH-DSA** (FIPS 205, hash-based signatures) in August 2024. **.NET 10** ships `MLKem`, `MLDsa`, `SlhDsa` and (draft-based) `CompositeMLDsa` in `System.Security.Cryptography`, backed by the OS crypto provider (OpenSSL 3.5+ on Linux, recent Windows CNG) — check `IsSupported`. Practical architecture stance:

1. **Crypto agility now** — algorithms and key sizes configurable, versioned envelopes (`alg`/`kid` stored with ciphertext and signatures), key identifiers rather than hard-coded keys.
2. **Hybrid** (classical + PQC) during the transition — TLS stacks and browsers already negotiate hybrid key exchange (X25519 + ML-KEM); let the platform do it.
3. **Inventory** where RSA/ECC protects long-lived secrets (archives, signatures that must verify for decades).
4. Don't hand-roll PQC protocols; adopt them through TLS, platform services and vetted libraries as support lands.

**The interview-grade sentence:** *"In .NET I don't invent or hand-assemble cryptography: Data Protection for round-tripped tokens, Key Vault envelope encryption for data at rest, AesGcm with a unique nonce when I genuinely need local AEAD, HMAC with FixedTimeEquals for integrity, RSA-PSS or ECDSA — ideally Key Vault keys so private keys never leave the HSM — RandomNumberGenerator for anything secret, PBKDF2 or Argon2id for passwords, and platform-negotiated TLS 1.2 or 1.3. For post-quantum, .NET 10 ships ML-KEM, ML-DSA and SLH-DSA where the OS supports them, and my architecture response is crypto agility — versioned envelopes with algorithm and key IDs — hybrid schemes through the platform, and an inventory of long-lived RSA and ECC dependencies because of harvest-now-decrypt-later."*

---

## Concept 63 — Multi-tenant isolation and object-level authorization

Multi-tenant SaaS turns one class of bug into the most damaging one: **a request from tenant A that reads or modifies tenant B's data**. It's an elevation of privilege and an information disclosure at the same time, often affecting every customer.

**Rule 1 — the tenant comes from the token, never from the request.** `tid` (Entra), an `org_id`/`tenant` claim issued by your IdP, or a server-side mapping from the authenticated principal. A tenant ID in a header, route or body is a *request* to act in that tenant, which must be checked against the principal's memberships — never trusted as identity.

**Rule 2 — isolate at the data layer so mistakes don't leak.** Choose the isolation model by risk, scale and cost:

| Model | Isolation | Cost / ops | Notes |
|---|---|---|---|
| **Database (or account) per tenant** | Strongest; per-tenant CMK and restore possible | Highest; fleet management (elastic pools help) | Common for enterprise tiers and regulated customers |
| **Schema per tenant** | Strong-ish | Medium | Migration fan-out |
| **Shared tables with `TenantId`** | Logical only | Lowest | Must be enforced *everywhere* — use mechanisms below |
| **Cosmos DB: tenant as partition key** (or hierarchical partition keys) | Logical; queries naturally scoped | Low | Mind hot tenants (Module 27); container-per-tenant for large ones |
| **Hybrid / tiered** | Pool small tenants, isolate large or regulated ones | Balanced | The usual SaaS answer (Module 21's "it depends," made concrete) |

**Mechanisms that make shared models safe:**

```csharp
// EF Core global query filter: every query is tenant-scoped unless explicitly bypassed.
public sealed class OrdersDb(DbContextOptions<OrdersDb> options, ITenantContext tenant) : DbContext(options)
{
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder b) =>
        b.Entity<Order>().HasQueryFilter(o => o.TenantId == tenant.TenantId);   // tenant from validated claims
}
// Writes: set TenantId in SaveChanges from ITenantContext, ignoring any client-supplied value.
// IgnoreQueryFilters() only in audited admin code paths.
```

- **SQL row-level security** (`CREATE SECURITY POLICY` with a predicate on `SESSION_CONTEXT('TenantId')`, set per connection from the validated tenant) — enforcement *inside* the database, independent of application bugs.
- **Per-tenant encryption keys** (envelope encryption with a KEK per tenant) for bring-your-own-key or crypto-shredding on tenant deletion.
- **Storage**: container or prefix per tenant with user delegation SAS scoped to it, or ABAC conditions on tags (Concept 42).

**Rule 3 — everything else that holds tenant data is also a boundary**, and it's where leaks hide:

- **Caches** — cache keys must include the tenant (`orders:{tenantId}:{orderId}`); a shared cache keyed only by `orderId` serves tenant A's data to tenant B (Module 10's patterns, with a security twist). Output caching must vary by tenant/user.
- **Search indexes** — security filters per tenant (or index per tenant) in Azure AI Search.
- **Background jobs and messages** — carry the tenant in the message and **re-establish tenant context** in the consumer; never process with "no tenant" defaults.
- **Logs and telemetry** — tenant IDs on spans are useful (Module 28), customer *data* in logs is not; restrict who can query.
- **File exports, reports, signed URLs** — scoped and expiring.
- **AI features** — retrieval (RAG) must filter by tenant *before* content reaches the model; a model can't be trusted to respect tenant boundaries.

**Rule 4 — object-level authorization on every access** (Concept 56), returning 404 for objects outside the caller's scope, and using **non-enumerable identifiers** (GUIDs/ULIDs rather than sequential integers) as defense in depth — not as the control.

**Rule 5 — test isolation explicitly**: automated cross-tenant tests for every endpoint (tenant B's token on tenant A's resources must get 404), included in CI; periodic penetration testing focused on tenant isolation.

**The interview-grade sentence:** *"In multi-tenant systems the tenant always comes from the validated token, never from a header or body, and isolation is enforced at the data layer so application bugs don't leak: database-per-tenant for regulated or large customers, otherwise shared stores with EF Core global query filters and SQL row-level security driven by session context, tenant partition keys in Cosmos, and per-tenant keys where customers bring their own. I treat every other store as a tenant boundary too — cache keys, search filters, message consumers that re-establish tenant context, exports and RAG retrieval — keep object-level checks returning 404 on every access, and run automated cross-tenant tests for every endpoint in CI."*

---
# Part G — Platform and network security on Azure

## Concept 64 — Network isolation on Azure

Network controls are a **defense-in-depth layer** (Concept 3), most valuable for (1) making data stores unreachable from the internet, (2) limiting lateral movement, and (3) controlling egress so a compromised workload can't exfiltrate data to arbitrary hosts.

**The building blocks:**

| Building block | What it does | Notes |
|---|---|---|
| **Virtual network (VNet) and subnets** | Private address space; workloads with VNet integration (App Service, Functions Premium/Flex, Container Apps environments, AKS, VMs) | Hub-and-spoke or Virtual WAN topologies for larger estates |
| **Network security groups (NSGs)** | Stateful L4 allow/deny rules on subnets/NICs; **application security groups** for role-based rules | Deny-by-default between tiers; flow logs for visibility |
| **Private endpoints** (Private Link) | A private IP in your subnet for a specific PaaS resource (Key Vault, Storage, SQL, Cosmos, Service Bus…) | Combine with **public network access disabled** on the resource |
| **Private DNS zones** | Resolve the resource's public name (`kv-orders-prod.vault.azure.net`) to the private endpoint IP via `privatelink.*` zones | The #1 source of private endpoint failures — centralize zones in the hub, link to spokes, use DNS Private Resolver for hybrid |
| **Service endpoints** | Optimized route from a subnet to a PaaS service and firewall rule by subnet | Older, simpler; traffic still targets the public endpoint; private endpoints preferred for production |
| **Network security perimeter** (GA) | Logical boundary around PaaS resources, public inbound/outbound denied by default, with access logs | Complements private endpoints for PaaS-to-PaaS and exfiltration (Concept 48) |
| **Azure Firewall / NVA** | Central L3–L7 egress and east-west filtering, FQDN rules, TLS inspection (Premium), threat intelligence | Route spoke egress through it with UDRs |
| **NAT Gateway** | Explicit, predictable outbound SNAT | Needed now that new VNets default to private subnets |
| **Azure Bastion** | Browser-based RDP/SSH without public IPs on VMs | Replace jump boxes with public IPs |

**Explicit egress is now the default.** Since **March 31, 2026**, **new VNets default to private subnets**: VMs no longer get implicit "default outbound access" and need NAT Gateway, load balancer outbound rules, a firewall, or a public IP to reach the internet (existing VNets unchanged). This aligns the platform with a security goal you should want anyway — **egress control**: a compromised workload that can reach any internet host can exfiltrate data and call home; route egress through a firewall with FQDN allow-lists for the dependencies you actually need.

**Patterns:**

1. **Private-by-default PaaS**: every data store with a private endpoint and public access disabled; workloads with VNet integration; DNS in the hub.
2. **Segment by tier and sensitivity**: front-end, application and data subnets with NSG rules between them; separate spokes (and subscriptions) for workloads with different risk.
3. **Inspect east-west only where it pays** — firewall inspection between every microservice adds latency and cost; identity-based service-to-service authentication (tokens, mTLS) is usually the primary control, with NSGs for coarse segmentation.
4. **Treat private endpoints as necessary but not sufficient**: a workload *inside* the network with a stolen token can still read data; identity remains the decisive control (Concept 5).

**The interview-grade sentence:** *"On Azure I use the network as a layer behind identity: every data store gets a private endpoint with public access disabled and centrally managed privatelink DNS — the usual failure point — workloads join VNets, NSGs segment tiers deny-by-default, a network security perimeter covers PaaS-to-PaaS paths and exfiltration, and egress is explicit through NAT Gateway or Azure Firewall with FQDN allow-lists — which since March 31, 2026 is the default for new VNets anyway. And I'm clear that a private endpoint doesn't stop an attacker who's inside with a stolen token; identity at every hop is still the primary control."*

---

## Concept 65 — The edge: Front Door, WAF, DDoS and API Management

The edge is where internet traffic first meets your system — the place for **volume** defenses and **coarse policy**, so the application deals only with plausible requests.

| Component | Role | Security features |
|---|---|---|
| **Azure Front Door** (Standard/Premium) | Global anycast entry, TLS termination, routing, caching | **WAF** (managed rule sets based on OWASP Core Rule Set patterns, bot protection, custom rules, geo filtering, rate limiting); managed certificates; **Private Link origins** (Premium) so origins accept traffic only from Front Door |
| **Application Gateway** + WAF | Regional L7 load balancer | WAF with managed rule sets; mTLS to clients; end-to-end TLS |
| **Azure DDoS Protection** (Network or IP Protection) | Adaptive mitigation of volumetric and protocol attacks on public IPs | Basic infrastructure protection is always on; Network/IP Protection adds tuning, telemetry, rapid response and cost protection |
| **API Management (APIM)** | API gateway: policy enforcement point | `validate-jwt` / `validate-azure-ad-token`, `rate-limit-by-key`, `quota-by-key`, IP filters, request/response transformation (strip internal headers), mTLS to backends, subscription keys (as *identification*, not authentication), managed identity to backends, OpenAPI-based request validation |

**Design guidance:**

1. **Lock origins to the edge.** If attackers can reach your App Service or Container App directly, the WAF is optional for them. Use Front Door **Private Link origins**, or access restrictions to the `AzureFrontDoor.Backend` service tag **plus** validation of the `X-Azure-FDID` header (your Front Door profile's ID) — the service tag alone allows *any* Front Door customer.
2. **WAF in detection mode first, then prevention**, with tuned exclusions for false positives; managed rules + rate limits + bot protection; log to Log Analytics.
3. **APIM as a gateway, not as the only authorization.** Validating JWTs at APIM rejects junk early and centralizes coarse policy (audience, issuer, required scopes), but **services must still validate tokens and authorize** (complete mediation — Concept 8). Gateway-only security breaks the moment someone reaches a service directly or a gateway policy is misconfigured.
4. **Subscription keys are not authentication.** They identify and meter API consumers; combine them with OAuth for actual authentication.
5. **Rate limits by meaningful keys** — per client ID, per tenant, per user — not only per IP (NAT and mobile networks put many users behind one IP; attackers rotate IPs).
6. **Normalize and strip headers at the edge**: remove client-supplied `X-Forwarded-*`, internal routing headers and anything downstream trusts; set them yourself.
7. **TLS policy**: minimum TLS 1.2, modern cipher suites; HSTS.

**The interview-grade sentence:** *"At the edge I put Front Door with WAF — managed OWASP-based rules, bot protection, geo filtering and rate limits, run in detection before prevention — and DDoS Protection on public IPs, and I lock origins so they only accept Front Door traffic, via Private Link origins or the service tag plus the X-Azure-FDID header check, because the service tag alone admits any Front Door customer. API Management validates tokens and rate-limits per client and tenant early, but every service still validates and authorizes itself, subscription keys identify rather than authenticate, and client-supplied forwarding headers are stripped at the edge."*

---

## Concept 66 — PaaS hardening

Most cloud breaches are **misconfigurations**, not exploits — OWASP moved Security Misconfiguration to **#2 in 2025**, and its data found some misconfiguration in essentially every tested application. A PaaS hardening baseline:

| Area | Setting | Why |
|---|---|---|
| **Network exposure** | Public network access **disabled** (private endpoint/NSP); otherwise firewall to known ranges | Stolen credentials can't be used from the internet |
| **Local authentication** | Shared keys, SAS keys, SQL logins, access keys **disabled** (Concept 58) | Removes the bypass around identity controls |
| **Transport** | Minimum **TLS 1.2** (1.3 where supported); HTTPS only; FTP/FTPS disabled on App Service | Downgrade and plaintext exposure |
| **Anonymous access** | Storage `allowBlobPublicAccess: false` | Public containers are a classic leak |
| **Identity** | Managed identities; least-privilege data roles | Concept 37 |
| **Encryption** | At rest by default; CMK where required; infrastructure (double) encryption where required | Concept 50 |
| **Diagnostics** | Resource logs to Log Analytics (audit categories), retention per policy | Detection and investigation (Concept 70) |
| **Recovery** | Soft delete (Storage blobs/containers, Key Vault), versioning, point-in-time restore, immutability policies for audit and backup data | Ransomware and accidental deletion |
| **Resource locks / deployment stacks deny settings** | `CanNotDelete` on critical resources | Accidental or malicious deletion |
| **Management** | No public management ports (Bastion), App Service SCM site behind access restrictions | Control-plane attack surface |
| **Container hardening** | Non-root user, read-only root filesystem, distroless/chiseled images, no secrets in images, pinned digests, admission policies on AKS | Container escape and supply chain (Concept 68) |
| **Function/App settings** | Key Vault references instead of literal secrets | Concept 53 |

**Make it impossible to regress — Azure Policy.** Express the baseline as **policy assignments** (built-in initiatives such as the **Microsoft cloud security benchmark**, plus custom policies) at management-group scope, with effects:

- **Deny** — block non-compliant deployments (e.g., storage with public access, Key Vault without RBAC, SQL without Entra-only auth);
- **Audit** — report for remediation;
- **DeployIfNotExists / Modify** — auto-remediate (e.g., deploy diagnostic settings, add tags, enforce TLS settings).

Combine with **landing zones** (Azure Landing Zones / Cloud Adoption Framework): management groups, subscriptions per workload and environment, centrally applied policy, networking and logging — so new workloads start compliant.

**The interview-grade sentence:** *"Since misconfiguration is now OWASP's number two risk, I harden every PaaS resource to a baseline — public network access off behind private endpoints or a perimeter, local and shared-key auth disabled, TLS 1.2 minimum, no anonymous blob access, managed identities, diagnostic logs, soft delete, versioning and immutability for critical data, locks on critical resources, hardened non-root images with no secrets — and I make regressions impossible with Azure Policy at management-group scope using deny, audit and deploy-if-not-exists effects, starting from the Microsoft cloud security benchmark inside a landing zone."*

---

## Concept 67 — Encryption in transit and at rest

**In transit:**

1. **TLS on every hop, including internal ones.** "It's inside the VNet" isn't a reason for plaintext (Zero Trust, Concept 5). Azure PaaS endpoints are TLS-only; enforce minimum TLS 1.2. Internal HTTP between services in AKS or Container Apps should be TLS too — Container Apps can encrypt peer-to-peer traffic within an environment; on AKS a service mesh (Istio-based add-on) or ingress/certificates provide it.
2. **mTLS (mutual TLS)** authenticates *both* sides with certificates. It's strong for service-to-service and B2B integrations (banks, partners) and is what service meshes automate (SPIFFE-style workload identities with short-lived certificates). Costs: certificate issuance and rotation for every workload, TLS-terminating proxies that must forward or re-establish client certs. In Azure-native systems, **managed-identity tokens over TLS** usually provide equivalent caller authentication with less machinery; mTLS shines where tokens don't fit (partners without Entra, legacy protocols, regulatory demands) or a mesh is already in place.
3. **Database connections**: `Encrypt=True` (default in modern Microsoft.Data.SqlClient) with certificate validation — never `TrustServerCertificate=True` in production.
4. **Never disable certificate validation** in `HttpClientHandler.ServerCertificateCustomValidationCallback` "temporarily."

**At rest:**

| Layer | Default on Azure | Stronger options |
|---|---|---|
| Storage / disks / databases | Encrypted at rest with **platform-managed keys** (AES-256) — always on | **CMK** (Concept 50); **infrastructure (double) encryption** |
| Azure SQL | **TDE** on by default | TDE with CMK; **Always Encrypted** (client-side, column-level; the database never sees plaintext; secure enclaves allow some computations) |
| Cosmos DB | Encrypted by default | CMK; **client-side encryption** (Always Encrypted for Cosmos in the .NET SDK) |
| Application fields | — | Envelope encryption in code for the most sensitive fields; tokenization (store a token, keep the real value in a vault or with a provider — e.g., card data with a PSP) |
| Backups | Encrypted | Immutable (WORM) backups; separate subscription/tenant for backup vaults against ransomware |

**Data in use — confidential computing.** Hardware-based **trusted execution environments** (AMD SEV-SNP and Intel TDX confidential VMs, confidential containers on AKS/ACI, Intel SGX enclaves, confidential GPUs) protect data *while being processed*, even from the cloud operator and host admins, with **remote attestation** proving what code runs. Use cases: multi-party computation on sensitive data, regulated workloads requiring operator exclusion, key release only to attested workloads (Managed HSM / Azure Key Vault secure key release). It's a niche, high-assurance tool — know it exists and what it protects against.

**What encryption at rest does and doesn't protect against** (a classic senior distinction): it protects against **stolen disks, backups and media, and some provider-side access**; it does **not** protect against an attacker using your application or your credentials — the application decrypts transparently for them. Access control, not encryption at rest, is the control for the latter; client-side/field-level encryption and tokenization narrow the exposure further.

**The interview-grade sentence:** *"In transit I require TLS 1.2 or later on every hop including internal ones, with database encryption and certificate validation always on; mTLS where tokens don't fit — partners, legacy protocols, or an existing mesh issuing short-lived workload certificates — while managed-identity tokens over TLS cover most Azure service-to-service authentication. At rest everything on Azure is already encrypted with platform keys; I add CMK, Always Encrypted or field-level envelope encryption and tokenization for the most sensitive data, and confidential computing with attestation where even the operator must be excluded — being explicit that encryption at rest protects against stolen media and some provider access, not against someone using the application or its credentials."*

---

## Concept 68 — Software supply chain security

**Software Supply Chain Failures** entered the OWASP Top 10 at **#3 in 2025** (expanding "Vulnerable and Outdated Components"), and incidents such as SolarWinds (2020), Log4Shell (2021), the `xz` backdoor (2024) and repeated npm/PyPI/NuGet typosquatting and account-takeover campaigns show why: **your system includes everything you build from and with.**

The supply chain has four stages, each with threats and controls:

| Stage | Threats | Controls |
|---|---|---|
| **Dependencies** | Known vulnerabilities (CVEs); malicious packages (typosquatting, dependency confusion, compromised maintainer accounts); abandoned packages | **NuGet audit** (on by default; since .NET 9 audits **transitive** packages too), Dependabot / GitHub Advanced Security / Defender for DevOps; **Central Package Management** (`Directory.Packages.props`) for one version per package; **lock files** (`RestorePackagesWithLockFile`, `--locked-mode` in CI); **package source mapping** (`packageSourceMapping` in `nuget.config`) to prevent dependency confusion between internal and public feeds; prefer signed packages and reserved prefixes; minimize dependencies |
| **Source** | Compromised developer accounts; malicious commits; secrets committed | MFA/passkeys for SCM; branch protection and required reviews; signed commits; **secret scanning with push protection**; CODEOWNERS for sensitive paths |
| **Build** | Compromised build agents or pipeline definitions; poisoned caches; injected steps | Ephemeral, isolated runners; pipelines as code with review; least-privilege pipeline identities (Concept 69); pinned actions/tasks by commit SHA; **provenance** attestations (**SLSA** levels; GitHub artifact attestations) |
| **Artifacts and deployment** | Tampered images or packages; deploying unverified artifacts | **SBOMs** (SPDX or CycloneDX — the .NET SBOM tool, `dotnet` and container tooling generate them); **signing** (Notation/Notary v2 or Sigstore cosign for images; NuGet signing); **verify signatures at deploy/admission** (AKS image integrity / admission policies); image scanning in the registry (Defender for Containers); pinned image digests; minimal base images (chiseled .NET images) |

**.NET specifics worth naming:**

```xml
<!-- Directory.Build.props / project -->
<PropertyGroup>
  <NuGetAudit>true</NuGetAudit>                        <!-- default on -->
  <NuGetAuditMode>all</NuGetAuditMode>                 <!-- include transitive (default since .NET 9) -->
  <NuGetAuditLevel>moderate</NuGetAuditLevel>
  <WarningsAsErrors>$(WarningsAsErrors);NU1902;NU1903;NU1904</WarningsAsErrors>  <!-- fail on moderate/high/critical -->
  <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
  <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
</PropertyGroup>
```

```xml
<!-- nuget.config: internal packages may only come from the internal feed -->
<packageSourceMapping>
  <packageSource key="nuget.org"><package pattern="*" /></packageSource>
  <packageSource key="contoso-internal"><package pattern="Contoso.*" /></packageSource>
</packageSourceMapping>
```

**Responding to a new critical CVE** (the Log4Shell drill): the SBOMs and dependency inventory answer *"where are we affected?"* in minutes instead of days; CPM and lock files make the fix a single version bump; automated builds and deploys make it shippable the same day; WAF virtual patching buys time.

**The interview-grade sentence:** *"Supply chain failures are OWASP's number three for 2025, so I secure all four stages: dependencies with NuGet audit including transitive packages failing the build on moderate and above, Central Package Management, lock files in locked mode and package source mapping against dependency confusion; source with MFA, branch protection, required reviews and secret scanning with push protection; builds on ephemeral runners with SHA-pinned actions, least-privilege federated identities and SLSA-style provenance; and artifacts with SBOMs, signed images verified at admission, registry scanning and minimal chiseled base images — which is also what turns the next Log4Shell from a week of searching into a same-day version bump."*

---

## Concept 69 — CI/CD identity: the pipeline is production

A deployment pipeline holds the keys to production: it can change code, configuration and infrastructure. Compromising it is often easier than compromising production directly — and historically pipelines were full of long-lived secrets (service principal secrets in variables, publish profiles, storage keys).

**The modern pattern — federated, least-privilege, gated:**

1. **No stored cloud credentials.** GitHub Actions uses **OIDC federation** (`azure/login` with `client-id`, `tenant-id`, `subscription-id` and `permissions: id-token: write`) to a user-assigned managed identity or app registration with a **federated credential** (Concept 38); Azure DevOps uses **workload identity federation service connections**. Remove publish profiles and SP secrets; disable basic auth publishing credentials on App Service.
2. **Separate identities per environment and per purpose** — a dev deployer can't touch prod; an infrastructure identity (needs resource-group Contributor) is different from an app-deploy identity (needs only website contributor or ACR push + Container Apps revision update).
3. **Federate production only to protected contexts**: GitHub **environments** with required reviewers, deployment branch policies and wait timers; the federated subject is `repo:org/repo:environment:prod`. Pull requests — especially from forks — must never get production (or any privileged) identities; run them with read-only tokens and no secrets.
4. **Least privilege for the pipeline token itself**: GitHub's `GITHUB_TOKEN` with explicit minimal `permissions:`; pin third-party actions to full commit SHAs; restrict which actions are allowed.
5. **Separation of duties**: the people who write code aren't the only approvers for production; infrastructure changes go through reviewed IaC (`what-if`/plan output in the PR).
6. **Secrets that remain** (third-party API keys needed at deploy time) live in Key Vault and are fetched with the federated identity at runtime, or set as Key Vault references — not stored as pipeline variables.
7. **Audit**: who approved which deployment; pipeline identity sign-ins in Entra logs; alert on federated identity use outside expected repositories.
8. **Mandatory MFA consequence** (Concept 41): pipelines running as *users* now break on ARM writes — another reason to move to workload identities.

```yaml
# GitHub Actions: secretless deployment to production through a protected environment
permissions:
  id-token: write      # allow requesting the OIDC token
  contents: read
jobs:
  deploy:
    environment: prod  # required reviewers; federated subject = repo:contoso/orders:environment:prod
    runs-on: ubuntu-latest
    steps:
      - uses: azure/login@<full-commit-sha>
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID_PROD }}   # not a secret — an identifier
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID_PROD }}
      - run: az containerapp update --name orders-api --resource-group rg-orders-prod --image $IMAGE
```

**The interview-grade sentence:** *"I treat the pipeline as production: GitHub Actions or Azure DevOps authenticate to Azure through OIDC workload identity federation, so no service principal secrets or publish profiles exist; each environment and purpose has its own least-privilege identity; production identities federate only to a protected environment with required reviewers — never to pull requests or forks — third-party actions are pinned by commit SHA with a minimal GITHUB_TOKEN, infrastructure changes go through reviewed what-if output, and federated identity usage is audited and alerted on — which also keeps pipelines working under mandatory MFA."*

---

## Concept 70 — Posture, detection and response

Prevention will eventually fail somewhere; **detection and response** determine whether a failure becomes a breach. Three capabilities:

**1. Posture management — find weaknesses before attackers do.**

- **Microsoft Defender for Cloud**: **CSPM** (secure score, recommendations against the Microsoft cloud security benchmark, attack path analysis, cloud security explorer, agentless scanning) and **workload protection plans** (Defender for Servers, Containers, Storage — including malware scanning on upload — SQL, Key Vault, App Service, APIs, Resource Manager, DevOps).
- **Azure Policy** compliance (Concept 66) and **Entra** recommendations / secure score for identity.
- **Exposure management** views that connect identity, network and data findings into attack paths ("internet-exposed VM → managed identity with Key Vault access → secrets").

**2. Detection — see attacks in progress.**

- **Logs that matter for security** (beyond Module 28's diagnostic telemetry): Entra **sign-in logs** (users and service principals, including managed identities), **audit logs** (role changes, new credentials on apps, consent grants, CA policy changes), **Azure Activity Log** (control-plane writes), **resource logs** for data planes (Key Vault `AuditEvent`, Storage read/write logs for sensitive accounts, SQL auditing), **WAF and Front Door logs**, **NSG/VNet flow logs**, **Defender alerts**.
- **Microsoft Sentinel** (SIEM/SOAR on Log Analytics): analytics rules, UEBA, threat intelligence, incidents, playbooks (Logic Apps) for automated response.
- **High-signal detections** an architect should ask for: new credential added to a privileged app or service principal; consent granted to a new multi-tenant app with high-privilege permissions; break-glass account sign-in; PIM activation outside working hours; managed identity or service principal sign-in from unexpected IPs; mass secret reads from Key Vault; Storage access with shared keys after they were supposedly disabled; spikes in 401/403 (credential stuffing, BOLA probing); a federated credential used from an unexpected repository.

**3. Response — limit damage quickly.**

- **Runbooks** for the likely incidents: leaked secret (rotate → review usage → find the leak → eliminate the secret), compromised user (revoke sessions, reset, review mailbox rules and consents), compromised service principal (remove credentials, revoke role assignments, review sign-in and audit logs), compromised workload (isolate, snapshot for forensics, rotate everything it could access), tenant data exposure (contain, assess scope with audit logs, legal/regulatory notification — GDPR's 72-hour clock).
- **Pre-built levers**: the ability to rotate any secret in minutes (Concept 51), revoke all Data Protection keys (Concept 59), block an app via Conditional Access or disable a service principal, scale down or isolate a workload, put a WAF rule in front of a vulnerable endpoint.
- **Audit trails that hold up** (Module 28 Concept 40): complete, tamper-evident, retained — so you can answer "what did the attacker access?" Without them, notification scope becomes "everyone."
- **Practice**: tabletop exercises and game days; blameless postmortems feeding threat models (Concept 16).

**The interview-grade sentence:** *"Because prevention eventually fails, I design for posture, detection and response: Defender for Cloud and Azure Policy to find misconfigurations and attack paths before attackers do; Entra sign-in and audit logs, the Activity Log, Key Vault and data-plane resource logs, WAF and flow logs feeding Sentinel with high-signal detections like new credentials on privileged apps, risky consent grants, break-glass sign-ins and mass secret reads; and rehearsed runbooks with pre-built levers — rotate any secret in minutes, revoke sessions and Data Protection keys, disable a service principal — backed by tamper-evident audit trails, because without them the answer to 'what was accessed?' becomes 'assume everything.'"*

---
# Part H — Operating and deciding

## Concept 71 — Standards and requirements

Knowing which document answers which question is a senior signal:

| Standard / list | What it is | Use it for |
|---|---|---|
| **OWASP Top 10:2025** | Awareness list of the most critical web application risk categories (A01 Broken Access Control, A02 Security Misconfiguration, **A03 Software Supply Chain Failures**, A04 Cryptographic Failures, A05 Injection, A06 Insecure Design, A07 Authentication Failures, A08 Software or Data Integrity Failures, A09 Security Logging & Alerting Failures, **A10 Mishandling of Exceptional Conditions**) | Training, prioritizing review attention, talking to stakeholders — *not* a testable standard |
| **OWASP API Security Top 10 (2023)** | API1 BOLA, API2 Broken Authentication, API3 Broken Object Property Level Authorization, API4 Unrestricted Resource Consumption, API5 Broken Function Level Authorization, API6 Unrestricted Access to Sensitive Business Flows, API7 SSRF, API8 Security Misconfiguration, API9 Improper Inventory Management, API10 Unsafe Consumption of APIs | API design reviews — the authorization-heavy list is more relevant than the web Top 10 for backend services |
| **OWASP ASVS 5.0** (May 2025) | ~350 verifiable requirements in 17 chapters, three levels (L1 baseline → L3 high assurance) | Turning security into **acceptance criteria and test plans**; procurement and pen-test scoping |
| **OWASP Top 10 for LLM Applications (2026)** and **for Agentic Applications (Dec 2025)** | AI-specific risk lists | Reviews of AI features and agents (Concept 44) |
| **OWASP Cheat Sheet Series** | Practical, topic-specific guidance (authentication, session management, OAuth, JWT, SSRF, secrets…) | Implementation detail |
| **NIST SP 800-63B** | Digital identity / authentication assurance | Password and MFA policy |
| **NIST SP 800-207** | Zero Trust architecture | Architecture framing |
| **NIST Cybersecurity Framework 2.0** (2024) | Govern, Identify, Protect, Detect, Respond, Recover | Organizational security programs |
| **NIST SSDF (SP 800-218)** | Secure software development practices | Supply-chain and SDLC requirements (referenced by US government procurement) |
| **Microsoft cloud security benchmark** | Azure-specific control set, mapped to CIS/NIST/PCI | Azure Policy initiatives and Defender for Cloud recommendations |
| **CIS Benchmarks** | Hardening baselines for OSes, Kubernetes, cloud platforms | Configuration baselines |
| **Compliance frameworks** — **SOC 2**, **ISO/IEC 27001:2022**, **PCI DSS v4.0.1**, **HIPAA**, **GDPR**, EU **NIS2** and **DORA** (financial sector, applying since January 2025), the **EU Cyber Resilience Act** (phased obligations for products with digital elements) | Contractual and regulatory obligations | Scoping controls, evidence and audits |

How they connect in practice: the **threat model** (Part B) decides *what matters for this system*; **ASVS** turns it into *requirements at the right level*; the **Top 10 lists** keep reviews focused on the most common failures; the **cloud benchmark + Azure Policy** enforce platform configuration; **compliance frameworks** define what evidence you must produce. A frequent interview trap: "we're compliant, so we're secure" — compliance is a floor and a snapshot; security is the threat-driven, continuous practice.

PCI DSS deserves one architectural sentence: **minimize scope** — use a payment provider's hosted fields or redirect so card data never touches your systems (Concept 14's *eliminate/transfer*), which shrinks the cardholder data environment from your whole platform to almost nothing.

**The interview-grade sentence:** *"I use each standard for its job: the OWASP Top 10:2025 — now with supply chain failures at three and mishandled exceptional conditions at ten — and the 2023 API Top 10 for awareness and review focus, ASVS 5.0 to turn the threat model into verifiable acceptance criteria at the right level, NIST 800-63B for authentication policy, the Microsoft cloud security benchmark through Azure Policy for platform configuration, and SOC 2, ISO 27001, PCI DSS, GDPR, NIS2 or DORA for the evidence we owe — while reminding people that compliance is a floor and a snapshot, and that the best PCI architecture keeps card data out of our systems entirely."*

---

## Concept 72 — Anti-patterns

| Anti-pattern | Why it hurts | Instead |
|---|---|---|
| **Treating a valid token as authorization** | BOLA/IDOR — any authenticated user reads anyone's data | Function- and object-level authorization on every access (Concepts 6, 56) |
| **Disabling audience or issuer validation** | Accepts tokens meant for other APIs or from attacker-created tenants | Configure validation for the tokens you actually receive (Concepts 20, 36) |
| **Tenant ID from header, route or body** | Cross-tenant access by editing a field | Tenant from the validated token; data-layer isolation (Concept 63) |
| **Tokens in `localStorage`** | XSS steals reusable credentials | BFF with HttpOnly cookies (Concept 30) |
| **Forwarding the user's token to downstream services** | Defeats audience isolation; one leak opens everything | OBO or token exchange per hop (Concept 29) |
| **Implicit or password grants** | Front-channel tokens; no MFA; password anti-pattern | Code + PKCE; client credentials; device flow (Concept 24) |
| **Shared secrets everywhere** (connection strings with keys, client secrets, SAS) | Leak-prone, long-lived, rarely rotated | Managed identity, federation, keyless services with local auth disabled (Concepts 7, 58) |
| **One identity shared by many services**, or `Contributor` for apps | Huge blast radius; control-plane rights lead to keys | Identity per workload; data-plane roles at narrow scope (Concepts 4, 42) |
| **Key Vault as a shared enterprise secret dump** with access policies | Contributor can grant itself data access; one vault's compromise exposes everything | Per-app vaults, RBAC, per-secret scope (Concepts 46–47) |
| **Reading secrets per request** | Throttling, latency, hard dependency | Cache with refresh (Concept 49) |
| **`DefaultAzureCredential` unconstrained in production** | Unpredictable identity; slow failures | Deterministic credential or `AZURE_TOKEN_CREDENTIALS=prod` (Concept 58) |
| **"It's on the private network, so it's trusted"** | Lateral movement after one foothold | Zero Trust: authenticate every hop (Concept 5) |
| **Gateway-only authorization** | Anyone who reaches a service directly bypasses everything | Complete mediation in every service (Concepts 8, 65) |
| **Security by obscurity** (unguessable URLs, hidden admin endpoints, sequential IDs "nobody will try") | Discovery is cheap | Real authorization; obscurity only as depth |
| **Hand-rolled crypto or JWT parsing** | Composition bugs, validation bypasses | Vetted libraries and platform services (Concepts 20, 62) |
| **`catch { return true; }` in security checks** | Fails open | Fail closed (Concept 8) |
| **Secrets and tokens in logs and telemetry** | Broadly readable, long-retained leaks | Redaction and never logging headers/config (Module 28) |
| **Manual certificate renewal** | Outages at 200-, then 100-, then 47-day lifetimes | Automated issuance and renewal with alerts (Concept 52) |
| **Production identities federated to pull requests** | Any contributor deploys to prod | Protected environments with reviewers (Concepts 38, 69) |
| **Standing admin access and many Global Admins** | Phished admin = tenant compromise | PIM, < 5 Global Admins, phishing-resistant MFA, break-glass done right (Concept 41) |
| **User consent open to any app** | Consent phishing | Verified publishers, low-impact permissions, admin consent workflow (Concept 35) |
| **Threat model written once** | Drifts from reality | Lightweight, repeated, in the repo (Concept 16) |
| **Diagnostic logs as the audit trail** | Incomplete, mutable, expiring | Transactional, tamper-evident audit store (Concept 70) |
| **Trusting the LLM to enforce permissions** | Prompt injection makes it do whatever the attacker wrote | Authorize tools and data outside the model; least-privilege tools; approvals (Concept 44) |

---

## Concept 73 — Hidden costs and trade-offs

Every control has a price. Senior answers name it and justify it.

| Control | Hidden cost | How to keep it proportionate |
|---|---|---|
| **Private endpoints everywhere** | Private DNS complexity (hybrid resolution, zone sprawl), per-endpoint cost, deployment agents that must be inside the network, harder local development | Central DNS in the hub, landing zone automation, VNet-injected runners, dev environments with relaxed (but separate) settings |
| **Customer-managed keys** | Key Vault/HSM cost, purge protection, risk that a key mistake makes data unrecoverable, outage if the vault is unavailable | Only where regulation or customers require it (Concept 50) |
| **Managed HSM / Cloud HSM** | Significant hourly cost per pool/cluster; security-domain and HSM administration responsibilities | Premium HSM-backed keys usually suffice |
| **Short token lifetimes and introspection** | More token requests, IdP dependency on the hot path, latency | Default lifetimes plus CAE; introspection only for high-risk operations |
| **OBO on every hop** | Token acquisition latency on cold caches, IdP availability coupling | Distributed token caches, short chains, app-only tokens for service-level calls |
| **mTLS everywhere / service mesh** | Certificate lifecycle, sidecar CPU/memory and latency, debugging difficulty | Tokens over TLS for most service-to-service auth; mesh where it's already justified |
| **WAF in prevention mode** | False positives blocking legitimate traffic, tuning effort | Detection first, tuned exclusions, per-route policies |
| **Azure Firewall with TLS inspection** | Significant cost, added latency, certificate distribution, privacy concerns | FQDN egress allow-lists without inspection for most workloads |
| **PIM and approvals** | Friction during incidents; approver availability | Short activation for on-call with MFA but without approval; break-glass for true emergencies |
| **Self-hosted identity provider** | 24/7 availability of sign-in, key rotation, patching, security reviews, fraud handling | Managed IdP unless there's a compelling reason (Concept 43) |
| **Security logging at full fidelity** | Log Analytics ingestion costs (Module 28's economics), privacy obligations | High-signal security tables in Analytics, verbose resource logs in Basic/Auxiliary or archived; Sentinel data tiers |
| **Per-tenant databases** | Fleet operations, migrations fan-out, cost per small tenant | Tiered tenancy: pool small tenants, isolate large or regulated ones |
| **Strict Conditional Access** | Lockouts, user frustration, support tickets | Report-only rollout, exclusions reviewed, passkeys to make strong auth easy |
| **Secret rotation automation** | Engineering effort per credential type | Eliminate secrets instead wherever possible |

The meta-trade-off is **security vs. delivery speed**. The resolution isn't to choose one; it's to make secure choices *defaults* — templates, landing zones, policies, shared libraries (one `AddContosoAuthentication()` extension used by every service) — so teams get security without re-deciding it.

**The interview-grade sentence:** *"Every control has a cost I should be able to name: private endpoints bring DNS and deployment-path complexity, CMK and HSMs bring cost and the risk of unrecoverable data, short lifetimes and OBO couple the hot path to the identity provider, meshes and TLS inspection add latency and certificate overhead, WAF prevention brings false positives, PIM brings incident friction and self-hosted identity brings round-the-clock operations. I choose controls from the threat model, accept some risks explicitly, and make the secure choices cheap by turning them into defaults — templates, landing-zone policies and a shared authentication library — rather than per-team decisions."*

---

## Concept 74 — The security design review, and when not to

**A checklist to narrate** for any design (it doubles as a structure for system-design interview deep dives):

**Model and threats**
1. What are the assets, who are the adversaries, and where are the **trust boundaries** (Concepts 1–2)?
2. Is there a current **DFD and STRIDE threat list** with owners, ratings and accepted risks (Part B)?

**Identity and access**
3. How does **every** caller at **every** hop authenticate — users (OIDC, MFA/passkeys), workloads (managed/federated identity), agents (Concepts 6, 21, 37–38, 44)?
4. Which **token** is presented at each hop, with which **audience and scopes**, and how is it validated (Concepts 19–20, 26, 29, 55)?
5. Where is **authorization** enforced — function-level, object-level, property-level, tenant-level — and is it deny-by-default (Concepts 56, 63)?
6. What's the **blast radius** of each identity; are privileges least, scoped and time-bound (Concepts 4, 41–42)?

**Secrets, keys and data**
7. Which **secrets exist at all**, and why can't they be eliminated? Where do the rest live, who can read them, and how are they rotated (Concepts 7, 45–53)?
8. How is data **classified**, encrypted in transit and at rest, and where are customer-managed keys or field-level encryption justified (Concepts 50, 67)?
9. Are **tokens and secrets kept out of browsers, URLs and logs** (Concepts 30, 72)?

**Platform**
10. What is **publicly reachable**, and is everything else private with explicit egress (Concepts 64–66)?
11. Is the edge (WAF, DDoS, rate limits) protecting origins that can't be bypassed (Concept 65)?
12. Is the **supply chain and pipeline** secretless, least-privilege and verified (Concepts 68–69)?

**Detect and respond**
13. Which **security events** are logged and alerted on; is there a tamper-evident **audit trail** (Concept 70)?
14. Can we **rotate any secret, revoke any session, disable any identity** within minutes — and have we practiced it?
15. Which **residual risks** are accepted, by whom, until when?

**When not to — restraint that signals seniority:**

- **Don't build your own identity provider or authorization server** when a managed one fits.
- **Don't introduce a service mesh just for mTLS** when managed-identity tokens over TLS already authenticate every call.
- **Don't buy Managed HSM or Cloud HSM** without a compliance requirement that Key Vault Premium can't meet.
- **Don't put customer-managed keys on everything** — they add operational risk without protecting against application compromise.
- **Don't add a policy engine or ReBAC service** for an app whose authorization fits in a dozen ASP.NET Core policies.
- **Don't turn on TLS inspection of all egress** for a team that can't operate it; FQDN allow-lists get most of the value.
- **Don't run a 40-page threat modeling process** — 90 minutes, a DFD and a table, repeated, beat a document nobody updates.
- **Don't make strong security painful** — if passkeys and managed identities are available, use them, because controls that hurt get bypassed.

**The meta-rule, echoing Modules 13, 26, 27 and 28:** *every control is also a cost, a dependency and something that can fail.* The best security architecture eliminates what it can (secrets, stored data, exposed endpoints), verifies every request at every hop with short-lived, narrowly scoped credentials, contains what it can't prevent, detects what gets through — and documents the risks it consciously accepts.

**The interview-grade sentence:** *"I review security from the threat model down: assets, adversaries and trust boundaries; how every caller at every hop authenticates and which token, audience and scopes it presents; where function-, object-, property- and tenant-level authorization happen and whether they're deny-by-default; each identity's blast radius; which secrets exist and why they can't be eliminated; data protection, exposure and egress, the edge, the pipeline and supply chain; security logging and audit; and whether we can rotate, revoke and disable within minutes. And I show restraint — no self-built identity provider, mesh, HSM, blanket CMK or policy engine without a requirement that demands it — because every control is also a cost and a dependency, and the best design eliminates first, verifies every hop, contains, detects and documents what it accepts."*

---
# Putting it together

## Worked example 1 — "Design the security architecture for a multi-tenant B2B SaaS API on Azure"

*A B2B invoicing SaaS: ~400 customer organizations (from 10 to 20,000 users each), a React SPA, a public REST API for customers' integrations, ASP.NET Core services on Container Apps (`invoices-api`, `payments-worker`, `documents-api`), Azure SQL, Blob Storage for PDFs, Service Bus, a payment provider and an email provider. Some enterprise customers require data residency and bring-your-own-key. The CTO asks: "How do we make sure one customer can never see another's data — and what happens when a credential leaks?"*

Narrate in this order:

**1. Assets, adversaries, boundaries** (Concepts 1–2): invoices and customer financial data, bank details, PDFs, the ability to mark invoices paid or change payout accounts. Adversaries: opportunistic internet attackers, malicious or curious users of one customer probing another, compromised customer integration credentials, compromised dependencies, insiders. Boundaries: internet → edge; SPA → BFF; customer integrations → public API; services → data; **tenant ↔ tenant**; platform → payment/email providers; pipeline → production.

**2. Human identity** (Concepts 21, 33, 35, 39): the app is a **multi-tenant Entra app**; customer organizations consent, their users sign in with their own corporate identities (their MFA and Conditional Access apply — a strong selling point). Small customers without Entra sign in through **Entra External ID** (email OTP, passkeys, Google) in a separate external tenant; both issuers are configured, each validated strictly. App roles: `Invoices.Reader`, `Invoices.Approver`, `Tenant.Admin`, assigned by each customer's admins.

**3. The SPA** (Concept 30): **BFF** in ASP.NET Core — confidential client running code + PKCE, tokens in an encrypted distributed cache, the browser holds only a `__Host-` HttpOnly SameSite cookie; required custom header on API calls against CSRF; strict CSP.

**4. Integration API** (Concepts 23, 26, 22): customers' systems use **client credentials** against *their own* app registrations (or ours, multi-tenant), with **certificate** or **federated** credentials encouraged; tokens carry app roles like `Invoices.Read.All`. Per-client rate limits at APIM. Legacy API keys offered only for a deprecation period, hashed and scoped.

**5. Token validation and authorization** (Concepts 36, 55–56, 63):
- `invoices-api` validates audience (`api://invoices-api`), issuers (Entra multi-tenant pattern with `tid` matching, and the External ID issuer), algorithm, lifetime; **tenant allow-list** against our onboarding database in `OnTokenValidated`.
- Fallback policy requires authentication; endpoint policies require scope-or-app-role.
- **Tenant from `tid`** (or the External ID tenant claim mapped to our customer ID) — never from the request. EF Core **global query filters** plus **SQL row-level security** with `SESSION_CONTEXT`; resource-based handlers return 404 across tenants.
- Blob paths `/{tenantId}/{invoiceId}.pdf`, downloads via short-lived **user delegation SAS** scoped to one blob after an authorization check.
- Cache keys and Service Bus messages carry tenant context; consumers re-establish it.

**6. Service-to-service** (Concepts 29, 37, 57–58): each Container App has its own **user-assigned managed identity**; `documents-api` is called **on behalf of the user** (OBO) for user-initiated actions; `payments-worker` uses its own identity for asynchronous processing with `initiatedBy` carried as data. Service Bus, Storage, SQL and Key Vault accessed with Entra tokens; **local auth disabled** on all of them; Azure SQL **Entra-only**.

**7. Secrets and keys** (Concepts 45–53): what remains — the payment provider's API key and webhook signing secret, the email provider's key — lives in a **per-app Key Vault** (RBAC, `Key Vault Secrets User` per secret), private endpoint, purge protection, expiry dates with **SecretNearExpiry** rotation via the dual-key pattern; consumed via Container Apps Key Vault references. **BYOK** for enterprise tenants: tenant data in a dedicated database with **TDE using the customer's key** in their (or a per-tenant) Key Vault/Managed HSM; PDFs encrypted with per-tenant KEKs (envelope encryption) so offboarding can crypto-shred.

**8. Tenancy model** (Concept 63): pooled database for small tenants; **database-per-tenant in the customer's chosen region** for enterprise and BYOK tenants (data residency), routed by a tenant catalog.

**9. Platform** (Concepts 64–66, 68–69): Front Door Premium + WAF + Private Link origins; Container Apps environment with internal ingress; all PaaS private with an NSP; egress through Azure Firewall with FQDN allow-list (payment and email providers, Entra); Azure Policy initiative denying public network access and local auth; GitHub Actions with OIDC federation to per-environment identities; protected `prod` environment; NuGet audit, CPM, lock files, signed images.

**10. Detection and response** (Concept 70): Entra sign-in/audit logs, Key Vault audit, SQL auditing and WAF logs to Sentinel; detections for cross-tenant 404 spikes (BOLA probing), new credentials on our multi-tenant app, mass blob downloads; **audit trail** of invoice approvals and payout-account changes written transactionally (Module 28). Runbooks: rotate provider keys in minutes; revoke a customer integration's credential; disable a compromised tenant admin.

**11. Answer the CTO's two questions directly:** *Isolation*: tenant from the token, enforced at three layers (query filters, row-level security, resource-based checks) plus physically separate databases for the most sensitive customers, verified by automated cross-tenant tests on every endpoint. *Leaked credential*: there are almost none to leak — managed and federated identities everywhere; the few provider secrets rotate in minutes with dual keys, Key Vault audit shows who read them, and provider logs show misuse.

**12. Close with what you're not doing**: no self-hosted identity provider; no service mesh (tokens over TLS suffice); no Managed HSM for everyone (only BYOK tenants get dedicated keys); no TLS inspection of egress.

---

## Worked example 2 — "Threat-model the checkout flow with STRIDE"

*Use the DFD from Concept 10: browser → Front Door/WAF → Checkout BFF → Orders API → Orders DB; Orders API → payment provider; Orders API → Service Bus → fulfillment worker → warehouse system; Entra External ID for sign-in.*

**Step 1 — confirm the model** (10 minutes): boundaries A (internet/edge) and B (our platform/third parties); assets: card payment tokens (not card numbers — the provider's hosted fields keep PANs out of our systems), order totals, customer addresses, the ability to mark orders paid.

**Step 2 — STRIDE per element/interaction** (abridged; a real session yields ~30):

| ID | Element | STRIDE | Threat ("actor can… by… resulting in…") | Existing / proposed mitigation | Risk |
|---|---|---|---|---|---|
| T1 | Flow 1 (browser → edge) | S | An attacker can hijack a session by stealing the cookie via XSS, resulting in purchases as the victim | HttpOnly `__Host-` cookie; CSP with nonces; BFF keeps tokens server-side | Medium |
| T2 | Checkout BFF | T | A customer can change the price by editing the cart payload, resulting in underpayment | **Server-side recomputation** of all prices and totals from the catalog; client values ignored | High → mitigated |
| T3 | Orders API | E / I | A customer can read or cancel other customers' orders by changing `orderId`, resulting in disclosure and fraud | Resource-based authorization; tenant/customer-scoped queries; 404 on mismatch; integration tests | **Critical** |
| T4 | Flow 6 (payment provider callback) | S / T | An attacker can forge a "payment succeeded" webhook, resulting in goods shipped unpaid | Verify provider **HMAC signature** with constant-time comparison; timestamp tolerance against replay; idempotent handling by provider event ID; confirm status via provider API before fulfillment | High |
| T5 | Flow 6 | I | The payment provider API key leaks from config or logs, resulting in refunds to attacker accounts | Key in Key Vault, Secrets User for the Orders API identity only; redaction in telemetry; dual-key rotation; provider-side IP allow-list | High |
| T6 | Orders DB | I | An operator with DB admin rights browses customer addresses | Entra-only auth; PIM for DB admin; SQL auditing to Sentinel; consider Always Encrypted for address columns | Medium (accepted until Q2, owner: CISO) |
| T7 | Service Bus `orders-events` | T / S | A compromised internal component publishes forged `OrderPaid` events, resulting in fulfillment without payment | Per-service identities; **only Orders API has `Data Sender`**; local auth disabled; fulfillment worker re-validates payment state for high-value orders | Medium |
| T8 | Fulfillment worker | R | A warehouse operator disputes having received a shipment instruction | Signed, timestamped messages to the warehouse system; audit log of instructions and acknowledgements | Low |
| T9 | Edge | D | A bot floods checkout with requests, or scripts card-testing attacks through the payment step | Front Door WAF bot protection and rate limits; per-customer rate limiting in the BFF; provider-side fraud tools; CAPTCHA on anomalous patterns | High |
| T10 | Entra External ID sign-in | S | Credential stuffing takes over customer accounts | Passkeys and email OTP offered; breached-password checks; risk detection; rate limits; notifications on new sign-in | High |
| T11 | Orders API | E | Mass assignment lets a customer set `Status=Paid` in the create-order payload | Explicit input DTOs; no entity binding | High → mitigated |
| T12 | Logs (data store) | I | Tokens or addresses in Application Insights readable by many engineers | Redaction (Module 28 Concept 38); no header logging; table-level RBAC | Medium |

**Step 3 — respond**: T3, T4, T5, T9, T10 and T11 become backlog items with acceptance tests; T6 accepted with an expiry; T2 confirmed as already mitigated with a test added. **Elimination**: card numbers never enter our systems (provider hosted fields — PCI scope stays minimal).

**Step 4 — validate**: integration tests for T3/T11 (cross-customer access returns 404; `Status` in payload ignored), a webhook test with a bad signature and a replayed timestamp for T4, a WAF rule test for T9; the DFD and table committed under `/docs/threat-model/checkout.md`.

**What to say in the interview:** the session took 90 minutes, produced six fixes and one accepted risk, and two of the six (T3, T4) would have been invisible to scanners.

---

## Worked example 3 — "Our SPA keeps the access token in localStorage. Is that a problem?"

*A React SPA uses MSAL.js with tokens cached in `localStorage`, calling three APIs directly. Security found a stored-XSS bug in a comments widget last month.*

1. **Name the risk precisely** (Concept 30): any XSS — like last month's — can read `localStorage` and exfiltrate the **access tokens and the refresh token**; the attacker then uses them from their own machine, for up to 24 hours (Entra's SPA refresh token limit), long after the page is closed. Tokens in memory are better (no persistence) but XSS can still hook requests. The SPA calling three APIs directly also means three audiences of tokens are exposed.
2. **Short-term mitigations**: switch MSAL's cache to `sessionStorage` or memory (reduces persistence, not XSS impact); fix the XSS and add a **strict CSP** with nonces and Subresource Integrity on third-party scripts; reduce scopes requested by the SPA.
3. **Target architecture — BFF** (Concepts 21, 30, 57):
   - Add an ASP.NET Core **BFF** (it can also serve the SPA's static files) registered as a **confidential client** with a **managed identity FIC** — no secret.
   - The BFF runs **code + PKCE** with `AddMicrosoftIdentityWebApp`, stores tokens in an **encrypted distributed token cache**, issues a `__Host-` HttpOnly, Secure, SameSite=Strict session cookie.
   - The SPA calls `/bff/api/*`; the BFF (e.g., with **YARP**) attaches the right access token per downstream API (`IDownstreamApi` / token acquisition) and forwards.
   - **CSRF**: require a custom header (`X-CSRF: 1`) on all BFF API calls and reject cross-origin requests; antiforgery for any form posts.
   - **Logout**: clear the cookie and session tokens, redirect to Entra's end-session endpoint.
   - **.NET 10 note**: cookie auth returns 401 rather than redirecting for API endpoints, so the SPA handles 401 by navigating to `/bff/login`.
4. **Migration plan**: introduce the BFF for one API first; feature-flag the SPA's API base URL; move the other APIs; remove the SPA redirect URIs from the app registration when done (so code + PKCE can't be used by the browser any more); remove MSAL.js.
5. **Trade-offs to state**: an extra hop (a few milliseconds, same region), the BFF becomes a critical component (scale it, monitor it), session affinity not needed with the distributed cache; in exchange XSS can no longer steal reusable credentials and the APIs never see browser-held tokens.

---

## Worked example 4 — "Eliminate the secrets: migrate a system from keys to identities"

*An existing .NET system: three App Services and two Functions use connection strings with keys for Storage, Service Bus and Cosmos DB, SQL authentication for Azure SQL, a client secret for calling an internal API, and a publish profile in GitHub secrets. Secrets live in app settings; one was found in a public gist last quarter.*

1. **Inventory** (Concept 7): list every credential, where it's used, who can read it, when it was last rotated, and whether the target supports Entra auth. Output: 14 credentials; 12 can be eliminated; 2 (payment provider API key, legacy partner FTP password) can't.
2. **Identities**: create one **user-assigned managed identity per app per environment** (Concept 37); grant **data-plane roles** at narrow scope — `Storage Blob Data Contributor` on the app's container, `Azure Service Bus Data Sender`/`Receiver` on specific queues, Cosmos DB data-plane role assignment on the database, contained Entra users in Azure SQL with minimal permissions.
3. **Code changes** (Concept 58): replace connection strings with **endpoints + `ManagedIdentityCredential`** via `AddAzureClients`; Functions triggers use identity-based connections (`<Connection>__fullyQualifiedNamespace` and `__credential`/`__clientId` settings); SQL connection strings use `Authentication=Active Directory Managed Identity`. Developers use their own Entra identities against dev resources.
4. **Internal API**: replace the client secret with **managed identity as FIC** on the calling app's registration (`SignedAssertionFromManagedIdentity` in Microsoft.Identity.Web).
5. **Pipeline** (Concept 69): replace the publish profile with **OIDC federation** to per-environment deploy identities; disable basic-auth publishing on App Service.
6. **Remaining secrets**: move to a per-app **Key Vault** with RBAC, private endpoint, expiry dates and rotation; consume with Key Vault references.
7. **Close the door** (Concept 58): once traffic is verified on identities (watch storage/Service Bus/Cosmos metrics by authentication type; Storage logs show `AuthenticationType`), **disable shared-key access, local auth and SQL authentication**, then **rotate all old keys** anyway (they leaked once). Azure Policy **deny** assignments prevent regression.
8. **Sequence and safety**: one app at a time; dual-run (keys still valid but unused) for a few days; alert on any key-based access before disabling; remember role-assignment propagation delays during cutover.
9. **Result**: from 14 secrets to 2, both in Key Vault with rotation; the gist incident class is gone; blast radius per app shrinks from "everything in the resource group" to its own resources.

---

## Worked example 5 — "A client secret leaked on GitHub this morning. What do you do?"

*Secret scanning alerted at 08:40: a commit in a public repository contains the client secret of `reporting-app`, an Entra app with application permission `Orders.Read.All` on the orders API and `Sites.Read.All` on Microsoft Graph. The commit was pushed at 08:12.*

**Contain (first 30 minutes):**

1. **Assume it's been used** — automated scanners harvest public secrets within minutes.
2. **Revoke**: delete the leaked secret from the app registration (if the app has another valid credential, workloads keep running; if not, create a new credential first — or better, switch to managed identity as FIC right now — and accept a brief outage over continued exposure).
3. **Contain sessions**: tokens already issued with the secret remain valid until expiry (~60–90 minutes); if misuse is suspected, **disable the service principal** temporarily or block it with Conditional Access for workload identities (if licensed), accepting downtime for the reporting feature.
4. **Remove the secret from the repository** (and history — though it's already public; removal doesn't un-leak it). Treat the commit author's machine as possibly compromised if the circumstances are unclear.

**Investigate (hours):**

5. **Entra service principal sign-in logs** for the app since 08:12: sign-ins from unexpected IPs, ASNs or locations.
6. **What could it reach?** `Orders.Read.All` (all orders, all tenants?) and `Sites.Read.All` (every SharePoint site in the tenant) — define the potential scope; then the **actual** scope from orders API logs (requests by that client ID / `azp`), Graph activity logs and SharePoint audit logs.
7. **Persistence check**: did the attacker use the access to add credentials or permissions anywhere (audit logs for app credential additions, consent grants, role assignments)?
8. **Decide on notification** with legal: if customer data was accessed, GDPR/contractual clocks may be running (72 hours to the supervisory authority under GDPR when there's risk to individuals).

**Fix the root cause (days):**

9. **How did it leak?** A developer copied production configuration for local debugging. Systemic causes: production secrets were readable by developers, local development needed them, and there was no push protection.
10. **Eliminate the secret**: `reporting-app` runs on Azure → **managed identity as FIC** (or a managed identity directly if it doesn't need an app registration); no secret exists to leak.
11. **Shrink the blast radius**: replace `Sites.Read.All` with **`Sites.Selected`** for the two sites it needs; scope `Orders.Read.All` to what reporting needs, or split a reporting read model.
12. **Guardrails**: **push protection** for secret scanning on all repositories; app management policy **blocking new client secrets** tenant-wide (exceptions by review); developers use their own identities against dev environments; alerts on new credentials for privileged apps.
13. **Postmortem**: blameless, feeding the threat model (Concept 16) and the runbook — next time containment should take 10 minutes, and ideally there's no secret to leak at all.

---

## Common interview questions, with model answers

**"What's the difference between OAuth 2.0 and OpenID Connect?"** OAuth is delegated *authorization*: it gives a client a scoped, short-lived access token for an API without sharing the user's password. OIDC adds *authentication* on top: with the `openid` scope the client also gets an ID token addressed to itself, describing who signed in and how, plus discovery and a UserInfo endpoint. Access tokens are for APIs; ID tokens are for the client and must never be sent to APIs (Concepts 17, 19, 27).

**"Walk me through the authorization code flow with PKCE. Why PKCE?"** The client creates a random verifier and sends its SHA-256 challenge with state and nonce to `/authorize`; the user authenticates; the AS redirects back with a one-time code and its issuer; the client checks state and issuer and redeems the code at `/token` over the back channel with the verifier and its client authentication; it validates the ID token and nonce. PKCE makes an intercepted or injected code useless because only the original client knows the verifier — which is why OAuth 2.1 requires it for every client (Concept 21).

**"Why is the implicit flow deprecated?"** It returned tokens in the URL fragment — the front channel — exposing them to browser history, referrers, extensions and injected scripts, with no client binding and no refresh tokens. Code + PKCE delivers tokens over the back channel and works for SPAs now that CORS is universal (Concept 24).

**"Where should a SPA store tokens?"** Ideally nowhere: use a Backend-for-Frontend that is a confidential client, keeps tokens server-side and gives the browser an HttpOnly, Secure, SameSite cookie, with CSRF defenses. If the SPA must hold tokens, keep them in memory, use refresh token rotation, minimal scopes and a strict CSP — accepting that XSS can still steal them (Concept 30).

**"How do you validate a JWT?"** Signature against the issuer's JWKS by `kid` with rotation; algorithm allow-list (no `none`, no algorithm confusion); exact issuer (tenant-aware); audience is this API; `exp`/`nbf` with small skew; token type; then scopes or roles for the endpoint. Never decode without validating, never disable audience or issuer checks (Concept 20).

**"Delegated or application permissions?"** Delegated when a user is present — the app acts within the intersection of its scopes and the user's rights, carried as `scp`. Application permissions when there's no user — the app acts as itself tenant-wide, carried as `roles`, always admin-consented, so grant them sparingly. APIs should distinguish the two in policy (Concept 35).

**"How does service A call service B on behalf of a user?"** Not by forwarding A's incoming token — its audience is A. A uses on-behalf-of (or RFC 8693 token exchange) to get a downscoped token for B that still represents the user; B validates a token issued for itself. Service-level work uses A's own app identity. Async work carries the user's identity as data (Concept 29).

**"How should our services authenticate to Azure SQL, Storage and Service Bus?"** With managed identities and data-plane roles at narrow scope — endpoints plus `ManagedIdentityCredential`, no connection-string keys — and then disable local/shared-key auth and SQL authentication so keys can't be used even if found, enforced by Azure Policy (Concepts 37, 42, 58).

**"Is DefaultAzureCredential OK in production?"** Microsoft positions it for early development: its probing chain makes the chosen identity depend on the environment and slows failures. In production use `ManagedIdentityCredential` or `WorkloadIdentityCredential` explicitly, or at least set `AZURE_TOKEN_CREDENTIALS=prod` and require it (Concept 58).

**"What's workload identity federation and why does it matter?"** An Entra app or managed identity trusts tokens from an external OIDC issuer — GitHub Actions, Kubernetes, another cloud, or a managed identity — for an exact subject, so the workload gets Entra tokens without any stored secret. It removes the most leak-prone credentials in CI/CD and cross-tenant apps; the subject is the security boundary, so production must federate only to protected environments (Concept 38).

**"Key Vault: RBAC or access policies?"** RBAC — it's the default for new vaults since API version 2026-02-01, and with access policies anyone with Contributor on the vault can grant themselves data access. Use per-app vaults, `Key Vault Secrets User` scoped to the secrets an app needs, PIM for administrators; update IaC before older control-plane API versions retire in February 2027 (Concept 47).

**"How do you rotate secrets without downtime?"** First eliminate them; otherwise the dual-credential pattern — two valid credentials, switch consumers to the new one, verify, revoke the old — automated from Key Vault's `SecretNearExpiry` Event Grid event, with consumers that reload configuration (Concept 51).

**"Do we need customer-managed keys?"** Data is already encrypted at rest with platform keys. CMK adds a revocation kill switch, your own rotation and audit, and separation of duties — useful for regulation or per-tenant BYOK — at the cost of operational risk and purge protection. It doesn't protect against a compromised application (Concept 50).

**"How would you threat-model this design?"** Draw a DFD with trust boundaries and assets; walk STRIDE per element and per boundary-crossing interaction; rate by likelihood × impact or a bug bar; respond with mitigate, eliminate, transfer or accept; turn results into backlog items and tests; keep it in the repo and repeat on significant change (Concepts 9–16).

**"What does Zero Trust mean concretely here?"** No trust from network location: every request authenticated and authorized with identity and context, least privilege just in time, assume breach — so service-to-service calls carry validated tokens, workloads use managed identities, admin access is via PIM, and private networking is an extra layer, not the basis of trust (Concept 5).

**"How do you prevent one tenant from seeing another's data?"** Tenant from the validated token only; data-layer isolation — query filters and row-level security for pooled tenants, separate databases for sensitive ones; tenant in cache keys, search filters, messages and RAG retrieval; object-level checks returning 404; automated cross-tenant tests in CI (Concept 63).

**"A secret leaked — what do you do?"** Assume use; revoke and replace immediately; contain sessions if needed; investigate sign-in and resource logs for the exposure window and for persistence; scope impact and notification; then fix the root cause, ideally by eliminating the secret, and add guardrails like push protection and secret-blocking policies (Worked example 5).

**"How do you secure a CI/CD pipeline deploying to Azure?"** OIDC workload identity federation instead of stored service principal secrets; per-environment least-privilege identities; production federated only to protected environments with reviewers; pinned actions; minimal pipeline token permissions; reviewed IaC; supply-chain controls on dependencies and artifacts (Concepts 68–69).

**"What changed in OWASP Top 10 2025?"** Broken Access Control stays #1 and now includes SSRF; Security Misconfiguration rose to #2; Software Supply Chain Failures is new at #3; Cryptographic Failures, Injection and Insecure Design moved to #4–6; Mishandling of Exceptional Conditions is new at #10 (Concept 71).

**"How do you secure an AI agent that can call tools?"** Give it its own identity, decide delegated vs autonomous authority explicitly, least-privilege audience-bound tokens per tool with no passthrough, human approval for irreversible actions, treat all model input as untrusted because prompt injection has no complete fix, and audit every action with the agent and delegating user (Concept 44).

---

## Common mistakes vs. senior signals

| Topic | Common mistake | Senior signal |
|---|---|---|
| Framing | "We'll use OAuth and Key Vault, so it's secure" | Names assets, adversaries, trust boundaries and the residual risks being accepted |
| AuthN vs AuthZ | Valid token ⇒ request allowed | Function-, object-, property- and tenant-level authorization on every access, deny by default |
| Tokens | Forwards the user's token to every downstream service | OBO/token exchange per hop with correct audiences and downscoping |
| Browser apps | Tokens in localStorage "because MSAL does it" | BFF with HttpOnly cookies, CSRF header, strict CSP |
| JWT validation | Disables audience/issuer validation to fix a 401 | Explains token versions and multi-tenant issuer validation; allow-lists tenants |
| Flows | Suggests ROPC for test automation or implicit for SPAs | Code + PKCE, client credentials, device flow with CA restrictions — and knows why the others are gone |
| Secrets | "Store it in Key Vault" as the first answer | "Can we not have it?" — managed identity, federation, keyless services with local auth disabled |
| Azure credentials | `DefaultAzureCredential` everywhere | Deterministic credentials in production, `AZURE_TOKEN_CREDENTIALS`, dev identities for developers |
| Permissions | Contributor for apps; one shared identity | Data-plane roles at narrow scope, identity per workload, blast-radius reasoning |
| Key Vault | One shared vault with access policies | Per-app vaults, RBAC (default since 2026-02-01), private endpoint/NSP, purge protection, caching |
| Certificates | Annual manual renewal | Knows 200/100/47-day schedule; automated issuance, renewal, reload and alerts |
| Network | "It's in the VNet, so it's fine" | Network as a layer; identity at every hop; explicit egress; origins locked to the edge |
| Multi-tenancy | Tenant ID from a header | Tenant from token; RLS/query filters/database-per-tenant; cross-tenant tests |
| Threat modeling | A one-time document, or none | 90-minute STRIDE sessions per change, risk register with owners, tests as evidence |
| Privileged access | Standing Global Admins, users running automation | PIM, break-glass with FIDO2, workload identities (mandatory MFA-ready) |
| Pipelines | Service principal secrets in CI variables | OIDC federation to protected environments, pinned actions, provenance |
| Supply chain | "We run Dependabot" | NuGet audit incl. transitive, CPM, lock files, source mapping, SBOMs, signed images |
| Detection | "We have logs" | High-signal detections, tamper-evident audit trail, rehearsed rotation/revocation runbooks |
| Restraint | Adds a mesh, HSMs, CMK and a policy engine by default | Chooses controls from the threat model and names their costs |

---

## Practice exercises

1. **JWT autopsy.** Take an Entra access token from a test tenant (jwt.ms decodes locally). Identify the version, issuer, audience form, `oid`/`tid`, `scp` vs `roles`, lifetime. Then write the exact `TokenValidationParameters` that would accept it — and list three misconfigurations that would accept tokens they shouldn't.
2. **PKCE by hand.** In C#, generate a code verifier with `RandomNumberGenerator`, compute the S256 challenge, and run an authorization code + PKCE flow against a test Entra app with `curl` for the token exchange. Explain in writing what an attacker with only the code can and can't do.
3. **Threat-model a feature you own.** Draw the DFD (Mermaid), mark trust boundaries, walk STRIDE per element, produce a risk register with at least ten threats, and convert the top three into tests.
4. **Build a BFF.** ASP.NET Core BFF with `AddMicrosoftIdentityWebApp`, YARP forwarding to an API with `IDownstreamApi`-acquired tokens, distributed token cache, `__Host-` cookie, custom-header CSRF check. Verify in the browser's dev tools that no token is ever visible to JavaScript.
5. **Deny-by-default authorization.** Add a fallback policy to an existing API; enumerate every endpoint that breaks; decide explicitly which get `AllowAnonymous`. Add resource-based authorization to one object endpoint and a cross-tenant integration test.
6. **Go keyless.** Convert a sample app using Storage and Service Bus connection strings to managed identity (user-assigned) with data-plane roles; then disable shared-key access and local auth; write the Azure Policy assignment that prevents regression.
7. **Federate a pipeline.** Configure GitHub Actions → Entra OIDC federation with a `environment:prod` subject and required reviewers; prove that a pull-request workflow cannot obtain the production identity.
8. **Rotation drill.** Create a Key Vault secret with an expiry; wire `SecretNearExpiry` to a Function that rotates a Storage account key using the dual-key pattern (in a sandbox where shared keys are still enabled); time how long a full emergency rotation takes.
9. **Multi-tenant hardening.** Implement EF Core global query filters and SQL row-level security keyed by `SESSION_CONTEXT`; write a test that deliberately bypasses the application filter and proves RLS still blocks cross-tenant reads.
10. **Leak tabletop.** Run Worked example 5 as a 45-minute tabletop with your team: who does what, which logs answer which questions, how long each step takes. Write the runbook.
11. **Certificate inventory.** List every certificate your system depends on (TLS, client assertions, SAML signing, mTLS) with expiry, renewal mechanism and consumer reload behavior; identify which would break first under 100-day lifetimes.
12. **Agent review.** For an AI feature (or a hypothetical agent with email and calendar tools), apply STRIDE plus the OWASP agentic list; define its identity, the authority model, tool scopes, approval points and audit fields.

---
# Free resources and learning material

All free to read or use. Specifications and vendor docs change — check dates on anything that states versions, limits or preview status.

### Start here (the five most valuable)

- [OAuth 2.0 Security Best Current Practice — RFC 9700](https://datatracker.ietf.org/doc/html/rfc9700) — the current consensus on OAuth security; read Section 2 first.
- [PortSwigger Web Security Academy](https://portswigger.net/web-security) — free, hands-on labs; do the [OAuth](https://portswigger.net/web-security/oauth), [JWT](https://portswigger.net/web-security/jwt), [access control](https://portswigger.net/web-security/access-control) and [SSRF](https://portswigger.net/web-security/ssrf) tracks.
- [OAuth 2.0 Simplified (oauth.com)](https://www.oauth.com/) — Aaron Parecki's free online book, the clearest end-to-end explanation of the flows.
- [Microsoft identity platform documentation](https://learn.microsoft.com/entra/identity-platform/v2-overview) — the Entra side of everything in Parts C–D.
- [Threat Modeling Manifesto](https://www.threatmodelingmanifesto.org/) — one page that resets how teams think about threat modeling.

### Foundations and architecture principles

- [Saltzer & Schroeder, "The Protection of Information in Computer Systems" (1975)](https://web.mit.edu/Saltzer/www/publications/protection/) — the origin of least privilege, fail-safe defaults, complete mediation, economy of mechanism.
- [NIST SP 800-207 — Zero Trust Architecture](https://csrc.nist.gov/pubs/sp/800/207/final)
- [Google — BeyondCorp: A New Approach to Enterprise Security](https://research.google/pubs/beyondcorp-a-new-approach-to-enterprise-security/)
- [Microsoft Zero Trust guidance center](https://learn.microsoft.com/security/zero-trust/zero-trust-overview)
- [Azure Well-Architected Framework — Security pillar](https://learn.microsoft.com/azure/well-architected/security/)
- [NIST Cybersecurity Framework 2.0](https://www.nist.gov/cyberframework)
- [Microsoft Secure Future Initiative](https://www.microsoft.com/trust-center/security/secure-future-initiative) — context for Microsoft's recent secure-by-default changes (mandatory MFA, RBAC defaults).

### Threat modeling

- [Microsoft Threat Modeling Tool](https://learn.microsoft.com/azure/security/develop/threat-modeling-tool) and [its STRIDE threat categories](https://learn.microsoft.com/azure/security/develop/threat-modeling-tool-threats)
- [OWASP Threat Modeling Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Threat_Modeling_Cheat_Sheet.html)
- [OWASP Threat Dragon](https://owasp.org/www-project-threat-dragon/) — free cross-platform modeling tool, models stored as JSON in the repo.
- [Threagile](https://threagile.io/) — threat modeling as YAML, runnable in CI.
- [OWASP pytm](https://owasp.org/www-project-pytm/) — threat models in Python.
- [Shostack + Associates — threat modeling resources](https://shostack.org/resources/threat-modeling) — articles and the four-question framework from Adam Shostack.
- [OWASP Cornucopia](https://owasp.org/www-project-cornucopia/) — card game for threat generation sessions.
- [LINDDUN privacy threat modeling](https://linddun.org/)
- [MITRE ATT&CK](https://attack.mitre.org/) and [MITRE CAPEC](https://capec.mitre.org/)
- [Microsoft Security Development Lifecycle (SDL)](https://www.microsoft.com/securityengineering/sdl)

### OAuth 2.0, OpenID Connect and related specifications

- [OAuth 2.0 — RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749) and [Bearer Token Usage — RFC 6750](https://datatracker.ietf.org/doc/html/rfc6750)
- [The OAuth 2.1 Authorization Framework (draft)](https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/)
- [PKCE — RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636)
- [JSON Web Token — RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519) and [JWT Profile for Access Tokens — RFC 9068](https://datatracker.ietf.org/doc/html/rfc9068)
- [Device Authorization Grant — RFC 8628](https://datatracker.ietf.org/doc/html/rfc8628)
- [Token Exchange — RFC 8693](https://datatracker.ietf.org/doc/html/rfc8693)
- [Pushed Authorization Requests — RFC 9126](https://datatracker.ietf.org/doc/html/rfc9126)
- [Issuer Identification (mix-up defense) — RFC 9207](https://datatracker.ietf.org/doc/html/rfc9207)
- [Rich Authorization Requests — RFC 9396](https://datatracker.ietf.org/doc/html/rfc9396)
- [DPoP — RFC 9449](https://datatracker.ietf.org/doc/html/rfc9449)
- [Protected Resource Metadata — RFC 9728](https://datatracker.ietf.org/doc/html/rfc9728)
- [OAuth 2.0 for Browser-Based Applications (draft BCP)](https://datatracker.ietf.org/doc/draft-ietf-oauth-browser-based-apps/) — the BFF, token-mediating backend and browser-client patterns.
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html) and [Back-Channel Logout](https://openid.net/specs/openid-connect-backchannel-1_0.html)
- [FAPI 2.0 Security Profile (final)](https://openid.net/specs/fapi-security-profile-2_0-final.html)
- [oauth.net — OAuth 2.0 index](https://oauth.net/2/) and [OAuth 2.1 summary](https://oauth.net/2.1/)
- [Model Context Protocol — Authorization](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization) — OAuth 2.1 profiled for AI agents and tools.
- [OAuth 2.0 and OpenID Connect (in plain English) — Nate Barbettini, video](https://www.youtube.com/watch?v=996OiexHze0)
- [Pragmatic Web Security (Philippe De Ryck)](https://pragmaticwebsecurity.com/) — free articles and talks on OAuth, SPAs and BFFs.
- [jwt.ms](https://jwt.ms/) — Microsoft's client-side token decoder (decodes locally; still, only paste test tokens).

### Microsoft Entra ID

- [Authorization code flow on the Microsoft identity platform](https://learn.microsoft.com/entra/identity-platform/v2-oauth2-auth-code-flow)
- [Client credentials flow](https://learn.microsoft.com/entra/identity-platform/v2-oauth2-client-creds-grant-flow)
- [On-behalf-of flow](https://learn.microsoft.com/entra/identity-platform/v2-oauth2-on-behalf-of-flow)
- [Access tokens](https://learn.microsoft.com/entra/identity-platform/access-tokens), [ID tokens](https://learn.microsoft.com/entra/identity-platform/id-tokens) and [Secure applications by validating claims](https://learn.microsoft.com/entra/identity-platform/claims-validation)
- [Permissions and consent overview](https://learn.microsoft.com/entra/identity-platform/permissions-consent-overview)
- [Application and service principal objects](https://learn.microsoft.com/entra/identity-platform/app-objects-and-service-principals)
- [Microsoft identity platform integration checklist](https://learn.microsoft.com/entra/identity-platform/identity-platform-integration-checklist)
- [Managed identities for Azure resources](https://learn.microsoft.com/entra/identity/managed-identities-azure-resources/overview)
- [Workload identity federation](https://learn.microsoft.com/entra/workload-id/workload-identity-federation) and [Configure an app to trust a managed identity](https://learn.microsoft.com/entra/workload-id/workload-identity-federation-config-app-trust-managed-identity)
- [Conditional Access overview](https://learn.microsoft.com/entra/identity/conditional-access/overview) and [Authentication strengths](https://learn.microsoft.com/entra/identity/authentication/concept-authentication-strengths)
- [Enable passkeys (FIDO2) in Entra ID](https://learn.microsoft.com/entra/identity/authentication/how-to-enable-passkey-fido2)
- [Continuous access evaluation](https://learn.microsoft.com/entra/identity/conditional-access/concept-continuous-access-evaluation)
- [Privileged Identity Management](https://learn.microsoft.com/entra/id-governance/privileged-identity-management/pim-configure)
- [Manage emergency access (break-glass) accounts](https://learn.microsoft.com/entra/identity/role-based-access-control/security-emergency-access)
- [Mandatory multifactor authentication for Azure](https://learn.microsoft.com/entra/identity/authentication/concept-mandatory-multifactor-authentication)
- [Microsoft Entra External ID overview](https://learn.microsoft.com/entra/external-id/external-identities-overview) and [Azure AD B2C FAQ (end of sale and support dates)](https://learn.microsoft.com/azure/active-directory-b2c/faq)
- [What is Microsoft Entra Agent ID](https://learn.microsoft.com/entra/agent-id/what-is-microsoft-entra-agent-id)
- [Multitenant identity considerations (Azure Architecture Center)](https://learn.microsoft.com/azure/architecture/guide/multitenant/considerations/identity) and [Tenancy models](https://learn.microsoft.com/azure/architecture/guide/multitenant/considerations/tenancy-models)

### Azure Key Vault, HSMs and certificates

- [Key Vault overview](https://learn.microsoft.com/azure/key-vault/general/overview) and [best practices](https://learn.microsoft.com/azure/key-vault/general/best-practices)
- [Azure RBAC for Key Vault](https://learn.microsoft.com/azure/key-vault/general/rbac-guide) and [Prepare for API version 2026-02-01 (RBAC default)](https://learn.microsoft.com/azure/key-vault/general/access-control-default)
- [Soft delete and purge protection](https://learn.microsoft.com/azure/key-vault/general/soft-delete-overview)
- [Service limits (throttling)](https://learn.microsoft.com/azure/key-vault/general/service-limits)
- [Configure key auto-rotation](https://learn.microsoft.com/azure/key-vault/keys/how-to-configure-key-rotation) and [Secret rotation tutorial with two sets of credentials](https://learn.microsoft.com/azure/key-vault/secrets/tutorial-rotation-dual)
- [Key Vault events with Event Grid](https://learn.microsoft.com/azure/key-vault/general/event-grid-overview)
- [Key Vault private endpoints](https://learn.microsoft.com/azure/key-vault/general/private-link-service)
- [Network security perimeter concepts](https://learn.microsoft.com/azure/private-link/network-security-perimeter-concepts)
- [Key Vault references in App Service and Functions](https://learn.microsoft.com/azure/app-service/app-service-key-vault-references)
- [Managed HSM overview](https://learn.microsoft.com/azure/key-vault/managed-hsm/overview) and [Azure Cloud HSM overview](https://learn.microsoft.com/azure/cloud-hsm/overview)
- [SSL.com — certificate validity changes under SC-081v3](https://www.ssl.com/article/ssl-certificate-validity-changes-what-you-need-to-know/) — the 200/100/47-day schedule explained.

### .NET and ASP.NET Core

- [ASP.NET Core security overview](https://learn.microsoft.com/aspnet/core/security/) and [authentication overview](https://learn.microsoft.com/aspnet/core/security/authentication/)
- [Configure JWT bearer authentication in ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/authentication/configure-jwt-bearer-authentication)
- [Policy-based authorization](https://learn.microsoft.com/aspnet/core/security/authorization/policies) and [resource-based authorization](https://learn.microsoft.com/aspnet/core/security/authorization/resourcebased)
- [ASP.NET Core Data Protection](https://learn.microsoft.com/aspnet/core/security/data-protection/introduction) and [configuration (key persistence and protection)](https://learn.microsoft.com/aspnet/core/security/data-protection/configuration/overview)
- [Passkeys in ASP.NET Core Identity (.NET 10)](https://learn.microsoft.com/aspnet/core/security/authentication/passkeys)
- [.NET 10 breaking change: cookie login redirects disabled for API endpoints](https://learn.microsoft.com/dotnet/core/compatibility/aspnet-core/10/cookie-authentication-api-endpoints)
- [Antiforgery in ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/anti-request-forgery), [CORS](https://learn.microsoft.com/aspnet/core/security/cors) and [rate limiting middleware](https://learn.microsoft.com/aspnet/core/performance/rate-limit)
- [Microsoft.Identity.Web documentation](https://learn.microsoft.com/entra/msidweb/) and [GitHub repository and wiki](https://github.com/AzureAD/microsoft-identity-web)
- [MSAL.NET documentation](https://learn.microsoft.com/entra/msal/dotnet/)
- [Azure SDK for .NET — authentication](https://learn.microsoft.com/dotnet/azure/sdk/authentication/) and [credential chains / DefaultAzureCredential guidance](https://learn.microsoft.com/dotnet/azure/sdk/authentication/credential-chains)
- [What's new in .NET 10 libraries — including post-quantum cryptography](https://learn.microsoft.com/dotnet/core/whats-new/dotnet-10/libraries)
- [.NET cryptography model](https://learn.microsoft.com/dotnet/standard/security/cryptography-model)
- [NuGet package auditing](https://learn.microsoft.com/nuget/concepts/auditing-packages) and [package source mapping](https://learn.microsoft.com/nuget/consume-packages/package-source-mapping)
- [Sample: ASP.NET Core web app + Microsoft identity platform (incremental tutorial)](https://github.com/Azure-Samples/active-directory-aspnetcore-webapp-openidconnect-v2)
- [Duende BFF documentation](https://docs.duendesoftware.com/bff/) — the clearest written treatment of the BFF pattern in .NET.
- [OpenIddict documentation](https://documentation.openiddict.com/)
- [Damien Bowden's blog](https://damienbod.com/) — years of practical ASP.NET Core, Entra, OIDC, BFF and passkey posts.

### OWASP standards and cheat sheets

- [OWASP Top 10:2025](https://owasp.org/Top10/2025/)
- [OWASP API Security Top 10 (2023)](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
- [OWASP ASVS](https://owasp.org/www-project-application-security-verification-standard/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/) — especially:
  [OAuth 2.0](https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html),
  [Authorization](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html),
  [Secrets Management](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html),
  [Password Storage](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html),
  [SSRF Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html),
  [.NET Security](https://cheatsheetseries.owasp.org/cheatsheets/DotNet_Security_Cheat_Sheet.html)
- [OWASP Top 10 for Agentic Applications (2026)](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) and [OWASP Top 10 for LLM Applications](https://genai.owasp.org/llm-top-10/)
- [NIST SP 800-63B — Authentication and Lifecycle Management](https://pages.nist.gov/800-63-4/sp800-63b.html)
- [Have I Been Pwned — Pwned Passwords API (k-anonymity)](https://haveibeenpwned.com/API/v3#PwnedPasswords)

### Platform, network, supply chain and operations

- [Microsoft cloud security benchmark](https://learn.microsoft.com/security/benchmark/azure/overview)
- [Azure Policy overview](https://learn.microsoft.com/azure/governance/policy/overview)
- [Microsoft Defender for Cloud](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-cloud-introduction) and [Microsoft Sentinel](https://learn.microsoft.com/azure/sentinel/overview)
- [Private endpoint DNS configuration](https://learn.microsoft.com/azure/private-link/private-endpoint-dns)
- [Default outbound access in Azure (private subnets by default)](https://learn.microsoft.com/azure/virtual-network/ip-services/default-outbound-access)
- [Secure traffic to Front Door origins](https://learn.microsoft.com/azure/frontdoor/origin-security) and [WAF on Azure Front Door](https://learn.microsoft.com/azure/web-application-firewall/afds/afds-overview)
- [API Management validate-jwt policy](https://learn.microsoft.com/azure/api-management/validate-jwt-policy)
- [AKS workload identity](https://learn.microsoft.com/azure/aks/workload-identity-overview)
- [GitHub Actions — configuring OpenID Connect in Azure](https://docs.github.com/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-azure)
- [Azure confidential computing](https://learn.microsoft.com/azure/confidential-computing/overview)
- [NIST Secure Software Development Framework (SSDF)](https://csrc.nist.gov/projects/ssdf)
- [SLSA — Supply-chain Levels for Software Artifacts](https://slsa.dev/), [Sigstore](https://www.sigstore.dev/) and [CycloneDX SBOM standard](https://cyclonedx.org/)

---
# Quick-recall sheet

**Stance**: security is a system property relative to a threat model · protect CIA + accountability · boundaries are where threats live · defense in depth with independent layers · least privilege on actions, scope and time; judge by blast radius · Zero Trust = verify explicitly, least privilege, assume breach · secure by default, fail closed, complete mediation · eliminate before protect.

**Credential ranking**: none (managed identity) > federated (GitHub/K8s/MI as FIC) > non-exportable asymmetric key (HSM) > exportable certificate > shared secret > human password for automation.

**Threat modeling**: four questions (working on / go wrong / do about it / good enough) · DFD: external entity, process, data store, data flow, trust boundary · STRIDE = Spoofing→authN, Tampering→integrity, Repudiation→accountability, Info disclosure→confidentiality, DoS→availability, EoP→authZ · per element (entity S,R · flow T,I,D · store T,I,D,(R) · process all) and per interaction · rate likelihood×impact or bug bar (not DREAD); CVSS for vulnerabilities · mitigate / eliminate / transfer / accept · 90 minutes, in the repo, repeated.

**OAuth/OIDC**: roles RO, client, AS, RS · front channel = code only; back channel = tokens · access token → API (opaque to client) · refresh token → AS only, high value · ID token → client only, never to APIs · JWT validation: signature via JWKS/kid, alg allow-list, iss, aud, exp/nbf, typ, then scp/roles · code + PKCE: verifier → S256 challenge; state (CSRF), nonce (ID token replay), exact redirect URI, iss (mix-up) · client credentials → roles, no scp · device flow (block by CA when unneeded) · implicit and ROPC removed · refresh: rotation + reuse detection or DPoP/mTLS; Entra SPA RT = 24 h · scopes limit the client, roles grant the principal; delegated = intersection · one audience per API · OBO/token exchange per hop, never forward tokens · SPA → BFF + HttpOnly `__Host-` cookie + custom-header CSRF + CSP · DPoP (RFC 9449), mTLS (8705), PAR (9126), RAR (9396), FAPI 2.0 final Feb 2025 · RFC 9700 (Jan 2025) = BCP; OAuth 2.1 still draft -15 · MCP: RFC 9728 metadata, resource indicators, CIMD, no token passthrough.

**Entra**: tenant = isolation unit · workforce / B2B / External ID (B2C: no new customers since May 2025, P2 gone Mar 2026, supported to ≥ May 2030) · app registration = definition; service principal = per-tenant instance · delegated (scp) vs application (roles, admin consent) · restrict user consent · token version set by API manifest; v2 iss `login.microsoftonline.com/{tid}/v2.0`, aud = client ID · key on tid+oid, never email · multi-tenant: validate issuer template with tid **and** allow-list tenants · tokens 60–90 min, ≤28 h with CAE; groups overflow at 200 → app roles · managed identity: user-assigned for prod, one per workload, data-plane roles, token caching delays revocation, SSRF→IMDS risk · WIF: issuer + exact subject; federate prod only to protected environments; MI as FIC GA May 2025 · CA: MFA for all, phishing-resistant for admins, block legacy auth and device code, compliant devices; passkeys (synced GA Mar 2026) resist AiTM · PIM, break-glass with FIDO2, < 5 Global Admins · mandatory MFA Phase 2 from Oct 1, 2025 for ARM writes → automation as workload identities · RBAC = principal + role + scope; control vs data plane; Contributor can listKeys.

**Key Vault**: secrets (read), keys (use in place), certificates (lifecycle) · Standard / Premium (HSM keys) / Managed HSM (single-tenant, security domain) / Cloud HSM (IaaS, successor of Dedicated HSM, retired to new customers; existing supported to Jul 31, 2028) · RBAC default for new vaults from API 2026-02-01; older control-plane APIs retire Feb 27, 2027; access policies let Contributor self-grant · roles: Secrets User, Crypto Service Encryption User, Crypto User, Reader · private endpoint + public access off; NSP GA · soft delete 7–90 days always; purge protection irreversible, required for CMK · cache secrets, singleton clients, per-app vaults · envelope encryption: DEK per object, KEK in HSM; rotate KEK = rewrap · CMK = kill switch + control, not protection from app compromise · rotation: eliminate → platform-rotated → dual-credential automation via SecretNearExpiry · public TLS: 200 days (since Mar 15, 2026) → 100 (2027) → 47 (2029) · consume via platform references, config provider with ReloadInterval + IOptionsMonitor, CryptographyClient.

**.NET**: UseAuthentication sets User, never rejects; UseAuthorization challenges/forbids · .NET 10: cookie auth → 401/403 for API endpoints; auth metrics; passkeys (Identity schema v3, ServerDomain); PQC (ML-KEM, ML-DSA, SLH-DSA) · `AddMicrosoftIdentityWebApi`; JwtBearer with explicit audience, alg allow-list, small skew, MapInboundClaims=false · fallback policy = deny by default · resource-based auth, 404 not 403 · DTOs vs mass assignment · IDownstreamApi: GetForUserAsync (OBO) / GetForAppAsync; distributed encrypted token cache; SignedAssertionFromManagedIdentity; cp1 · ManagedIdentityCredential in prod; `AZURE_TOKEN_CREDENTIALS=prod` · disable local auth everywhere + Azure Policy · Data Protection: shared, encrypted key ring + application name · EF parameterization, encoding, antiforgery, explicit CORS, SSRF IP checks after DNS, LocalRedirect, NonBacktracking · Identity PBKDF2-SHA512 100k · AesGcm unique nonce, FixedTimeEquals, RandomNumberGenerator, crypto agility · tenant from token; query filters + RLS; tenant in cache keys/messages/RAG; cross-tenant tests.

**Platform**: private endpoints + privatelink DNS · NSGs · NSP · explicit egress (new VNets private by default since Mar 31, 2026) · Front Door WAF + DDoS; lock origins (Private Link or service tag + X-Azure-FDID) · APIM validates early, services still authorize · harden: public access off, local auth off, TLS 1.2+, no anonymous blobs, logs, soft delete, locks; Azure Policy deny · TLS everywhere; mTLS where tokens don't fit · supply chain (OWASP A03:2025): NuGet audit (transitive), CPM, lock files, source mapping, secret push protection, SHA-pinned actions, SBOMs, signed images · CI/CD: OIDC federation, per-env identities, protected environments · detect: sign-in/audit logs, Key Vault audit, Sentinel; respond: rotate/revoke/disable in minutes.

**Standards**: OWASP Top 10:2025 (A01 access control incl. SSRF, A02 misconfig, A03 supply chain, A10 exceptional conditions) · API Top 10 2023 (API1 BOLA) · ASVS 5.0 (May 2025) · Agentic Top 10 (Dec 2025), LLM Top 10 2026 · NIST 800-207, 800-63B, CSF 2.0, SSDF · compliance is a floor.

**The one-liner**: *Eliminate secrets, data and exposure where you can; verify every request at every hop with short-lived, audience-bound credentials and deny-by-default authorization; contain what you can't prevent; detect what gets through; and write down the risks you accept.*

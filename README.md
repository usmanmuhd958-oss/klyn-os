<div align="center">

# Klyn OS

**A headless-first runtime for autonomous software engineering with evidence-gated execution**

**INTENT → AUTHORIZATION → ISOLATED EXECUTION → VERIFICATION → EVIDENCE → STATE TRANSITION**

*Least privilege. Explicit boundaries. Proof before mutation.*

</div>

---

> **THE CORE RULE**
>
> An agent may propose an operation. It cannot authorize itself, expand its capabilities, disable containment, or commit protected state. Policy authorizes; the native runtime enforces; the verifier evaluates; the commit gate mutates.

## Document map

1. [Purpose and operating principles](#1-purpose-and-operating-principles)
2. [Architecture](#2-architecture)
3. [Execution contract](#3-execution-contract)
4. [Isolation and containment](#4-isolation-and-containment)
5. [Security model](#5-security-model)
6. [Evidence and auditability](#6-evidence-and-auditability-kel)
7. [Supply-chain integrity](#7-supply-chain-integrity-and-release-gates)
8. [MCP integration](#8-mcp-integration)
9. [Quickstart](#9-quickstart)
10. [Repository and change ownership](#10-repository-and-change-ownership)
11. [Qualification strategy](#11-qualification-strategy)
12. [Non-goals and honest limits](#12-non-goals-and-honest-limits)
13. [Architectural references](#13-architectural-references)

---

## 1. Purpose and operating principles

Klyn OS provides a runtime for autonomous software engineering. It converts an engineering request into a bounded execution plan, authorizes each operation, executes work in an isolated environment, validates the resulting evidence, and applies state changes through explicit, auditable transitions.

Klyn OS uses a **headless-first** model. Clients—including command-line tools, IDE integrations, CI systems, and agent applications—submit requests through controlled API and MCP interfaces. The native runtime is responsible for privileged execution. TypeScript services coordinate requests but MUST NOT bypass native enforcement.

Every agent operation is bound to an explicit scope, and its authorization and execution evidence are recorded. Each operation is attributable and revocable. It is reproducible when all relevant inputs are available and immutable. The system MUST reject an operation when required evidence is absent or invalid.

> **Project status and scope**
>
> Klyn OS is in alpha development. This document defines architectural requirements and verification criteria. It does not assert that every specified control is implemented or qualified. Each implementation and release claim MUST be tied to an identifiable source revision, build, artifact, and supporting evidence.

### 1.1 Normative language and implementation truth

This README specifies required architecture and release behavior. It is not, by itself, proof that a control exists in a particular build.

| Term | Exact meaning | Permitted claim |
|---|---|---|
| **MUST / MUST NOT** | Mandatory invariant | A release violating it is not qualified |
| **SHOULD / SHOULD NOT** | Strong default | A deviation requires documented threat analysis and an approved exception |
| **MAY** | Optional implementation choice | It must not weaken a mandatory invariant |
| **REQUIRED** | Architecture or release process must provide the control | Describes the target contract, not implementation status |
| **IMPLEMENTED** | Exact source revision contains enforcement code and relevant tests pass | May be claimed only with revision-specific evidence |
| **QUALIFIED** | Implementation evidence, integration tests, adversarial tests, and release attestations exist for the exact artifact | May be claimed only with evidence tied to that artifact |

> **Status logic**
>
> - **REQUIRED ≠ IMPLEMENTED**
> - **IMPLEMENTED ≠ QUALIFIED**
> - **QUALIFIED** requires verifiable evidence for the exact source revision, build, and artifact.
>
> A design description, TODO, configuration placeholder, successful compile, or skipped test does not establish implementation or qualification.

Reports MUST distinguish `PASS`, `FAIL`, `UNSUPPORTED`, and `NOT_RUN`. For a security-sensitive control, `NOT_RUN` and `UNSUPPORTED` MUST NOT be interpreted as passing results.

**Formal release predicates**

Let `R` be a release candidate, `S(R)` its source revision, `A(R)` its built artifact, and `C(R)` the mandatory controls applicable to its declared platform and risk profile.

- `REQUIRED(R)` means every control in `C(R)` is specified by the applicable architecture and policy.
- `IMPLEMENTED(R)` means the enforcement code for those controls exists at `S(R)` and the implementation tests for that revision pass.
- `QUALIFIED(R)` is true only when implementation evidence is bound to `S(R)`, the build and artifact digests are identified, required integration and adversarial tests pass, signatures and provenance verify, and every applicable platform control is demonstrated.

The release predicate is:

> `QUALIFIED(R) ⇔ IMPLEMENTED(R) ∧ TestsPass(R) ∧ ProvenanceValid(A(R)) ∧ SignaturesValid(A(R)) ∧ PlatformControlsEnforced(R) ∧ EvidenceBound(S(R), A(R))`.
>
> If any applicable mandatory control reports `FAIL`, `UNSUPPORTED`, or `NOT_RUN`, then `QUALIFIED(R)` is false. An approved exception changes the applicable policy scope; it does not change the result of a failed control.

### 1.2 System contract at a glance

| Property | Required contract |
|---|---|
| Operating model | Headless-first API/MCP gateway plus native execution runtime |
| Trust model | Zero trust between clients, agents, control plane, workers, and artifacts |
| Execution | Explicit plans, bounded capabilities, isolated workers, verification before mutation |
| State integrity | Versioned transitions with compare-and-swap (CAS) and fencing |
| Evidence | Content-addressed records and append-only Merkle commitments |
| Network | Deny by default; explicit destination and protocol grants only |
| Credentials | Secret-brokered, short-lived, scoped; never inferred from prompt text |
| Unsupported platform | Fail closed; no silent downgrade to an unconfined executor |
| Release qualification | Evidence-bound; source revision, build, tests, signatures, and provenance must agree |

## 2. Architecture

> **Architecture invariant**
>
> A request crosses independent trust boundaries. No component may inherit authority merely because an upstream component accepted, transformed, or forwarded the request.

### 2.1 Control flow

The sequence below defines the required request and execution path. Each numbered stage is an independent trust boundary. Acceptance by one stage does not establish trust at the next stage. Each stage MUST validate the authority and evidence relevant to its responsibility. Upstream validation MUST NOT replace local enforcement.

**01 — External client**

`[ CLIENT / AGENT ]`  
`       |`  
`       |  intent + immutable references`  
`       v`

**02 — Protocol gateway**

`[ MCP / API GATEWAY ]`  
`       |`  
`       |  authenticate · validate · rate-limit`  
`       v`

**03 — Identity and policy**

`[ IDENTITY + POLICY DECISION ]`  
`       |`  
`       +---- DENY ----> [ REJECT + AUDIT DECISION ]`  
`       |`  
`       +---- ALLOW ---> [ ORCHESTRATOR ]`

**04 — Plan and capability issuance**

`[ ORCHESTRATOR ]`  
`       |  canonical plan + idempotency key`  
`       v`  
`[ CAPABILITY BROKER: OCap + RuntimeLease ]`

**05 — Native enforcement**

`[ NATIVE RUNTIME DAEMON ]`  
`       |`  
`       +---- standard risk ----> [ LINUX SANDBOX WORKER ]`  
`       |`  
`       +---- elevated risk ----> [ MICROVM WORKER ]`

**06 — Verification and commit gate**

`[ SEALED WORKER RESULT ]`  
`       |`  
`       v`  
`[ VERIFIER: TESTS + POLICY + ARTIFACT CHECKS ]`  
`       |`  
`       +---- FAIL / INCOMPLETE ---> [ REJECT; NO PROTECTED MUTATION ]`  
`       |`  
`       +---- VERIFIED -----------> [ COMMIT GATE: CAS + FENCING ]`  
`                                      |`  
`                                      v`  
`                                [ AUTHORITATIVE STATE ]`

**Evidence path — independent of control-flow result**

The gateway, orchestrator, native runtime, verifier, and commit gate emit typed events to the Klyn Evidence Ledger (KEL). Evidence recording MUST NOT grant authority, bypass state-transition rules, or expose plaintext credentials.

> **Trust-boundary rule**
>
> A successful gateway response is not proof of execution. A successful worker exit is not proof of correctness. A passing verifier is not permission to ignore CAS. A ledger entry is not a substitute for authorization.

### 2.2 Trust boundaries and ownership

| Boundary | Responsibility | MUST NOT be trusted to |
|---|---|---|
| Client / agent | Submit intent and receive scoped results | Grant itself permissions or assert its own verification |
| MCP / API gateway | Authenticate, validate protocol envelopes, rate-limit, and route | Execute arbitrary host commands or expose ambient credentials |
| Policy engine | Decide whether a principal may perform an action on a resource | Treat model-generated text as policy |
| Orchestrator | Build plans, issue leases, coordinate idempotent operations | Bypass authorization or mutate protected state outside CAS |
| Native daemon | Enforce local execution policy and launch workers | Trust caller-supplied paths, IDs, environment, or privilege claims |
| Worker / microVM | Execute one bounded workload | Retain authority after lease expiry or reach undeclared resources |
| Verifier | Evaluate outputs against declared acceptance criteria | Modify the candidate result it is verifying |
| Evidence ledger | Commit tamper-evident execution records | Store plaintext secrets or silently rewrite history |

> **Authority invariant**
>
> The model proposes. The policy engine authorizes. The runtime enforces. The verifier evaluates. The commit gate mutates state. No model response alone is sufficient authority for a protected operation.

### 2.3 Logical subsystem map

These boundaries define subsystem responsibility. They do not assert that every subsystem is implemented or production-qualified.

| Subsystem | Owns | Explicitly does not own |
|---|---|---|
| Gateway and protocol adapters | MCP/HTTP validation, authentication hand-off, rate limits, tool discovery, response normalization | Host-level execution authority |
| Control plane | Project and execution coordination, policy decisions, idempotency keys, leases, state transitions, cancellation | Direct unconfined process execution |
| Native runtime | `klyn-daemon`, `klyn-cli`, worker lifecycle, local policy enforcement, platform capability detection | Tenant policy inferred from prompt text |
| Isolation layer | `klyn-sandbox`, filesystem boundary (`klyn-fs`), resource controls, syscall policy, optional microVM execution | Cross-tenant data access |
| Execution engine | `klyn-runtime`, task scheduling, bounded tool invocation, deterministic transition processing | Silent policy relaxation |
| Evidence layer | KEL, artifact digests, attestations, verification records, audit export | Plaintext credential storage |

The native runtime is the enforcement boundary. A gateway or orchestration service may request an operation, but it cannot attest that the native runtime enforced the request. Protocol adapters and orchestration services SHOULD remain replaceable without weakening local enforcement.

## 3. Execution contract

### 3.1 Intent is not authority

Every request MUST be normalized into a typed operation with the following fields:

- **Identity:** authenticated principal and tenant.
- **Target:** project, repository, and immutable revision.
- **Operation:** declared action and canonical resource scope.
- **Inputs:** input and configuration digests.
- **Tools and network:** requested tools and network destinations.
- **Budgets:** time, CPU, memory, process, and storage limits.
- **Verification:** acceptance policy and required approvals.
- **Concurrency:** idempotency key and expected state version.

Free-form prompt text is untrusted input. It may describe intent, but it MUST NOT add permissions, change tenant identity, weaken policy, or authorize secret access.

The authorization decision MUST use the authenticated identity, explicit policy, canonical resource identifiers, and requested operation. Claims in user-controlled text MUST NOT affect that decision.

> **Intent is data, not authority.**
>
> Neither natural-language instructions nor retrieved content can create capabilities, switch tenants, weaken a policy, or approve their own execution.

### 3.2 Least-privilege capability tokens: OCap

The object-capability contract (**OCap**) grants narrowly scoped authority over specified actions and resources. It does not grant a general user-session privilege.

A capability MUST be cryptographically authenticated and bind, at minimum:

- **Identity:** issuer, subject, audience, tenant, and workload identity.
- **Authority:** allowed operation set and canonical resource scope.
- **Constraints:** byte limits, destination allowlist, permitted paths, and operation count.
- **Validity:** issue time, expiry, unique token identifier, and policy version/digest.
- **Delegation:** maximum depth and attenuated child scopes.
- **Execution binding:** execution or lease binding where supported.

**Mandatory enforcement rules**

1. **No ambient authority:** workers receive only capabilities needed for the assigned operation.
2. **No privilege amplification:** delegated capabilities may only narrow authority.
3. **Short lifetime:** expiry and revocation are checked at the enforcement point.
4. **Audience binding:** a token issued for one service or workload is invalid elsewhere.
5. **Replay resistance:** one-shot operations are bound to a nonce or idempotency record.
6. **No model minting:** agents cannot sign, widen, or self-approve capabilities.
7. **No bearer leakage:** tokens are never written to prompts, source files, logs, KEL payloads, or tool output.

A token is necessary but not sufficient. The native runtime MUST independently validate the token, current lease, policy version, and requested resource. A valid token MUST NOT override a denied policy decision, revoked lease, unsupported isolation profile, or expired execution.

**Capability validity predicate**

For principal `p`, action `a`, resource `r`, capability `c`, and evaluation time `t`, `ValidCapability(c, p, a, r, t)` holds if and only if every condition below is satisfied:

- The capability signature and issuer trust chain verify.
- The subject, tenant, audience, and workload identity match the authenticated execution context.
- `a` is a member of the capability's permitted operation set.
- `r` is within the capability's canonical resource scope.
- Every capability constraint is satisfied.
- The capability is within its validity interval and has not been revoked or consumed.
- The capability's policy version and execution/lease binding match the current request.

Authorization requires both an affirmative policy decision and a valid capability. Neither condition is sufficient on its own.

For principal `p`, action `a`, resource `r`, and time `t`:

> `Authorized(p, a, r, t) ⇔ PolicyAllows(p, a, r, t) ∧ ValidCapability(c, p, a, r, t) ∧ LeaseCurrent(c, t)`.
>
> If any term is false or cannot be established, authorization MUST be denied.

> **Capability invariant**
>
> Authority is explicit, attenuable, audience-bound, short-lived, and checked at the enforcement point. Possession of a token never grants authority beyond its signed scope.

### 3.3 Runtime lease and deterministic state transitions

`RuntimeLeaseV1` defines the execution lease. It binds one worker to a bounded execution context containing:

- execution ID;
- immutable input snapshot;
- policy digest;
- capability set;
- resource budget;
- expiry;
- fencing generation.

The authoritative state machine is event-driven. The lifecycle below defines the permitted transition hierarchy:

1. **QUEUED — request admitted to the control plane**
   - **AUTHORIZED — policy decision allows the exact principal, operation, and resource.**
     - **LEASED — RuntimeLeaseV1 issued with bounded capabilities and budget.**
       - **RUNNING — native runtime admits the worker under the selected isolation profile.**
         - **VERIFYING — worker result and output manifest are sealed.**
           - **COMMITTED — evidence validates and CAS succeeds with the current fencing generation.**
           - **REJECTED — verification fails or required evidence is incomplete; protected state remains unchanged.**
           - **FAILED — verifier or commit operation fails; protected state remains unchanged.**
         - **FAILED — runtime fault before a candidate result is sealed.**
         - **CANCELLED — cancellation is accepted and the worker is fenced.**
       - **EXPIRED — lease expires before worker admission or completion.**
   - **REJECTED — policy denies the request; no worker is admitted.**

The terminal states are `COMMITTED`, `REJECTED`, `FAILED`, `CANCELLED`, and `EXPIRED`. A terminal execution MUST NOT transition to `RUNNING`. A retry MUST use a new execution identifier or fencing generation and MUST undergo authorization again.

**Transition guarantees**

- **Typed transitions:** each transition has a typed event, a precondition, an expected version, and an auditable result.
- **Compare-and-swap:** state writes use CAS against the expected version. A stale worker cannot overwrite a newer state.
- **Fencing:** a monotonically increasing fencing generation invalidates prior workers after lease replacement, cancellation, or expiry.
- **Idempotency:** repeated requests with the same idempotency key resolve to the same logical operation; conflicting payloads are rejected.
- **Candidate versus commit:** a worker result is a candidate, not a commit. Only the verifier and commit gate authorize a protected mutation.
- **Failure atomicity:** failure, timeout, cancellation, or incomplete evidence leaves protected state unchanged.
- **Effective cancellation:** cancellation is not complete merely because the control plane recorded a request; the runtime MUST fence the worker and reject subsequent writes from its generation.

> **Commit invariant**
>
> A worker may produce bytes, artifacts, and proposed changes. It cannot commit them. Protected state changes only after required evidence validates and the expected version and fencing generation still match.

**Deterministic transition predicate and bounds**

Let `S` be a canonical prior state, `E` a validated event, and `Q` an immutable policy version. The state transition function is defined as `δ(S, E, Q) → S'`.

- `δ` MUST be a deterministic function over canonical inputs. Equal `(S, E, Q)` inputs MUST yield the same `S'`.
- The transition function MUST validate preconditions and produce a proposed next state; it MUST NOT perform unbounded external side effects as part of state evaluation.
- Persistence MUST use CAS against the version of `S`. A computed `S'` is not committed merely because the function returned it.
- External effects MUST be separately authorized, bounded, and represented by idempotent or uniquely identified operations. Their outcomes MUST be recorded if they influence later state.

The commit safety conditions are conjunctive:

> `Commit(E) ⇒ Authorized(E) ∧ EvidenceValid(E) ∧ LeaseCurrent(E) ∧ FenceCurrent(E) ∧ CASMatches(E)`.
>
> `¬EvidenceValid(E) ∨ ¬LeaseCurrent(E) ∨ ¬FenceCurrent(E) ∨ ¬CASMatches(E) ⇒ ¬Commit(E)`.

Neural inference is not assumed to be deterministic. Model outputs, remote services, and external systems may vary. The deterministic guarantee applies to the validated state-transition function when it receives the same canonical prior state, event, policy version, and inputs. Reproduction also requires pinned artifacts, tool versions, configuration, and all external responses that influence the result. An execution that lacks these inputs MUST NOT be classified as reproducible.

### 3.4 State-transition invariants

| Invariant | Required behavior | Failure behavior |
|---|---|---|
| Version precondition | Every mutation names the expected prior state version | Reject stale transition |
| CAS | Commit only if the expected version still matches | Return conflict; do not overwrite |
| Fencing | Worker generation must equal current lease generation | Reject stale worker output |
| Idempotency | Same key and same canonical request map to one logical operation | Return prior result or in-progress state |
| Key conflict | Same key with a different canonical request is invalid | Reject; do not reuse the operation |
| Lease expiry | Expired lease cannot authorize execution or commit | Fence and reject |
| Verification | Required evidence must validate before protected mutation | Reject and preserve prior state |
| Cancellation | Cancelled generation cannot commit afterward | Reject stale result |

## 4. Isolation and containment

The runtime selects an isolation profile according to workload risk and verified host capabilities. It MUST NOT substitute a weaker profile without an explicit policy decision.

> **Fail-closed rule**
>
> If a required isolation primitive is unavailable or cannot be verified, the worker is not admitted under that profile. It MUST NOT silently execute without the requested confinement.

### 4.1 Containment matrix

| Layer | Required control | Enforcement objective |
|---|---|---|
| Process | Dedicated worker identity, PID namespace where available, process limits | Prevent cross-workload process visibility and fork/resource abuse |
| Filesystem | Private mount view, read-only base image, scoped writable workspace, Landlock where supported | Prevent host and sibling-project file access |
| Syscalls | Seccomp-BPF allowlist/profile | Reduce kernel attack surface |
| Resources | cgroups v2 CPU, memory, PIDs, and I/O limits | Bound exhaustion and contain noisy workers |
| Namespaces | User, mount, PID, network, IPC, and UTS namespaces as supported | Separate workload views and identifiers |
| Network | Deny-by-default egress through an explicit policy proxy | Prevent unapproved exfiltration and lateral movement |
| Strong isolation | MicroVM with a dedicated guest kernel for higher-risk or multi-tenant workloads | Add a guest-kernel boundary between workload and host |

### 4.2 Linux sandbox admission contract

Before admitting a worker, the native runtime MUST probe the kernel and feature ABI required by the selected profile. A configuration setting alone does not demonstrate that the kernel enforced the requested control.

- **Namespaces:** verify each required namespace is available and successfully applied to the worker.
- **cgroups v2:** verify controller availability and successful placement of the worker into the configured resource domain.
- **Seccomp-BPF:** verify the intended filter is installed before untrusted workload code executes.
- **Landlock:** probe the supported ABI and verify the ruleset is successfully applied to the worker's filesystem view.
- **Filesystem:** verify mount isolation, read-only inputs, and narrowly scoped writable paths.
- **Network:** verify deny-by-default egress and explicit destination/protocol grants.
- **Resource limits:** verify the runtime has applied CPU, memory, process, and I/O budgets rather than merely recording requested values.

If a required control is unavailable—for example, because the Landlock ABI, a namespace, a cgroup controller, or a syscall-filter capability is unsupported—the runtime MUST reject the workload with a typed platform or policy error. It MUST NOT execute the workload without the required isolation.

### 4.3 MicroVM boundary

A microVM provides a separate guest-kernel boundary for elevated-risk or multi-tenant workloads. The runtime MUST report this profile as active only when the hypervisor and guest kernel are running and their state is measured. A container, namespace configuration, or restricted process MUST NOT be identified as a microVM.

The microVM profile SHOULD bind the guest image digest, virtual hardware configuration, vCPU and memory limits, virtual device set, host/guest communication channels, and network policy to the execution record. Guest output remains untrusted and must pass the same verification and commit gates as any other worker result.

### 4.4 Platform failure semantics

Fail-closed behavior is mandatory:

1. Detect the host and kernel capabilities required by the selected profile.
2. Resolve the requested security profile against the actual available capabilities.
3. Reject admission if any mandatory control cannot be enforced.
4. Record the unsupported control and policy version in the execution evidence.
5. Do not silently retry with weaker isolation.
6. Permit a weaker profile only when policy explicitly authorizes that profile for the workload and its risk classification.

An unsupported platform MUST produce a typed rejection. No execution path may bypass a required control or report an unavailable primitive as successfully enforced.

## 5. Security model

### 5.1 Security invariants

These are non-negotiable release requirements:

1. **Complete mediation:** every privileged operation is authorized at the point of use.
2. **Least authority:** every worker receives the minimum capability and resource budget for one task.
3. **Tenant isolation:** principal, project, execution, storage, and evidence scopes are checked independently.
4. **Immutable inputs:** execution binds to content digests and immutable revisions, not mutable branch names alone.
5. **No ambient secrets:** credentials are brokered for a narrow purpose and never exposed to the model by default.
6. **Untrusted tool output:** files, logs, web pages, issue text, compiler output, and model responses cannot modify trusted policy.
7. **Verified mutation:** protected state changes only after evidence validation and a successful versioned commit.
8. **Revocable execution:** expiry, cancellation, and policy revocation fence stale workers.
9. **Tamper-evident history:** committed evidence is append-only and cryptographically verifiable.
10. **Fail closed:** missing controls, malformed proofs, ambiguous identity, and unsupported platforms produce rejection—not a permissive fallback.

> **Non-negotiable security property**
>
> No untrusted input may directly expand authority. No stale worker may commit. No unverified result may mutate protected state. No missing isolation primitive may be treated as success.

### 5.2 Threat and hardening matrix

**Prompt injection**

- **Attack path:** malicious instructions embedded in code, documentation, web pages, or tool output.
- **Required controls:** treat retrieved content as data; separate policy from model context; require explicit capability for each tool.
- **Release evidence:** adversarial injection tests show no privilege expansion.

**Tool abuse**

- **Attack path:** an agent invokes a dangerous or out-of-scope tool.
- **Required controls:** typed tool schemas, per-tool authorization, resource scope, approval gates, and bounded execution.
- **Release evidence:** deny tests for unauthorized action/resource combinations.

**Worker escape**

- **Attack path:** malicious build or test code attacks the host kernel or runtime.
- **Required controls:** namespaces, seccomp, Landlock, cgroups, and microVM isolation for elevated-risk workloads.
- **Release evidence:** hostile workload tests, escape regression tests, and isolation telemetry.

**Secret exfiltration**

- **Attack path:** a prompt or process attempts to read or transmit credentials.
- **Required controls:** secret broker, short-lived scoped handles, egress deny-by-default, and output redaction.
- **Release evidence:** canary-secret tests, network policy tests, and log scans.

**Cross-tenant access**

- **Attack path:** forged identifiers or a confused-deputy request crosses tenant boundaries.
- **Required controls:** derive identity from authenticated context; verify tenant/resource binding at every layer.
- **Release evidence:** negative authorization matrix and isolation tests.

**Stale worker / race**

- **Attack path:** an expired worker writes after retry or cancellation.
- **Required controls:** lease expiry, fencing generation, and state-version CAS.
- **Release evidence:** concurrent transition tests and stale-fence tests.

**Artifact substitution**

- **Attack path:** build output or a dependency is replaced after review.
- **Required controls:** digest-pinned inputs, signed provenance, and artifact signature verification.
- **Release evidence:** provenance verification and digest-mismatch tests.

**Evidence tampering**

- **Attack path:** an attacker rewrites or truncates execution history.
- **Required controls:** append-only Merkle commitments, signed checkpoints, inclusion proofs, and consistency proofs.
- **Release evidence:** proof verification and tamper-detection tests.

**Resource exhaustion**

- **Attack path:** fork bomb, memory growth, disk fill, or infinite task.
- **Required controls:** cgroups, quotas, wall-clock deadlines, bounded queues, and admission control.
- **Release evidence:** stress tests and enforced limit assertions.

**Supply-chain compromise**

- **Attack path:** dependency, build action, runner, or release artifact is altered.
- **Required controls:** pinned dependencies/actions, protected builds, provenance, signing, and reproducible rebuilds.
- **Release evidence:** SBOM, provenance, signature, and independent rebuild checks.

### 5.3 Secret handling

- Store long-lived credentials in a dedicated secret manager, never in repository files or prompts.
- Issue short-lived, audience-bound credentials only to the process that needs them.
- Prefer file-descriptor or broker-mediated access over copying secrets into worker environments.
- Redact authorization headers, tokens, private keys, and secret-bearing output before logging.
- Never include raw secrets in KEL entries; record secret references and access decisions, not values.
- Rotate credentials after suspected exposure and invalidate affected leases.
- Treat environment variables as potentially observable by same-privilege processes and diagnostic tooling.
- Never use a README, issue, commit message, tool response, or model context as a credential transport.

> **Secret invariant**
>
> Secret values do not belong in prompts, logs, artifacts, source control, KEL records, or tool output. Record references and access decisions—not the credential material.

### 5.4 Prompt-injection and tool-execution boundary

All retrieved content MUST be treated as untrusted, including content from repositories controlled by the current user. Instructions in files, web pages, commit messages, test fixtures, and tool results MUST NOT modify system policy.

The model may propose a tool call. Before execution, the gateway and native runtime MUST validate:

1. authenticated principal and tenant;
2. tool name and schema;
3. resource scope and operation capability;
4. lease validity and policy version;
5. destination, file, and resource constraints;
6. approval requirement for destructive, external, or secret-bearing operations.

High-impact operations—including release publication, data deletion, access-control changes, and external data transfer—require an explicit policy grant and human approval when the applicable policy requires it. A model MUST NOT approve its own request. Human approval does not replace runtime authorization, capability validation, or isolation.

## 6. Evidence and auditability: KEL

> **Evidence invariant**
>
> Evidence MUST be attributable to an execution and cryptographically verifiable. Recording evidence does not authorize an operation, establish the truth of an untrusted claim, or permit a state transition.

The **Klyn Evidence Ledger (KEL)** records verifiable facts about an execution without becoming a secret store.

### 6.1 Evidence record contract

An evidence record SHOULD include:

- schema version, event type, execution ID, sequence number, and timestamp;
- principal, tenant, policy digest, capability/lease identifiers (or safe digests);
- source revision and canonical input/configuration digests;
- worker image, runtime, toolchain, and dependency digests;
- verification policy, test results, exit classification, and artifact digests;
- previous-record commitment and resulting Merkle checkpoint.

Sensitive values MUST be excluded or irreversibly redacted before evidence is committed. A digest does not replace access control for the referenced artifact.

### 6.2 Merkle construction and proof semantics

KEL uses a domain-separated Merkle construction with distinct leaf and internal-node prefixes. It supports:

- **Inclusion proofs:** show that a record is committed under a particular root.
- **Consistency proofs:** show that a newer checkpoint extends a previously observed tree.
- **Signed checkpoints:** bind a root hash and tree size to an authorized signer.
- **Independent verification:** verify proofs without trusting the component that produced the record.

The design follows the Merkle inclusion and append-only consistency model described in [RFC 9162](https://www.rfc-editor.org/rfc/rfc9162.html). The hash suite, canonical serialization, key-rotation procedure, checkpoint format, and verifier version MUST be versioned.

> **Auditability limit**
>
> A Merkle root alone does not prove that an event is truthful. Signed identities, trustworthy event capture, secure key custody, and independent verification remain necessary. KEL MUST NOT be treated as proof that a tool action was safe merely because the action has a ledger entry.

## 7. Supply-chain integrity and release gates

> **Supply-chain invariant**
>
> A release artifact MUST be traceable to an identified source revision and build process. Unsigned, unprovenanced, or digest-mismatched artifacts MUST NOT be admitted to a trusted release channel.

Klyn OS treats source, build infrastructure, dependencies, and release artifacts as part of the trusted computing base.

### 7.1 Required release controls

- **Protected source:** protect the default branch; require reviewed changes and status checks for privileged code.
- **Signed source:** sign commits and release tags; verify signer identity and signature policy in CI.
- **Pinned build:** pin third-party CI actions and build tools to immutable revisions.
- **Locked dependencies:** record resolved dependency digests and produce an SBOM (SPDX or CycloneDX).
- **Build provenance:** generate SLSA-compatible provenance identifying source revision, builder, build definition, resolved dependencies, and artifact subjects.
- **Artifact signatures:** sign artifacts and attestations; verify them before deployment or execution.
- **Reproducibility:** rebuild from the same declared inputs in an isolated environment and compare artifact digests where reproducibility is claimed.
- **Fail closed:** reject unsigned, unprovenanced, or digest-mismatched production artifacts.

**Reference baselines**

- [SLSA build tracks](https://slsa.dev/spec/v1.0/levels)
- [SLSA provenance](https://slsa.dev/spec/v1.1/provenance)
- [Sigstore documentation](https://docs.sigstore.dev/)
- [NIST SP 800-218 — Secure Software Development Framework](https://csrc.nist.gov/pubs/sp/800/218/final)

### 7.2 Evidence required for a release claim

A control is not satisfied by a checklist entry. Its release record MUST link to the actual CI run, attestation, signature-verification result, test report, or reproducible-build comparison.

| Claim | Minimum evidence |
|---|---|
| Signed source | Verified commit/tag signature and signer-policy result |
| Pinned build | Immutable action/tool references and recorded build definition |
| SBOM available | Versioned SPDX or CycloneDX document tied to the artifact digest |
| SLSA provenance | Verifiable provenance naming the source, builder, build definition, and artifact subject |
| Artifact signed | Signature and certificate/identity verification result |
| Reproducible build | Independent rebuild and digest comparison for the declared environment |
| Qualified sandbox | Host/kernel capability report and successful enforcement tests for the exact profile |
| Qualified release | All mandatory gates passed for the exact source and artifact digests |

> **Release-claim invariant**
>
> Do not advertise a SLSA level, signed-build guarantee, or reproducibility guarantee until the corresponding requirements are independently met and linked to the exact artifact being released.

## 8. MCP integration

MCP defines a protocol interface; it does not provide authentication or authorization by itself. Klyn OS exposes only explicitly registered tools and resources. The deployment operator MUST provide the endpoint URL. This README does not publish a deployment-specific endpoint.

### 8.1 Client connection contract

The client configuration MUST include:

- the deployment's HTTPS MCP endpoint;
- a short-lived bearer credential or supported OAuth flow;
- the expected tenant/project scope;
- the approved tool set;
- the protocol version supported by both client and server.

Store credentials in the client secret store. Tokens MUST NOT appear in URLs, source code, checked-in configuration, screenshots, or shell history. The connection MUST use TLS. The server MUST reject invalid credentials, unsupported protocol versions, malformed JSON-RPC envelopes, and arguments that fail schema validation.

For Streamable HTTP, clients send JSON-RPC requests to the configured MCP endpoint using the negotiated transport and protocol version. The implementation MUST follow the [MCP specification](https://modelcontextprotocol.io/specification/) for initialization, capability negotiation, tool discovery, invocation, notifications, and transport behavior. Session and streaming semantics MUST be determined from the protocol version supported by both endpoints.

> **MCP trust invariant**
>
> MCP standardizes the protocol exchange. It does not confer authority. Authentication, tenant scoping, tool authorization, resource checks, and native runtime enforcement remain Klyn OS responsibilities.

### 8.2 Tool invocation contract

A tool call MUST be accepted only after authentication, schema validation, policy evaluation, and capability validation. Each tool implementation MUST:

- use stable, versioned tool names and JSON schemas;
- reject unknown fields where strict validation is required;
- enforce tenant and resource scope server-side;
- impose deadlines, payload limits, and rate limits;
- use idempotency keys for mutating operations;
- return a typed result with operation ID, state, and evidence reference;
- return structured errors without leaking secrets or host details.

The table below defines an illustrative response-field contract. It does not assert that every deployed tool currently returns this exact structure:

| Field | Illustrative value | Meaning |
|---|---|---|
| `operation_id` | `op_…` | Stable identifier for the logical operation |
| `state` | `accepted` | Current state-machine state |
| `result` | `null` | Result is absent until available and verified |
| `evidence.record_id` | `kel_…` | Reference to the evidence record |
| `evidence.checkpoint` | `sha256:…` | Digest identifying the checkpoint |

**Suggested machine-readable error classes**

- `UNAUTHENTICATED`
- `FORBIDDEN`
- `INVALID_ARGUMENT`
- `POLICY_DENIED`
- `LEASE_EXPIRED`
- `CONFLICT`
- `RESOURCE_LIMIT`
- `UNSUPPORTED_PLATFORM`
- `VERIFICATION_FAILED`
- `INTERNAL`

Error messages are diagnostic; authorization decisions MUST use stable machine-readable codes. Responses MUST NOT disclose secret values, host filesystem details, internal credentials, or information belonging to another tenant.

### 8.3 Native runtime boundary

Communication between the gateway and daemon MUST use an authenticated local channel, explicit peer identity, and a versioned message schema. JSON-RPC over a Unix domain socket is one possible transport. The daemon MUST independently validate the request, capability, lease, input digest, and platform policy.

The gateway MUST NOT be able to invoke an unrestricted host shell through a generic proxy method. The daemon MUST treat all gateway-supplied paths, IDs, environment entries, tool arguments, and artifact references as untrusted until independently validated.

## 9. Quickstart

> **Execution caution**
>
> Run repository commands only in an environment whose credentials, working directory, and host permissions are appropriate for the task. Do not place bearer tokens, private keys, or production secrets in command arguments or shell history.

### 9.1 Get the repository

1. Clone the repository: `git clone https://github.com/usmanmuhd958-oss/klyn-os.git`
2. Enter the checkout: `cd klyn-os`

### 9.2 Inspect the available workspace commands

The package scripts and Rust workspace metadata in the checked-out revision define the available commands. Documentation or examples MUST NOT be treated as evidence that a command exists.

1. Install the locked JavaScript dependencies: `pnpm install --frozen-lockfile`
2. List declared package scripts: `pnpm run`
3. Inspect the Rust workspace metadata: `cargo metadata --no-deps --format-version 1`

Run only the build, test, lint, and qualification scripts declared by the checked-out revision. Native sandbox qualification requires a supported Linux host and the required kernel features; unsupported platforms MUST report an explicit unsupported result rather than silently running an unconfined worker.

> **Quickstart boundary**
>
> These commands inspect and prepare the declared workspace. They do not, by themselves, establish that the native sandbox, microVM profile, production MCP endpoint, or release supply chain is qualified.

### 9.3 Connect an MCP client

Configure the HTTPS endpoint and credential through the client or deployment secret manager:

| Configuration key | Value source | Handling requirement |
|---|---|---|
| `KLYN_MCP_URL` | Deployment-provided MCP endpoint | Environment/secret configuration; do not hard-code |
| `KLYN_MCP_TOKEN` | Short-lived bearer credential, when bearer authentication is enabled | Secret store only; never commit or paste into prompts |

These values MUST NOT be committed or included in prompts and documentation. Use the deployment's published protocol and authorization configuration. The endpoint path and supported protocol version are deployment-specific.

## 10. Repository and change ownership

This table defines logical ownership. It does not prescribe a directory layout. Ownership assigns responsibility for interfaces and enforcement; it does not establish that a subsystem is implemented, deployed, or qualified.

| Area | Owns | MUST NOT own |
|---|---|---|
| Gateway | MCP/HTTP protocol, request validation, auth hand-off | Host-level execution authority |
| Control plane | Plans, policy decisions, leases, CAS state | Direct unconfined process execution |
| Native daemon | Worker admission, local enforcement, lifecycle | Tenant policy inferred from prompt text |
| Sandbox / filesystem | Isolation profiles and resource boundaries | Cross-tenant data access |
| Runtime | Scheduling, task protocol, cancellation, result sealing | Silent policy relaxation |
| Verifier | Acceptance checks and proof validation | Mutating the candidate it checks |
| KEL | Append-only evidence commitments | Plaintext credential storage |

Privileged enforcement SHOULD reside in the smallest reviewable native component set. Protocol adapters and orchestration services SHOULD remain replaceable without weakening local enforcement. A subsystem name identifies an architectural responsibility; it does not establish implementation completeness.

## 11. Qualification strategy

Compilation alone does not qualify a release candidate. The qualification suite SHOULD cover the following areas.

### 11.1 Unit and property tests

- Unit tests for capability parsing, authorization, canonicalization, and state transitions.
- Property tests for CAS, idempotency, lease expiry, fencing, and transition monotonicity.
- Invariants proving that stale versions and stale fencing generations cannot commit.

### 11.2 Protocol and parser assurance

- Fuzzing for protocol frames, parsers, token claims, and evidence records.
- Malformed JSON-RPC envelope and schema-validation tests.
- Tests for oversized payloads, timeouts, rate limits, duplicate requests, and conflicting idempotency keys.

### 11.3 Adversarial security tests

- Prompt injection, path traversal, symlink races, malformed tool arguments, and cross-tenant identifiers.
- Canary-secret tests covering prompt, process, log, artifact, and network-exfiltration paths.
- Unauthorized tool/resource combinations and approval-gate bypass attempts.
- Worker termination, lease expiry, cancellation, retry, and stale-write races.

### 11.4 Native isolation tests

- Linux integration tests for namespaces, cgroups v2, seccomp-BPF, Landlock, and filesystem restrictions.
- Tests that prove the expected controls were applied to the actual worker before untrusted code ran.
- MicroVM tests only on hosts that actually provide the required virtualization support.
- Stress tests for CPU, memory, process count, disk, I/O, and bounded-queue limits.

### 11.5 Evidence and supply-chain tests

- KEL inclusion/consistency proof tests and independent verifier checks.
- Signature verification, SBOM validation, provenance verification, and reproducible-build checks.
- Digest-mismatch, untrusted-signer, expired-identity, and malformed-attestation negative tests.

> **Qualification rule**
>
> Every security-sensitive test MUST fail closed. A skipped test does not pass the corresponding control. Reports MUST distinguish `PASS`, `FAIL`, `UNSUPPORTED`, and `NOT_RUN`. An unsupported host may support development, but it cannot qualify a security profile whose kernel or hypervisor requirements were not tested.

## 12. Non-goals and honest limits

> **Residual-risk statement**
>
> These controls reduce specified risks. They do not prove the absence of vulnerabilities or remove reliance on the host kernel, hypervisor, hardware, cryptographic key custody, dependencies, or operators.

- Klyn OS does not guarantee deterministic model inference or semantic correctness.
- A Merkle commitment does not prove that recorded facts were true when captured.
- Containers and namespaces are not equivalent to a separate guest kernel.
- Cryptographic signatures establish integrity and signer identity only to the extent that keys and identity policy are trustworthy.
- Human approval does not replace runtime authorization or isolation.
- No architecture can eliminate every kernel, hypervisor, hardware, side-channel, dependency, or operator risk.
- A successful build does not prove that the runtime is secure, that a sandbox is active, or that an artifact is reproducible.
- A design requirement does not establish implementation status; implementation status does not establish release qualification.

Klyn OS defines mechanisms to identify, constrain, observe, and test risks. Residual risks MUST be documented for each deployed configuration and its applicable threat model.

## 13. Architectural references

> **Reference status**
>
> External standards provide design and assurance references. A citation does not establish conformance. Conformance requires an implementation mapping and verification evidence for the deployed revision.

- [Model Context Protocol specification](https://modelcontextprotocol.io/specification/)
- [RFC 9162 — Certificate Transparency Version 2.0](https://www.rfc-editor.org/rfc/rfc9162.html)
- [SLSA build track](https://slsa.dev/spec/v1.0/levels)
- [SLSA provenance](https://slsa.dev/spec/v1.1/provenance)
- [Sigstore documentation](https://docs.sigstore.dev/)
- [NIST SP 800-218 — Secure Software Development Framework](https://csrc.nist.gov/pubs/sp/800/218/final)

---

<div align="center">

**Klyn OS — Intent → Authorization → Isolated Execution → Verification → Evidence → State Transition.**

*Headless-first execution. Least-privilege authorization. Fail-closed admission. Evidence-gated state mutation.*

</div>

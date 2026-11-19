# Klyn OS Architecture and Security Comparison

**Assessment date:** 2026-11-19  
**Project maturity:** Alpha  
**Document status:** Architecture comparison; not a security certification or implementation audit

## 1. Purpose and assessment boundaries

This document compares the Klyn OS architecture specified by this repository with publicly documented execution and security models from OpenAI's Codex-based Agents API, Cursor, Anthropic's Claude Code, and GitHub Copilot.

The systems are not identical product categories. Klyn OS is specified as a headless-first software-engineering runtime with a native execution boundary and an MCP/API gateway. The comparison products combine models, agent harnesses, developer tools, hosted execution environments, or integrations in different ways. The comparison therefore focuses on architecture and security boundaries rather than model quality, benchmark scores, user experience, price, or an overall product ranking.

The assessment is based on public product documentation linked in the References section and the architecture contracts published in this repository. It is not based on source-code audits of competitor implementations, penetration tests, confidential architecture material, or independently reproduced vendor claims. Product behavior can vary by release, plan, operating system, configuration, and deployment model.

**Central conclusion:** Klyn's documented architectural thesis is to make authorization, native enforcement, deterministic state transitions, and evidence-gated mutation explicit contracts of one headless runtime. This is a meaningful design direction. It is not evidence that Klyn currently implements those contracts end to end, that the combination is unique, or that Klyn is more secure or capable than established products.

## 2. Terms and evidence discipline

The following terms have deliberately separate meanings:

| Term | Meaning | Evidence needed |
|---|---|---|
| **REQUIRED** | A design or release contract says a control must exist | Normative specification and threat-model rationale |
| **IMPLEMENTED** | The exact source revision contains the enforcement mechanism and its implementation tests pass | Source references, test results, and revision identity |
| **QUALIFIED** | A specific built artifact has passed all applicable release and platform gates | Artifact digest, test evidence, provenance, signatures, and platform-enforcement evidence |

These statuses are not interchangeable:

- **REQUIRED does not mean IMPLEMENTED.**
- **IMPLEMENTED does not mean QUALIFIED.**
- A successful compile, a declared configuration value, a TODO, a passing mock, or a skipped platform test is not proof of a security control.
- A control that is unknown, unsupported, or not tested cannot be treated as passing.

The repository describes Klyn OS as an alpha project. This comparison preserves that maturity statement. It does not assign IMPLEMENTED or QUALIFIED status to Klyn controls because this document does not contain the required per-control mapping from an exact source revision to a tested and attested release artifact.

## 3. Klyn OS target architecture

### 3.1 Headless-first execution model

The target system separates protocol handling, policy decisions, execution enforcement, verification, and protected state mutation. Clients may include a CLI, IDE integration, CI service, or another agent. They submit typed requests through an API or MCP boundary; they do not receive ambient host authority.

The intended execution sequence is:

1. **Intent intake:** authenticate the caller and normalize the request into a typed operation.
2. **Policy decision:** evaluate principal, tenant, action, canonical resource, policy version, and risk.
3. **Capability and lease issuance:** issue narrowly scoped authority and a bounded execution lease.
4. **Native admission:** independently validate the request and enforce the selected platform isolation profile.
5. **Worker execution:** run one bounded task with explicit filesystem, process, network, and resource limits.
6. **Verification:** evaluate the sealed candidate result against the declared acceptance policy.
7. **Protected commit:** check authorization, evidence, lease, fencing generation, and expected state version before mutation.
8. **Evidence recording:** retain a tamper-evident record of decisions, execution, verification, and state transition.

A gateway response is not proof that a worker ran. A successful process exit is not proof that the output is correct. A passing test suite is not permission to bypass the commit gate. A ledger record is not an authorization grant.

### 3.2 Deterministic state transition and commit contract

Let S be a canonical prior state, E a validated event, and Q an immutable policy version. The state transition function is:

**δ(S, E, Q) -> S'**

The reducer must be deterministic over canonical inputs: equal (S, E, Q) inputs must produce the same proposed S'. The reducer must validate transition preconditions and must not perform unbounded external side effects. Persistence is a separate operation guarded by compare-and-swap (CAS).

The protected commit condition is conjunctive:

**Commit(E) => Authorized(E) ∧ EvidenceValid(E) ∧ LeaseCurrent(E) ∧ FenceCurrent(E) ∧ CASMatches(E)**

All terms must be established at the commit boundary. If any term is false, stale, invalid, or unknown, the operation must be rejected and protected state must remain unchanged. A candidate result is not a committed result.

The fencing generation prevents an expired, cancelled, or replaced worker from writing after a newer execution has taken ownership. CAS prevents a worker that computed against an old state version from overwriting a newer version. Idempotency prevents retries from creating conflicting mutations under the same logical operation.

### 3.3 Capability-based authorization

The target OCap contract gives a workload only the authority required for one declared operation and resource scope. A capability should bind at least:

- Issuer, subject, audience, tenant, and workload identity.
- Permitted actions and canonical resource scope.
- Policy version or digest.
- Issue time, expiry, unique token identifier, and revocation state.
- Operation limits, data limits, network destinations, and path constraints.
- Delegation rules that permit attenuation but never privilege expansion.
- Execution or lease identity where required.

Authorization is not established by possession of a string or by model-generated text. The intended predicate is:

**Authorized(p, a, r, t) <=> PolicyAllows(p, a, r, t) ∧ ValidCapability(c, p, a, r, t) ∧ LeaseCurrent(c, t)**

The policy engine decides. The native runtime independently verifies the capability, lease, resource scope, and platform policy at the enforcement point. A valid capability cannot override a denied policy decision or an expired lease.

### 3.4 Isolation and resource containment

The Linux target profile calls for layered controls rather than reliance on a single container setting:

- Namespaces for process, mount, network, IPC, UTS, and user-identity separation where supported.
- cgroups v2 for CPU, memory, process-count, and I/O limits.
- seccomp-BPF to restrict the system-call surface.
- Landlock or an equivalent supported mechanism to constrain filesystem access.
- A minimal worker environment, sanitized environment variables, explicit mounts, and controlled device access.
- Deny-by-default network egress with narrowly defined destinations and protocols.
- A dedicated microVM profile for workloads whose threat model requires a separate guest-kernel boundary.

These controls have different purposes. Namespaces do not by themselves create a separate kernel. cgroups limit resource consumption but do not authorize file access. seccomp constrains system calls but is not a complete filesystem policy. Filesystem restrictions do not replace network policy. A microVM can strengthen the isolation boundary, but it still depends on the host, hypervisor, configuration, and operational controls.

The native runtime must verify the effective controls on the actual worker before untrusted code starts. Unsupported platforms must report UNSUPPORTED and fail closed for profiles whose required controls cannot be established. A silent fallback to an unconfined executor is not an acceptable compatibility behavior.

### 3.5 Evidence ledger and artifact provenance

The Klyn Evidence Ledger (KEL) is specified as an append-oriented, tamper-evident record of typed execution events and state transitions. A useful record model binds the event type and schema version, execution identity, source and input digests, policy and capability references, worker and platform identity, verifier outcome, artifact digests, state version, fencing generation, and timestamp.

A Merkle construction can support inclusion proofs and consistency proofs. RFC 9162 is a useful reference for these proof properties, but adopting a Merkle tree alone does not establish a trustworthy ledger. KEL still needs canonical serialization, domain-separated hashing, authenticated checkpoints, key custody, verifier implementations, retention policy, and monitoring for split views or equivocation.

A proof that a record is included in a tree proves inclusion relative to the supplied root. It does not prove that the record was true, that the worker was honest, that the signer was authorized, or that the committed state was correct. Those properties require separate controls and trusted evidence.

Build and release provenance should use a verifiable artifact statement with source identity, build definition, builder identity, parameters, dependencies, and output digests. SLSA v1.2 is the current SLSA specification at the assessment date. Klyn must not claim a SLSA level unless the exact requirements for that level are independently demonstrated.

### 3.6 MCP is a protocol boundary, not an authority boundary

MCP standardizes client-server interaction and tool exchange. It does not, by itself, grant a tool caller permission to access a tenant, repository, file, secret, or host resource.

Each mutating tool must authenticate its caller, validate its schema, apply server-side tenant and resource checks, enforce capability and lease constraints, impose budgets and deadlines, use idempotency where appropriate, and return a typed result with an operation and evidence reference. The daemon must independently validate requests received from the gateway. A generic MCP-to-shell bridge would bypass the intended privilege boundary and is not an acceptable substitute for typed native operations.

## 4. Publicly documented comparison systems

### 4.1 OpenAI Codex-based Agents API

OpenAI documents a managed Codex harness that handles agent sessions, orchestration, context compaction, recovery, and tool interaction. An application configures the agent and supplies tools; it can use an OpenAI-hosted sandbox or connect a self-hosted environment. The environment can be a remote sandbox, laptop, container, or another compute environment depending on the deployment model. The security guidance recommends workload isolation, restricted network access, review of agent actions, and separation or brokering of credentials. See the [Agents API overview](https://developers.openai.com/api/docs/guides/agents-api/overview), [architecture guide](https://developers.openai.com/api/docs/guides/agents-api/architecture), [sandbox security guide](https://developers.openai.com/api/docs/guides/agents-api/environments/security), and [self-hosted environments guide](https://developers.openai.com/api/docs/guides/agents-api/environments/self-hosted).

**Architectural overlap:** a managed agent harness, resumable sessions, tools and MCP connectivity, and a separable execution environment overlap with Klyn's gateway/control-plane/runtime decomposition.

**Comparison boundary:** Klyn's proposal makes its own native daemon, capability checks, lease/fencing rules, and CAS commit gate part of the target runtime contract. The cited OpenAI documents describe the harness and environment split, along with security controls. Their public descriptions do not establish an identical end-to-end contract to Klyn's predicates. This is a statement about the documentation reviewed, not evidence that an unmentioned control is absent from OpenAI's implementation.

### 4.2 Cursor

Cursor documents two distinct environments. Local Run Modes control command execution, approvals, and sandboxing on a developer machine. The local sandbox can constrain filesystem and network access; some operations that require broader access may run outside the sandbox after review. Cursor's Auto-review mode also uses a classifier for certain calls. Cloud Agents run in dedicated cloud virtual machines, with per-agent isolation and Firecracker-based microVM infrastructure. The cloud-agent documentation describes repository access inherited from the triggering user's Git-provider permissions and configurable network egress controls. See [Run Modes](https://cursor.com/docs/agent/security/run-modes) and the [Cloud Agent security overview](https://prod.cursor.com/docs/cloud-agent/security).

**Architectural overlap:** policy-driven approvals, OS-level local sandboxing, remote per-agent isolation, and network-egress controls overlap with portions of Klyn's containment and authorization goals.

**Comparison boundary:** local Run Modes and Cloud Agents are different security models and should not be conflated. A developer-machine sandbox has a different host trust boundary from a remote microVM. Klyn's target additionally specifies lease-bound worker admission and an evidence-validated CAS mutation contract. The cited Cursor pages do not establish that same contract; they also demonstrate that strong isolation and cloud execution are already present in the market.

### 4.3 Anthropic Claude Code

Anthropic describes Claude Code as permission-based, with read-only behavior by default and approval for many modifying or command-execution actions. Its sandboxing work uses operating-system features to create filesystem and network boundaries, reducing the need for repeated approval prompts within the configured sandbox. Anthropic's published Auto Mode engineering discussion describes additional safeguards intended to reduce approval friction while retaining risk review. See [Making Claude Code safer with sandboxing](https://www.anthropic.com/research/claude-code-sandboxing) and [How we built Claude Code auto mode](https://www.anthropic.com/research/claude-code-auto-mode).

**Architectural overlap:** least-privilege defaults, permission checks, filesystem isolation, network controls, and safer automation modes overlap with Klyn's security goals.

**Comparison boundary:** Claude Code is a coding-agent product with a permission and sandbox model; Klyn is specified as a headless runtime and state-transition system. Klyn's OCap, RuntimeLease, fencing, CAS, and KEL concepts describe a broader lifecycle contract. This does not establish that Claude Code lacks auditability, policy enforcement, or other internal controls beyond those described in the cited pages.

### 4.4 GitHub Copilot

GitHub documents local sandboxing and hosted cloud sandboxes as separate execution choices. Local sandbox controls can restrict filesystem access, network connectivity, system capabilities, and selected credential exposure; the available controls depend on the client and operating system. GitHub-hosted cloud sessions run away from the developer's machine in isolated environments. GitHub also documents agent-session logs, test and linter execution in ephemeral development environments, and signed cloud-agent commits linked to session logs. See the [Copilot security and governance overview](https://docs.github.com/en/copilot/concepts/security-governance-and-network-settings), [local sandbox documentation](https://docs.github.com/en/copilot/concepts/security-governance-and-network-settings/about-cloud-and-local-sandboxes), [cloud sandbox documentation](https://docs.github.com/en/copilot/concepts/security-governance-and-network-settings/about-cloud-sandboxes), and [agent session management](https://docs.github.com/en/copilot/how-tos/copilot-on-github/use-copilot-agents/manage-and-track-agents).

**Architectural overlap:** isolated cloud execution, configurable local restrictions, session-level traceability, validation runs, and signed commit attribution overlap with Klyn's execution and evidence objectives.

**Comparison boundary:** GitHub documents useful provenance and audit links for agent sessions. Those features should be recognized rather than described as absent. Klyn's target emphasizes a separate native commit gate whose acceptance predicate includes current authorization, valid evidence, current lease, current fencing generation, and a matching state version. The cited Copilot documentation does not establish that exact state-transition predicate.

## 5. Comparative architecture matrix

The matrix summarizes documented architectural patterns, not independently verified security effectiveness.

| Dimension | Klyn OS target | Publicly documented market patterns |
|---|---|---|
| Primary shape | Headless runtime, API/MCP gateway, native daemon | IDE agents, managed harnesses, CLI agents, hosted agent services |
| Execution | Bounded native worker; optional stronger isolation profile | Local sandboxing, hosted containers, dedicated VMs or microVMs, self-hosted environments |
| Authorization | Policy decision plus scoped capability and current lease | Permission prompts, allowlists, repository identity, tool policy, credential scoping |
| Isolation | Namespaces, cgroups, seccomp, filesystem controls, optional microVM | OS sandboxing, container environments, dedicated VMs or microVMs |
| Protected mutation | Verifier plus lease, fencing, and CAS gate | Approval, agent review, pull-request workflow, session or commit traceability |
| Evidence | KEL event commitments plus artifact provenance | Session logs, signed commits, build and release attestations, product-specific audit records |
| Main trust boundary | Native runtime independently enforces the operation | Varies by local client, hosted harness, executor, cloud environment, and provider integration |

The table must not be read as a claim that Klyn already provides every item in its target column or that the other systems lack an equivalent feature under a different name. The system descriptions are at different abstraction layers, and public documentation is not a complete inventory of proprietary implementation details.

## 6. Security properties and trade-offs

### 6.1 Authorization must be enforced where the resource is used

A gateway can authenticate and route a request, but a privileged daemon must not trust the gateway's statement that an action is allowed. It must check the principal, target, action, capability, policy version, and current lease at the execution boundary. This reduces confused-deputy risk and limits the impact of a compromised or misconfigured control-plane component.

### 6.2 Approval prompts are not a substitute for isolation

Human approval can reduce unauthorized actions, but prompt frequency can create approval fatigue. A sandbox reduces the consequences of an allowed action by limiting what the process can reach. Both mechanisms have value; neither should be presented as a complete replacement for the other.

Klyn should define risk tiers and map each tier to an enforceable profile. A policy decision may require human review, stronger isolation, restricted network access, or refusal. A user approval must never silently widen the worker's signed capability or disable a mandatory platform control.

### 6.3 A sandbox is only as strong as its effective configuration

A configuration file that requests Landlock, seccomp, namespaces, or a microVM is not evidence that the running worker received those controls. Qualification must inspect the effective worker state and test forbidden operations from inside the worker. The test must fail when a control is absent, partially applied, or unsupported.

The host kernel, hypervisor, device model, runtime, privileged helper, and deployment configuration remain part of the trusted computing base. Klyn must document these dependencies and residual risks rather than claiming that a container or microVM removes them.

### 6.4 Evidence has separate integrity and truth properties

Hash chains and Merkle trees can make later modification detectable under their assumptions. They do not make an event true. Signatures tie evidence to a key; they do not prove that the key was uncompromised or that the signer was authorized for every assertion. A sound evidence system therefore needs authenticated producers, key custody, independent verification, protected checkpoints, and a clear mapping from events to source revisions and artifacts.

### 6.5 MCP tools require explicit authority

A tool schema is not an authorization system. Tool names, JSON schemas, and protocol negotiation define how a request is expressed; server-side identity and policy checks determine whether it can be performed. Remote MCP services are separate trust domains and must be assessed and granted only the data and actions they need.

## 7. Klyn's defensible architectural thesis

Klyn can make a credible architectural contribution by implementing and validating a coherent contract across the following boundaries:

1. **Typed authority:** policy plus an attenuable capability, bound to an authenticated identity and canonical resource.
2. **Lease-bound execution:** a bounded worker lifetime, resource budget, policy digest, and fencing generation.
3. **Native enforcement:** independent checks at the point where host resources are accessed.
4. **Deterministic state management:** a pure transition function and CAS-protected persistence, with stale workers fenced.
5. **Independent verification:** candidate results are sealed and checked before protected mutation.
6. **Evidence continuity:** execution events, verification results, artifact digests, and state transitions are linked to a verifiable release record.
7. **Fail-closed platform qualification:** unsupported or untested controls are reported explicitly and cannot be promoted to passing status.

The potential distinction is the explicit composition of these contracts in a headless software-engineering runtime. The individual ideas—least privilege, sandboxing, VM isolation, approval controls, session logs, attestations, and provenance—are established engineering approaches and are already represented across the products reviewed. Klyn should claim a design-level integration, not novelty of each primitive.

## 8. Implementation and qualification gates

A public claim should be traceable from requirement to implementation, test, and exact release artifact. The following gates are the minimum path from an alpha design to a defensible qualification claim.

### Gate 1 — Requirement-to-code traceability

- Assign a stable identifier to each security requirement.
- Map each requirement to source files, enforcement points, test cases, and owner.
- Identify the supported platform and threat-model scope for each control.
- Mark every control REQUIRED, IMPLEMENTED, UNSUPPORTED, or NOT_RUN with revision-specific evidence.

### Gate 2 — Authorization and state-machine verification

- Test capability signature, issuer, audience, tenant, resource, expiry, revocation, replay, and attenuation.
- Test policy-version mismatch and lease expiration at the native enforcement point.
- Property-test deterministic δ(S, E, Q) behavior.
- Race-test CAS, cancellation, lease replacement, retries, and stale fencing generations.
- Prove that failed verification, stale state, or an expired lease leaves protected state unchanged.

### Gate 3 — Effective isolation verification

- Inspect the actual worker configuration before execution.
- Exercise denied filesystem, syscall, process, network, device, and resource operations from inside the worker.
- Test symlink and path-resolution races, inherited descriptors, environment leakage, and worker cleanup.
- Verify that a missing kernel feature produces an explicit unsupported result and prevents admission.
- Test microVM profiles on hosts that actually provide the required virtualization support.

### Gate 4 — Protocol and adversarial testing

- Fuzz protocol framing, JSON parsing, capability parsing, and KEL record parsing.
- Test malformed, duplicated, oversized, replayed, and cross-tenant requests.
- Exercise prompt injection and malicious repository content as untrusted data.
- Verify that tool output, logs, errors, and evidence do not disclose credentials or unrelated tenant data.
- Confirm that cancellation and timeouts stop or fence the underlying worker, not only the orchestration record.

### Gate 5 — Evidence and supply-chain verification

- Independently verify KEL inclusion and consistency proofs.
- Test signer identity, key rotation, revocation, checkpoint integrity, and malformed attestations.
- Generate provenance that identifies the source revision, build definition, builder, dependencies, and artifact digest.
- Verify the artifact digest and provenance from a separate consumer-side verifier.
- Do not claim a SLSA level unless all applicable SLSA v1.2 requirements are met and evidenced.

### Gate 6 — Release qualification

For release candidate R, let S(R) be its exact source revision and A(R) its artifact. The release predicate is:

**QUALIFIED(R) <=> IMPLEMENTED(R) ∧ TestsPass(R) ∧ ProvenanceValid(A(R)) ∧ SignaturesValid(A(R)) ∧ PlatformControlsEnforced(R) ∧ EvidenceBound(S(R), A(R))**

Qualification must be denied when any applicable mandatory control is FAIL, UNSUPPORTED, or NOT_RUN. A test skipped because the host lacks a kernel feature may permit development on that host, but it cannot qualify the affected security profile. Evidence must identify the exact source revision, build process, artifact digest, platform profile, and test results.

An exception must be explicit, scoped, approved, time-bounded where appropriate, and reflected in the release's declared threat model. An exception does not convert a failed control into a passing control.

## 9. Assessment and conclusion

Klyn's public architecture defines a strong set of intended invariants: authority is explicit, the native runtime is the enforcement boundary, workers are bounded by leases and capabilities, stale workers are fenced, and protected state changes only after evidence validation and CAS.

The comparison also shows that advanced agent products already document substantial security and execution capabilities: local OS sandboxing, approval and allowlist policies, hosted isolated environments, microVMs, network restrictions, credential handling, session audit trails, and signed commits. These are real capabilities and must be acknowledged in any fair comparison.

The differentiating question is therefore not whether Klyn can list more security primitives. It is whether Klyn can demonstrate that its policy, capability, lease, isolation, verifier, CAS, and evidence contracts are enforced together in the actual runtime under adversarial conditions.

Until those controls are traced to source, tested on supported platforms, bound to signed provenance, and verified for a specific artifact, Klyn remains an alpha architecture with a qualification target—not a production-qualified security platform. That is the appropriate and technically defensible statement of its current position.

## 10. References

### Klyn OS

- [Klyn OS README and normative architecture contracts](../README.md)

### Agent runtimes and execution security

- [OpenAI Agents API overview](https://developers.openai.com/api/docs/guides/agents-api/overview)
- [OpenAI Agents API architecture](https://developers.openai.com/api/docs/guides/agents-api/architecture)
- [OpenAI sandbox security](https://developers.openai.com/api/docs/guides/agents-api/environments/security)
- [OpenAI self-hosted environments](https://developers.openai.com/api/docs/guides/agents-api/environments/self-hosted)
- [Cursor Run Modes](https://cursor.com/docs/agent/security/run-modes)
- [Cursor Cloud Agent security overview](https://prod.cursor.com/docs/cloud-agent/security)
- [Anthropic: Making Claude Code safer with sandboxing](https://www.anthropic.com/research/claude-code-sandboxing)
- [Anthropic: How we built Claude Code auto mode](https://www.anthropic.com/research/claude-code-auto-mode)
- [GitHub Copilot security, governance, and network settings](https://docs.github.com/en/copilot/concepts/security-governance-and-network-settings)
- [GitHub Copilot local sandboxing](https://docs.github.com/en/copilot/concepts/security-governance-and-network-settings/about-cloud-and-local-sandboxes)
- [GitHub Copilot cloud sandboxes](https://docs.github.com/en/copilot/concepts/security-governance-and-network-settings/about-cloud-sandboxes)
- [GitHub Copilot agent session management](https://docs.github.com/en/copilot/how-tos/copilot-on-github/use-copilot-agents/manage-and-track-agents)

### Protocol, transparency, and provenance standards

- [Model Context Protocol specification](https://modelcontextprotocol.io/specification/)
- [RFC 9162 — Certificate Transparency Version 2.0](https://www.rfc-editor.org/rfc/rfc9162.html)
- [SLSA v1.2 specification](https://slsa.dev/spec/v1.2/)
- [SLSA v1.2 build provenance](https://slsa.dev/spec/v1.2/build-provenance)
- [Sigstore documentation](https://docs.sigstore.dev/)
- [NIST SP 800-218 — Secure Software Development Framework](https://csrc.nist.gov/pubs/sp/800/218/final)

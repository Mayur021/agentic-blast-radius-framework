# Agentic Blast Radius Framework

## Chapter: OSI Network-Stack Cross-Walk for Agentic Security

**Companion lens to the OS-analog framework introduced by Pirch et al. (arxiv 2605.14932, May 2026)**

---

## 1. Why an OSI-Stack Lens

The Pirch et al. paper "Toward Securing AI Agents Like Operating Systems" (2026) maps agentic security to operating-system primitives: LLM ↔ user, agent runtime ↔ kernel, tools ↔ syscalls, skills ↔ programs, LLM context ↔ memory, files ↔ storage, gateway ↔ network, cron/heartbeat ↔ scheduler. Their analysis identifies three security boundaries — **network**, **kernel**, **file system** — and shows that the four widely-used OpenClaw-style agents (OpenClaw, IronClaw, Nanobot, NemoClaw) implement most OS-style protection mechanisms only partially or not at all.

That framework is well-formed for host-side reasoning. It clarifies which agent component plays the role of kernel, which acts as a user, and where privilege boundaries should be enforced.

What it does not surface directly is the **network-stack lens** — the OSI seven-layer model that practitioners use to reason about distributed system attack surface. Agentic systems are increasingly distributed: agents call remote tools, federate across clusters, traverse trust boundaries between organizations, and persist credentials across long-running multi-hop sessions. The Pirch framework treats network egress as a single boundary mediated by the gateway. In practice the egress crosses seven distinct layers, each with its own attack vectors and its own existing controls.

This chapter introduces the OSI network-stack as a **complementary lens** on top of the OS-analog. The two are orthogonal: Pirch covers host-side privilege separation; OSI covers wire-level mediation. Together they form a more complete attack-surface model for agentic systems, and they feed directly into the **blast radius axes** introduced in earlier chapters of this framework (action class, chain depth, external reach, reversibility window, identity scope).

The OSI layers used in this chapter are the standard ISO/IEC 7498-1 layering:

| Layer | Name | Traditional concern |
| :---: | --- | --- |
| L1 | Physical | Hardware, cabling, signal |
| L2 | Data Link | MAC frames, switches |
| L3 | Network | Routing, IP, packets |
| L4 | Transport | TCP/UDP, ports, sessions |
| L5 | Session | Session lifecycle |
| L6 | Presentation | Serialization, encryption |
| L7 | Application | Application protocols |

For agentic systems, each layer carries new responsibilities and new attack vectors. The following sections walk the stack from L1 to L7, identify the agent-security concern at each layer, the attack vectors that have emerged in 2025-2026 incidents, the controls in existing frameworks that address them, and the gaps that remain.

---

## 2. Layer 1 — Physical

### Concern at this layer

Agent runtimes execute on physical hosts that the agent does not control and often does not know are present. The agent's identity, its credentials, and its memory all live on hardware whose trustworthiness has to be established before the agent's runtime decisions can be trusted. The Pirch framework treats this as the "host" axiom — outside the threat model. For distributed agentic systems running on multi-tenant cloud or edge infrastructure, the host cannot be treated as axiomatic.

### Attack vectors observed 2025-2026

- **Untrusted host substitution.** Agent runtimes deployed onto compromised or attacker-controlled hosts continue to authenticate as the same agent identity, because the identity is software-bound rather than hardware-rooted.
- **Memory inspection of ephemeral state.** Pirch identifies the ephemeral state (session store, task queue, logs) as security-relevant. On compromised hardware without memory encryption, this state is readable to a co-tenant or to a privileged host attacker.
- **Persistent state at rest.** Agent identity files (Pirch's `AGENT.md`), credentials, and configuration sit on disk. Without disk encryption rooted in a hardware key, a host-side attacker who can mount the disk reads everything.

### Existing controls

| Control | Source |
| --- | --- |
| Hardware-rooted attestation (TPM 2.0, Intel TDX, AMD SEV-SNP, ARM CCA) | TCG / Intel / AMD / ARM specs |
| Confidential compute environments for agent runtime | NIST SP 800-207 Zero Trust Architecture; SPDX 3.1 AI Profile (hardware provenance) |
| Disk + memory encryption with hardware root of trust | Standard cloud-provider primitives |
| Workload attestation tokens bound to hardware identity | SPIFFE/SPIRE attestor plugins (AWS instance identity, Azure managed identity, GCP workload identity) |

### Gaps

- **Agent identity rarely bound to persistent hardware identity.** Most OpenClaw-style runtimes generate or load a software identity at runtime. The hardware root of trust, when present, attests the host — not the agent. Without explicit binding, an attacker who can spin up an attested host can also run any agent on it with no detectable break.
- **Confidential-compute coverage for agent state is partial.** TDX/SEV protect memory contents from the host operator but do not by themselves protect against a malicious tool or skill running inside the agent's own enclave.
- **No standard schema for hardware-attested agent identity.** NIST AI 600-1 (Generative AI Profile) names hardware attestation as a control objective but does not specify the binding mechanism. WIMSE and SPIFFE work in this direction but are still drafting.

### Cross-walk to blast radius

The L1 layer feeds the **identity scope axis** (workload-bound vs federated) and the **chain depth axis** (does the chain audit terminate in a hardware-rooted principal). When L1 is missing, the lowest verifiable identity in the chain is software-issued — and the worst-case-governs rule (AISVS C9.2.7 Proposed for v1.01) becomes harder to anchor.

---

## 3. Layer 2 — Data Link

### Concern at this layer

Within a single cluster or single host, agents communicate with sidecars, tools, and other agents through direct links — Unix domain sockets, named pipes, in-process channels, intra-node service mesh routes. The data-link layer concern is whether the link itself can be trusted, whether the peer on the other end of the link is the agent the runtime thinks it is, and whether the link's data plane can be observed or modified by another tenant on the same host.

### Attack vectors observed 2025-2026

- **Sidecar bypass.** When the agent talks to a sidecar over a Unix socket, an attacker with file-system access to the socket can interpose. Pirch's case study finds that file-access control in OpenClaw-style agents runs at the same privilege level as the LLM, which means a compromised skill on the host can write to the same socket.
- **Intra-node agent-to-agent spoofing.** Two agents running on the same node may share the same workload identity attestor (e.g., they both attest as "this Kubernetes pod"). Without per-agent identity below the pod level, the agents are indistinguishable to each other from a wire perspective.
- **MAC-layer attacks on agent service mesh.** ARP spoofing and CAM-table attacks within a flat L2 network reach agent-to-agent traffic that has not been encrypted by an overlay.

### Existing controls

| Control | Source |
| --- | --- |
| Workload identity at the agent layer | SPIFFE/SVID, WIMSE workload-identity-practices |
| Mutual TLS at the workload layer | SPIRE, service mesh implementations |
| Per-agent identity below pod (one identity per agent instance, not per pod) | CoSAI WS4 Issue #99 Agent Credentials (RFC in 4-week draft window) |
| Per-link integrity protection (mTLS, IPsec, MACsec) | Standard primitives |
| L2 segmentation | Standard datacenter / Kubernetes network policy |

### Gaps

- **Per-agent identity below the pod level is still rare.** SPIFFE identities are typically issued per workload, which in Kubernetes maps to per-pod. Two agents in the same pod share an identity. The CoSAI WS4 Trust-Aware Dataplane RFC (#50, Parul Singh, accepted) addresses this with workload-identity-based authorization at hop boundaries, but the per-agent (vs per-workload) binding is still being specified.
- **Sidecar trust boundary is implicit.** Most agent runtimes treat the sidecar as trusted. Pirch shows that agent runtime, agent core, tool execution, and file access in current OpenClaw-style agents all operate within the same application-level trust domain — sidecar bypass is therefore in scope.

### Cross-walk to blast radius

L2 attacks affect the **identity scope axis** (shared vs per-agent identity) and the **chain depth axis** (whether intra-node hops in a multi-step plan can be independently attributed). They also feed the **reversibility window axis** — sidecar bypass that lets an attacker impersonate the agent leaves no audit trail at the L4+ layers, so the time-to-detect is unbounded.

---

## 4. Layer 3 — Network

### Concern at this layer

Once agent traffic leaves the host, it traverses routers and crosses administrative domains. Modern agentic systems often span multiple clusters, multiple clouds, and multiple organizations. The network layer concern is whether routes are reachable that should not be, whether segmentation between trust zones holds, and whether cross-domain traffic preserves the identity context established at L1 and L2.

### Attack vectors observed 2025-2026

- **Cross-tenant agent reach.** Agents in dev/staging environments calling production endpoints because the network policy was set to permit "all internal traffic." The PocketOS/Railway incident (April 2026) showed an agent in a staging task reaching production deletion endpoints through this exact failure mode.
- **Multi-cloud federation breaks.** Agent issued an identity in cloud A operating in cloud B with no consistent way to assume a role. Prashanth Reddy Pasham flagged this in the CSA NHI Working Draft review: "no consistent way for a workload running in one cloud to assume a role in another without intermediate static credentials."
- **DNS-based agent service discovery poisoning.** Agent runtime resolves a tool endpoint through DNS; attacker poisons the response and the agent connects to an attacker-controlled tool.
- **East-west traffic that should have been blocked.** Agent runtimes that should never talk to each other have routes between them because the segmentation policy was permissive by default.

### Existing controls

| Control | Source |
| --- | --- |
| Zero Trust network access | NIST SP 800-207 |
| Default-deny segmentation at agent runtime boundaries | CSA Microsegmentation Guidance v3.5 |
| Service-mesh based identity-aware routing | Standard service-mesh primitives |
| Workload identity federation across clouds | SPIFFE federation, WIMSE federation drafts |
| DNS-over-HTTPS / DNSSEC at agent runtime resolver | Standard primitives |

### Gaps

- **Multi-cloud agent identity federation is underspecified.** SPIFFE federation works between SPIFFE trust domains; it does not provide a portable mechanism for an agent issued in one cloud's IAM to assume a role in another. The CoSAI WS4 Identity Architecture Playbook (PR #116, Parul Singh, WIP) addresses some of this with a "Federated Identity Bridge" layer, but the bridge contract is still being drafted.
- **Default-deny segmentation for agent runtimes is rare.** Most production deployments use coarse-grained network policies that group agent runtimes with adjacent workloads. The Pirch framework's "network boundary" assumption that the gateway mediates all egress fails the moment an agent runtime makes a direct outbound connection without going through a designated gateway.

### Cross-walk to blast radius

L3 maps directly to the **external reach axis** (in-tenant vs cross-tenant vs outside-org vs third-party). Cross-tenant reach without explicit segmentation expands blast radius without any code change on the agent side. The PocketOS/Railway case is canonical: the agent's code was unchanged but the network reach grew when staging policies leaked production endpoints.

---

## 5. Layer 4 — Transport

### Concern at this layer

The transport layer governs how agent connections are established, how long they live, and how they are authenticated. For agentic systems, the L4 question becomes: is the connection bound to the identity it was opened with, can the connection be reused after the identity context changes, and does the port-level access control match the agent's declared scope.

### Attack vectors observed 2025-2026

- **Long-lived agent sessions accumulating privilege.** An agent connection opened with one set of credentials remains valid as the agent's role and the user's authorization change. AISVS C9.5.6 specifically targets this: "long-running agent sessions re-evaluate current backend authorization policy on every privileged action" (renumbered from former C9.6.7 in the 2026-06-15 PR #928 + #934 cleanup).
- **Connection reuse across delegation hops.** Agent A opens a TLS connection to a backend; agent A then delegates a task to agent B, which inherits the connection without re-attesting. The receipt-binding gap is documented in the AISVS research chapter for C9.2.3.
- **Token theft and replay over the wire.** Without channel binding, a stolen bearer token can be replayed from any client to any service.
- **Port-level exposure of agent control planes.** PraisonAI CVE-2026-44338 (May 2026): agent server bound to 0.0.0.0 with AUTH_ENABLED=False as a hardcoded default; check_auth() that fails open. The orchestrator's approval logic was unreachable on the wire path attackers actually used.

### Existing controls

| Control | Source |
| --- | --- |
| Channel-bound tokens | RFC 8705 (OAuth 2.0 Mutual-TLS Client Authentication and Certificate-Bound Access Tokens) |
| HTTP message signatures over canonical request | RFC 9421 |
| Demonstrating Proof of Possession at the application layer | RFC 9449 (DPoP) |
| Token exchange for delegation | RFC 8693 |
| Rich authorization requests (scope binding) | RFC 9396 |
| Continuous policy re-evaluation on every privileged action | OWASP AISVS C9.5.6 (Level 3 — renumbered from former C9.6.7 in 2026-06-15 cleanup) |
| Closed-by-default agent control-plane bind addresses | NIST SP 800-207 |

### Gaps

- **Connection reuse across delegation hops is undetected.** Most agent runtimes do not re-attest at hop boundaries; once a TLS connection is open, it carries whatever identity it was opened with regardless of whether the delegation context has narrowed.
- **Channel-bound tokens are used inconsistently.** RFC 8705 and RFC 9449 give the primitives, but adoption across agent runtimes and tool servers is uneven.
- **Default bind addresses on agent control planes have repeatedly defaulted to 0.0.0.0 with auth-off.** This is a long-known anti-pattern in software engineering generally; it has been observed in agent runtimes specifically through 2025-2026.

### Cross-walk to blast radius

L4 affects the **reversibility window axis** (time to detect compromised connection and revoke) and the **identity scope axis** (does the connection's identity match the action's declared scope). A long-lived connection that outlives the user session's authorization scope is a reversibility-window gap by design.

---

## 6. Layer 5 — Session

### Concern at this layer

Sessions are how agents persist context across multiple events. The Pirch framework treats session state as part of the ephemeral state of the agent. The L5 question is whether sessions are bound to identities, whether they expire, whether resumption preserves the identity context, and whether privileged actions inside a long-lived session require fresh authentication.

### Attack vectors observed 2025-2026

- **Session resumption attacks.** TLS session resumption can be spoofed if the session ticket key is leaked. For agent runtimes that use long-running sessions, the blast radius of a leaked session ticket key is the entire active session population.
- **Cross-session state leakage.** When the agent runtime shares an LLM context across sessions for cost reasons, a previous user's data leaks into a later user's session. Pirch identifies this directly: "all four agents feed trusted and untrusted data into a shared LLM context, violating the classic principle of process isolation."
- **Privileged actions without re-authentication.** A privileged action invoked in the middle of a long-lived session uses the credential issued at the session start, regardless of whether the user remains authenticated.
- **The "always-allow" wrapper attack class.** OpenClaw CVE-2026-29607 ("allow-always wrapper bypass") and the "Claw Chain" cluster (CVE-2026-44112 / 44113 / 44115 / 44118) exploit session-scoped permission caches that fail to re-evaluate on changed payloads.

### Existing controls

| Control | Source |
| --- | --- |
| Per-action policy re-evaluation in long-running sessions | OWASP AISVS C9.5.6 (Level 3 — renumbered from former C9.6.7 in 2026-06-15 cleanup) |
| Session-bound credentials with short TTL | OAuth 2.1, AISVS C9.2.3 |
| Cryptographic binding of approvals to nonce + TTL | AISVS C9.2.3 (Level 2) |
| Independent context window per agent | (former AISVS C9.8.3 — removed with C9.8 section in 2026-06-15 PR #928 + #934 cleanup; multi-agent isolation now has no v1.0 main anchor; v1.01 contribution opportunity) |
| Memory namespace isolation across agents | (former AISVS C9.8.2 — removed with C9.8 section in 2026-06-15 cleanup; v1.01 contribution opportunity) |

### Gaps

- **Session-bound credentials are inconsistent in adoption.** Many agent runtimes still issue long-lived bearer tokens at session start and reuse them across the entire session lifetime.
- **Cross-session LLM context sharing remains common.** Pirch flags this as one of the highest-impact unfixed gaps in OpenClaw-style agents.
- **No standard for session-handoff between agents.** When agent A hands off a session to agent B, the credential binding, the action class scope, and the audit trail context all transfer in implementation-defined ways. CoSAI WS4 #99 (Agent Credentials, Section 4 Deployment Patterns and Section 5 Lifecycle & Enforcement) is drafting the canonical model here.

### Cross-walk to blast radius

L5 maps to the **reversibility window axis** (session TTL is the upper bound on how long a compromised credential is useful) and the **chain depth axis** (handoff between agents in the same session is a chain hop). When sessions outlive their authorization scope, the worst-case reachable action grows even though the agent has not taken any new action.

---

## 7. Layer 6 — Presentation

### Concern at this layer

The presentation layer governs how data is serialized, encoded, and cryptographically protected on the wire. For agentic systems, L6 is where canonicalization for approval-binding lives, where payload encryption for sensitive context lives, and where the most subtle attack vectors of 2025-2026 have surfaced.

### Attack vectors observed 2025-2026

- **Canonicalization bugs in approval-binding.** AISVS C9.2.3 requires approvals to be cryptographically bound to canonicalized action parameters. The Adversa AI SymJack test (May 26, 2026) showed Claude Code displaying the pre-resolution literal path while writing to the post-resolution symlink target. Five separate controls in the agent runtime each looked at a different representation of the same write; none resolved the symlink. The fix in v2.1.129 was to display the canonical destination, not the literal argument.
- **JSON canonicalization variations.** Key ordering, Unicode normalization, redaction, diff rendering, and large payload summarization can all cause the human approver to see a different representation than the one signed.
- **Heredoc shell expansion.** OpenClaw CVE-2026-44115 (May 15 2026) exercised shell-expansion tokens inside here-documents that passed the allowlist check at the wrapper layer but resolved to blocked commands at the heredoc-expander layer. The approval receipt has to bind to the post-heredoc-rendered argument vector.
- **Loopback impersonation flags.** OpenClaw CVE-2026-44118 exercised a path where a non-owner client spoofed the senderIsOwner flag. The receipt's principal must be a cryptographically authenticated agent instance identity, not a transport-level boolean.

### Existing controls

| Control | Source |
| --- | --- |
| RFC 8785 JSON Canonicalization Scheme (JCS) | IETF |
| Canonicalized + cryptographically bound approval payloads | OWASP AISVS C9.2.2, C9.2.3 |
| Per-axis canonical resolution (symlink, heredoc, wrapper) | Implied by AISVS C9.2.2; not specified in detail |
| Agent-instance identity bound into receipts | AISVS C9.4.1, C9.4.2 |
| JWS-style envelope with `created`, `expires`, nonce | RFC 9421 / Cloudflare Visa/Mastercard agentic commerce pattern |
| Display the canonical post-resolution form to the human approver | Claude Code v2.1.129 corrective shape |

### Gaps

- **Approval binding to canonical pre-resolution form** is the recurring subtle gap. The L6 question is which representation is canonical when multiple equivalent representations exist (literal path vs resolved symlink, allowlist-wrapper-name vs post-expansion command, JSON with vs without key reordering).
- **Cross-layer canonicalization.** A receipt issued at L6 has to be replayable at the enforcement point (L7 application logic). When the enforcement point resolves the canonical form differently than the receipt issuer did, the binding breaks.

### Cross-walk to blast radius

L6 attacks affect the **action class axis** directly. The Pirch framework's threat model assumes the LLM is completely untrusted; the L6 attacks show that even when the LLM is correct and the policy is correct, the canonical representation can be made to disagree. That means an irreversible action can pass an authorization check that thinks it is gating a different action.

---

## 8. Layer 7 — Application

### Concern at this layer

L7 is where most of the existing literature on agentic security clusters. The application-layer concern is the agent's decision logic itself: tool invocation, action manifests, action class declaration, prompt template integrity, and the protocol-level interfaces (MCP, A2A, OpenAPI agent extensions) that the agent uses to act on the world.

### Attack vectors observed 2025-2026

- **Prompt injection and tool descriptor poisoning.** Pirch's threat model treats this as the "completely untrusted LLM" assumption. AISVS C9.9 (Architectural Data-Flow Isolation and Origin Enforcement) addresses the structural fix: separate plan generation from untrusted data processing.
- **Agent-supplied action classification.** If the agent both proposes and classifies an action's risk level, a manipulated agent can mislabel an irreversible action as low-risk to clear its own gate. This is the gap AISVS C9.2.6 (Proposed for v1.01) closes: declare the action class in the tool/action manifest, not at runtime.
- **Composition attacks across multi-step plans.** Individual steps each pass a low-risk gate; the composed chain reaches an irreversible high-impact outcome that no single-step gate caught. AISVS C9.2.7 (Proposed for v1.01) introduces the worst-case-governs rule: before a multi-step plan executes, the gate is set by the least-reversible, highest-blast-radius action reachable in the chain.
- **MCP tool descriptor manipulation.** A malicious MCP server returns tool descriptors that override the agent's policy or instructions.
- **A2A AgentCard spoofing.** An A2A peer presents an AgentCard claiming capabilities it does not have, or claiming an identity it has not been issued.

### Existing controls

| Control | Source |
| --- | --- |
| Pre-execution gate on privileged or irreversible actions | OWASP AISVS C9.2.1 (Level 1) |
| Approval payload bound to canonicalized parameters | OWASP AISVS C9.2.2 (Level 2) |
| Cryptographic binding of approval to identity + scope + chain ID + nonce + TTL | OWASP AISVS C9.2.3 (Level 2) |
| Manifest-declared action class drives the gate | OWASP AISVS C9.2.6 (Level 2, Proposed for v1.01) |
| Worst-case action class across the chain governs | OWASP AISVS C9.2.7 (Level 3, Proposed for v1.01) |
| Tool manifests declare privileges and side-effect level | OWASP AISVS C9.3.5 (Level 2) |
| Architectural separation of plan generation from untrusted data | OWASP AISVS C9.3.6 (Level 2 — closest equivalent after former C9.9.1 was removed in 2026-06-15 PR #928 + #934 cleanup) |
| Origin-aware policy enforcement | (former AISVS C9.9.4 / 9.9.5 — removed with C9.9 section in 2026-06-15 cleanup; no direct v1.0 main anchor; v1.01 contribution opportunity) |
| MCP authentication and authorization | MCP 2026-07-28 spec, OAuth 2.1 + RFC 9728 + RFC 8707 |
| A2A AgentCard signed with JWS + JCS + mTLS | A2A v1.0.1 |
| Verifiable credentials for agent claims | W3C VC Data Model 2.0 (Recommendation 15 May 2025), DID Core 1.0 |
| Capability attenuation tokens | Macaroons (Birgisson et al.) / Biscuits |

### Gaps

- **Cross-step composition gating remains rare in production.** AISVS C9.2.7 is Level 3 and Proposed for v1.01 — implementations are not yet broadly available.
- **Manifest-declared classification is implementation-specific.** AISVS specifies that the class belongs in the tool/action manifest as a structured property; it does not specify the wire format. CoSAI WS4 Section 5 (Credential Lifecycle & Enforcement, in the Agent Credentials RFC sub-section signup window) is one venue for converging on a wire-level schema.
- **Cryptographic chain audit at the application layer** is still emerging. Per 2026-06-15 deep audit of `/root/Defining_Non-Human_Identity.docx`: CSA NHI v1.0 anchors a four-element attribution language (delegator / agent / intent / actions) at paragraph 222 — a joint peer-review contribution with Mallikarjunarao Sunke. The full six-property chain audit schema developed in the same joint work (identity, authentication mechanism, scope, lifecycle stamp, parent-chain binding, and immutability of the originating principal as schema property) is NOT verbatim in v1.0 and is targeted for v2.0 inclusion. The schema needs to be carried in the action manifest and verified at the gate; specification of where in the manifest is still under discussion.

### Cross-walk to blast radius

L7 is where every blast-radius axis lands its enforcement decision. The action class axis is enforced by the C9.2.1 gate using the C9.2.6 manifest-declared class; the chain depth axis is enforced by the C9.2.7 worst-case rule; the external reach axis is enforced by the C9.3.5 manifest declaration of side-effect level; the reversibility window is enforced by the C9.2.3 TTL on the receipt; the identity scope is enforced by the C9.4.1 cryptographic identity of the agent instance.

---

## 9. Cross-Layer Concerns

### 9.1 Identity provenance from L1 to L7

The agent's identity has to survive every layer transition. A hardware-attested identity at L1 has to be inherited by the workload identity at L2, the federated identity at L3, the channel-bound token at L4, the session-bound credential at L5, the canonical receipt at L6, and the action manifest at L7. If any layer breaks the chain — for example, if the workload identity at L2 is per-pod rather than per-agent, or the session-bound credential at L5 is shared across agents — the identity provenance lost cannot be reconstructed at a later layer.

The Pirch framework's "agent identity" component (the `AGENT.md` file) lives at L7 in their analysis. The OSI lens shows that L7 identity is only as strong as the lowest-layer identity the chain inherits from.

### 9.2 Action-class lineage across hops

The Agentic Blast Radius framework's **action class axis** declares each action as read-only, reversible, external-reversible, or irreversible. The declaration lives in the action manifest at L7. The declaration has to survive every layer transition to be useful: it has to ride the L4 transport with channel-binding, persist through the L5 session even if the session is handed off between agents, remain canonical at L6, and be verified at L7 by the gate.

When the action class declaration is stripped at any layer in transit, the gate at the receiving end has to default to the most restrictive class — or fail closed. The current gap is that few production runtimes carry the class explicitly; the receiver re-derives the class from the action parameters and the agent's claimed risk score, which is exactly the gap AISVS C9.2.6 closes.

### 9.3 Chain audit across the stack

The full six-property chain audit schema (joint work with Mallikarjunarao Sunke; targeted for CSA NHI v2.0 inclusion — v1.0 anchors only the four-element attribution language at paragraph 222) captures identity, authentication mechanism, scope, lifecycle stamp, parent-chain binding, and immutability of the originating principal as schema property. Each property answers a question at a different OSI layer:

| Property | Layer where it is established | Layer where it is verified |
| --- | --- | --- |
| Identity | L1 (hardware) → L2 (workload) → L4 (channel-bound token) | L7 (gate) |
| Authentication mechanism | L4 / L5 | L7 (gate) |
| Scope | L7 (manifest declaration) | L7 (gate) |
| Lifecycle stamp | L5 (session TTL) | L4 (token TTL) / L7 (gate) |
| Parent-chain binding | L7 (manifest) | L7 (gate) |
| Immutability of originating principal | L1 (hardware root) → L7 (manifest) | L7 (gate) |

The audit record reconstructs the chain across all layers post-execution. Without coverage at any one layer, the reconstruction is incomplete.

### 9.4 Blast radius propagation across layers

An irreversible L7 action originating from a forged L1 identity has higher blast radius than the same action originating from a valid L1 identity, because the post-incident recovery has to invalidate every L2/L3/L4/L5/L6/L7 artifact that inherited from the forged identity. The blast radius framework's **identity scope axis** captures part of this; the OSI lens makes the per-layer recovery cost explicit.

---

## 10. Framework Cross-Walk Matrix

Coverage of each OSI layer by major frameworks. ✓ = layer is addressed; partial = some controls but not comprehensive; — = not addressed at this layer by this framework.

| Framework | L1 | L2 | L3 | L4 | L5 | L6 | L7 |
| --- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| NIST SP 800-207 Zero Trust | ✓ | ✓ | ✓ | ✓ | partial | — | ✓ |
| NIST AI 600-1 (GenAI Profile) | — | — | partial | — | partial | partial | ✓ |
| NIST IR 8596 Cyber AI Profile | partial | partial | ✓ | ✓ | ✓ | partial | ✓ |
| CSA NHI v1.0 (Working Draft) | partial | ✓ | — | ✓ | ✓ | — | ✓ |
| OWASP AISVS C9 (Proposed for v1.01) | — | partial | — | partial | ✓ | ✓ | ✓ |
| CoSAI WS4 #99 Agent Credentials | partial | ✓ | — | ✓ | ✓ | partial | ✓ |
| CoSAI WS4 #50 Trust-Aware Dataplane | — | ✓ | ✓ | ✓ | — | — | ✓ |
| CoSAI WS4 #57 / #91 AI Gateways | — | — | ✓ | ✓ | partial | partial | ✓ |
| CoSAI WS4 #52 / #116 Identity Arch Playbook | — | ✓ | partial | ✓ | ✓ | — | ✓ |
| MCP 2026-07-28 spec | — | — | — | ✓ | ✓ | ✓ | ✓ |
| A2A v1.0.1 | — | — | — | ✓ | partial | ✓ | ✓ |
| W3C VC Data Model 2.0 | — | — | — | partial | — | ✓ | ✓ |
| DID Core 1.0 | — | — | — | partial | — | ✓ | ✓ |
| SPIFFE / SPIRE | partial | ✓ | partial | ✓ | — | — | partial |
| WIMSE (drafts) | — | ✓ | — | ✓ | partial | — | ✓ |
| MITRE ATLAS | — | — | — | — | partial | — | ✓ |
| ISO/IEC 42001 | — | — | — | — | — | — | partial |
| EU AI Act Article 14 | — | — | — | — | — | — | ✓ |
| Singapore IMDA MGF Agentic v1.5 | — | — | — | — | — | — | ✓ |
| Five Eyes "Careful Adoption of Agentic AI Services" (May 1 2026) | — | — | partial | partial | partial | — | ✓ |
| Pirch et al. arxiv 2605.14932 (OS-analog) | partial | ✓ | partial | partial | ✓ | — | ✓ |

### What the matrix shows

L1 (Physical) and L6 (Presentation) are the layers least covered by current frameworks. Most frameworks cluster their attention at L4-L5-L7, mirroring the historical pattern of identity, session, and application security in the pre-agentic world.

The L1 gap is consequential. Without hardware-rooted identity, every layer above L1 inherits from a software-issued principal. The L6 gap is the more subtle one: canonicalization bugs at L6 enable approval-binding attacks (SymJack, Claw Chain) that the other layers cannot detect.

---

## 11. Gap Analysis — Where Current Frameworks Do Not Land

### 11.1 Per-agent identity below the workload boundary

SPIFFE / WIMSE / Kubernetes service accounts typically issue identity per workload (pod). When multiple agents run inside the same workload, they share identity. The blast radius of any compromised agent in a shared-identity pod extends to every other agent in the pod, because the workload-identity tokens are indistinguishable. CoSAI WS4 #99 is the most active drafting venue for closing this gap; no framework has yet shipped a per-agent identity model below the pod boundary as a standard.

### 11.2 Cross-cloud agent identity federation

SPIFFE federation works across SPIFFE trust domains. There is no portable mechanism for an agent issued in one cloud's IAM to assume a role in another cloud's IAM without an intermediate static credential. The CoSAI WS4 Identity Architecture Playbook (PR #116) is drafting the "Federated Identity Bridge" pattern; AISVS does not yet specify it; CSA NHI Working Draft mentions it but defers operational specification.

### 11.3 Canonicalization for approval binding

AISVS C9.2.2 requires canonicalized action parameters; C9.2.3 requires cryptographic binding. Neither specifies which canonical form. The SymJack and Claw Chain attack clusters demonstrated that the canonical form must be the **post-resolution** form (post-symlink, post-heredoc, post-wrapper) and that every layer that displays the action must display the same canonical form. No framework has yet given a precise specification of canonical-form scope at L6.

### 11.4 Cross-step composition gating

AISVS C9.2.7 introduces the worst-case-governs rule as Level 3 and Proposed for v1.01. Implementation is rare. The Pirch framework's case study finds that none of OpenClaw, IronClaw, Nanobot, or NemoClaw implements anything close to cross-step composition gating. The CoSAI WS4 Agent Credentials RFC Section 5 (Credential Lifecycle & Enforcement) is the venue most likely to ship a normative cross-step composition rule in 2026.

### 11.5 Chain audit schema as wire format

The full six-property chain audit schema (joint work with Mallikarjunarao Sunke; targeted for CSA NHI v2.0 inclusion — v1.0 anchors only the four-element attribution language at paragraph 222) is well-formed at the property level. The wire format — where in the action manifest, the credential, or the JWS envelope the properties live — is not yet specified. CoSAI WS4 #99 and CSA NHI v2.0 drafting window are the active drafting venues.

### 11.6 Hardware-rooted agent identity

No framework yet specifies how an agent's L7 identity inherits from an L1 hardware root of trust. NIST AI 600-1 names hardware attestation as an objective; SPIFFE/WIMSE attestor plugins provide host-level attestation; no spec yet binds the agent-instance identity to the hardware identity in a portable way. This is a strong forward direction for an IETF Internet-Draft (Q3-Q4 2026 target referenced earlier in this framework's roadmap).

---

## 12. Forward Direction

### 12.1 What this chapter contributes to the broader framework

The OSI lens complements the OS-analog (Pirch et al.) by making the network-stack attack surface explicit. Together, the two analyses cover host-side and wire-side concerns. For the **Agentic Blast Radius Framework**, the OSI cross-walk gives each blast-radius axis a per-layer enforcement decomposition:

- The **action class axis** is declared at L7 (manifest), bound at L6 (canonicalization), carried at L5 (session), transported at L4 (channel-bound), routed at L3 (segmentation-aware), linked at L2 (workload identity), and rooted at L1 (hardware).
- The **chain depth axis** is composed across L4 hops, audited at L7, and rooted in identity at L1.
- The **external reach axis** is enforced primarily at L3, declared at L7, and audited across all layers.
- The **reversibility window axis** is the TTL at L4 + L5 + L6 receipts; failure at any layer extends the window.
- The **identity scope axis** is the lowest layer at which the agent's identity is uniquely attested.

### 12.2 Open research questions

- A portable cross-cloud agent identity federation specification.
- A normative canonical-form specification for approval binding at L6.
- A wire-format specification for the full six-property chain audit schema at L7 (CSA NHI v2.0 hook).
- A hardware-rooted agent-instance identity binding spec for L1 → L7.
- A cross-step composition gating reference implementation (AISVS C9.2.7 reference).

### 12.3 Cross-framework engagement points

This chapter is intended as input to any working group, standards body, or research effort that wants to adopt or extend it. Adjacent surfaces where the substance is directly relevant:

- The CoSAI WS4 Agent Credentials RFC (#99, Section 5 Credential Lifecycle & Enforcement sub-section).
- The OWASP AISVS v1.01 train (C9.2.6 / C9.2.7 reference implementation).
- The IETF Internet-Draft on action-class authority + full six-property chain audit schema (Q3-Q4 2026 target; CSA NHI v2.0 hook).
- Working groups under CSA, OWASP, CoSAI, NIST, ISO, IETF where blast-radius modeling at the identity layer is in scope.

### 12.4 Anchor citation

Pirch, L., Horlboge, M., Großmann, P., Asif, S. M., Kireev, K., Holz, T., and Rieck, K. *"Toward Securing AI Agents Like Operating Systems."* arxiv 2605.14932 (May 14, 2026). The OS-analog framework remains the cleanest host-side analysis available in the public literature as of June 2026; this chapter's OSI lens is intended to complement, not replace, that analysis.

---

## Appendix A — Citation Discipline Notes

- **OWASP AISVS C9.2.6 and C9.2.7:** Proposed for v1.01 (verbatim from the merged PR #822 text in the AISVS `research/` directory). Not yet in `1.0/en/`. Cite with the qualifier "Proposed for v1.01."
- **CSA NHI v1.0:** Currently subtitled "Working Draft" in the docx header (verified 2026-06-11). Do not cite as a released v1.0 specification. Use "under peer review for v1.0" or "Working Draft."
- **CoSAI WS4 Agent Credentials (#99):** In 4-week initial-draft window as of 2026-06-04. Cite as "in drafting" or "active drafting in WS4."
- **CoSAI WS4 Trust-Aware Dataplane (#50):** Accepted, on focus slide. Cite as "accepted RFC."
- **CSA NHI joint contribution:** Verified 2026-06-15 against `/root/Defining_Non-Human_Identity.docx`. v1.0 anchors a four-element attribution language (delegator / agent / intent / actions) at paragraph 222 (joint Mayur + Mallikarjunarao Sunke). The full six-property chain audit schema (chain-id binding, originating-principal immutability as schema property, audit telemetry surface) is targeted for v2.0 inclusion (joint Mayur + Mallikarjunarao Sunke). Always joint-credit; never solo claim; specify which anchor (v1.0 four-element subset OR v2.0-targeted full schema).
- **Pirch et al. 2605.14932:** Anchor citation for the OS-analog framework. Cite by author and arxiv ID.

---

*End of chapter. Companion images: `osi_stack_agent_overlay.png`, `framework_crosswalk_matrix.png`.*

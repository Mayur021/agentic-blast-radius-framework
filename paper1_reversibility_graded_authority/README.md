# Paper 1: Action-Class Authority for Agentic AI

**Author:** Mayur Agnihotri (StraightArc Technologies Pvt. Ltd., India)
**Status:** Preprint
**Target venue:** IEEE Security & Privacy Magazine (Full Article, 6-8 pages)
**Version:** v2 (2026-06-15)

---

## Title

**Action-Class Authority: A Verification-Layer Pattern for Agentic AI Security**

## Abstract

Production-scale agentic AI systems fail at the same architectural seam: a prompt-injected, memory-poisoned, or supply-chain-compromised agent retains valid credentials and reasons over its own action authority. We propose **action-class authority** as a verification-layer pattern that decouples action classification from agent runtime reasoning. Four independent May 2026 publications, across academic systems security, industry framework engineering, empirical production-scale evaluation, and academic adversarial machine learning, converge on the same architectural conclusion: the boundary must move to the action layer. We document the OWASP AISVS verification requirements (C9.2.3 trusted reversibility classification, C9.2.4 runtime enforcement, C9.2.10 worst-case chain rule) that operationalize this conclusion, analyze three canonical 2026 cases (Codex docker bind-mount, Meta Instagram AI Support Assistant exploit, Red Hat npm Miasma supply chain), and present a cross-walk to eight adjacent standards bodies and contributions showing the action-class authority pattern as an emerging cross-organization synthesis. We close with five open research questions.

**Keywords:** agentic AI security, action-class authority, OWASP AISVS, manifest-declared classification, deterministic gate, reversibility-graded authority, multi-step chain rule, agent verification, capability-based security

---

## Five contributions

1. **Convergence synthesis** of four independent May 2026 publications into a single architectural conclusion, with verbatim empirical numbers cited from one source rather than re-aggregated across four.
2. **Verification standard documentation** of the OWASP AISVS C9.2 trusted reversibility classification requirement (C9.2.3, with C9.2.4 runtime enforcement) and the worst-case chain rule (C9.2.10), with worked threat model showing why the same agent cannot both propose and classify its action class. The manifest-declaration mechanism is this author's mechanism for making the classification trusted, not a separate shipped control.
3. **Worked case analysis** of three canonical 2026 cases (Codex Docker bind-mount, Meta Instagram AI Support Assistant exploit, Red Hat npm Miasma supply chain) demonstrating that the architectural pattern is identical across coding agent, customer-facing AI, and CI/CD pipeline.
4. **Cross-domain prior-art framing** of action-class authority with structural parallels in capability-based security, transactional database reversibility, formal verification of reversible computation, and digital forensic chain of custody.
5. **Standards cross-walk** to eight adjacent standards-track contributions (CSA NHI, OWASP SPVS, OWASP Cornucopia AAI, OWASP AIVSS, MITRE ATLAS, IETF AI-agent-authentication, CSA ZT Microsegmentation).

---

## Files in this directory

| File | Description | Size |
|---|---|---|
| `paper1_ieee_sp_magazine.pdf` | Compiled PDF preprint | ~210 KB |
| `paper1_ieee_sp_magazine_v2.md` | Markdown source (v2, 2026-06-15) | ~30 KB |
| `README.md` | This file | n/a |

---

## Cite this preprint

```
Agnihotri, M. (2026). Action-Class Authority: A Verification-Layer Pattern
for Agentic AI Security. Preprint.
github.com/Mayur021/agentic-blast-radius-framework/tree/main/paper1_reversibility_graded_authority
```

BibTeX:

```bibtex
@misc{agnihotri2026actionclass,
  author = {Agnihotri, Mayur},
  title  = {Action-Class Authority: A Verification-Layer Pattern for Agentic AI Security},
  year   = {2026},
  note   = {Preprint},
  url    = {https://github.com/Mayur021/agentic-blast-radius-framework/tree/main/paper1_reversibility_graded_authority}
}
```

---

## AISVS v1.0 mapping note

The OWASP AISVS controls this paper documents shipped in v1.0, released June 2026, in the C9 Orchestration & Agentic Security chapter: **C9.2.3** (a trusted reversibility classification of high-impact actions: read-only, reversible, externally reversible, or irreversible; Level 2), **C9.2.4** (runtime enforcement by that classification; Level 2), and **C9.2.10** (the highest-impact reversibility class present anywhere in a multi-step or multi-agent chain governs the gate; Level 3). The manifest-declaration mechanism, blast radius as an independent axis, and the fail-closed-for-unclassified-tools default are this author's architectural extensions, not separate AISVS v1.0 controls.

---

## Related work in this repository

- Framework overview: [`../README.md`](../README.md)
- OSI layer cross-walk chapter: [`../chapter_osi_layer_crosswalk.md`](../chapter_osi_layer_crosswalk.md)
- Framework cross-walk matrix figure: [`../framework_crosswalk_matrix.png`](../framework_crosswalk_matrix.png)
- OSI stack agent overlay figure: [`../osi_stack_agent_overlay.png`](../osi_stack_agent_overlay.png)

---

## License

See the repository root [LICENSE](../LICENSE) file.

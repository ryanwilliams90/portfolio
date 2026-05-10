# Cryptographic Provenance for AI-Assisted Code

**Status:** Architecture study

> This is a worked example of the provenance, deployment-safety, and release-traceability layer of [Calling the Model Is the Easy Part](../writing/calling-the-model-is-the-easy-part.md). It argues that once AI materially participates in generating, modifying, or reviewing code, "a human reviewed the PR" stops being a sufficient provenance model — and that the right answer is an artifact-centered signed chain, not a stronger version of the human-process audit.

## Executive summary

AI-assisted software delivery requires traceability across intent, review, build, artifact, deployment, and runtime. The provenance model should be **artifact-centered and signed**, not reconstructed after the fact from scattered human-process records (PR comments, Jira tickets, wiki pages, deployment Slack threads).

The claim is narrow on purpose. AI-generated code does not require a fundamentally different SDLC. What it requires is that the operational artifact — the signed build, the deployed container, the runtime instance — becomes the system of record, with cryptographic links back to the change that produced it. Once an AI materially influences code generation, code review, or deployment decisions, organizations need stronger artifact-level traceability than informal human review records can provide.

## The problem

Pre-AI software delivery had a workable, if loose, provenance model: a human reviewed the PR, the build system produced an artifact from that commit, the deployment system shipped that artifact to production. Where things drifted, the audit was reconstructable — Git history, build logs, and a CI/CD trail were enough to answer "what was deployed?" most of the time.

AI-assisted delivery changes the shape of the question.

When an AI participates in generating, modifying, or reviewing code, several previously implicit anchors stop holding:

- **The author of a change is no longer obvious.** A diff produced by an AI agent and lightly edited by a human is, in some sense, both authored. In a meaningful sense, neither.
- **"A human reviewed the PR" is not the same control as before.** Reviewers who know AI participated read the diff differently, and reviewers who don't know weight their attention against assumptions that no longer hold.
- **AI review outputs are themselves outputs.** A multi-model code review system that classifies findings as Consensus / Majority / Minority / Contested is producing a structured audit record. If that record is not tied to the commit, the artifact, and the deployment, it lives nowhere — and a downstream consumer asking "what was the AI review's verdict on the change that's running in production right now?" cannot get a clean answer.
- **The deployment chain is the same shape as before, but it now needs to carry more.** Build provenance, AI involvement metadata, review verdicts, approval attestations — none of that fits in a Git tag, and tags are mutable anyway.

The risk this creates is not primarily that an AI writes bad code. AI-generated code, like human-generated code, is a normal-distribution problem; bugs happen, review catches some, the rest are caught downstream. The larger operational risk is **loss of traceability across the chain**: a deployment whose AI involvement is unclear, whose review verdict is unrecoverable, whose artifact identity has drifted from the reviewed commit, or whose runtime instance can no longer be tied back to a specific change.

That risk is operational, not theoretical. It surfaces during incidents ("what changed?"), during audits ("how did this go to production?"), during rollbacks ("can we revert without unwinding state?"), and during compliance reviews ("can you produce the review record for this artifact?"). In each case, the question is fundamentally one of provenance, and the answer is fundamentally one of artifact-level identity.

## Threat model

The threat model is **loss of trust and traceability** in AI-assisted software delivery. Concrete failure modes:

- **AI-generated code reaches production without a recorded provenance chain.** AI involvement was never captured as a structured event tied to the artifact.
- **Review outputs aren't tied to deployed artifacts.** The multi-model review produced a verdict; the verdict lives in the orchestrator's database; the artifact lives in the registry; nothing links them. "What did the review say about the artifact on this pod?" requires manual reconstruction.
- **Build artifacts aren't traceable back to approved commits.** Whether an artifact's content corresponds to a *specific* approved commit becomes a property of trust in the CI system, not a verifiable property of the artifact itself.
- **Unsigned containers reach runtime.** Deployment pulls by tag, registry resolves to a digest, runtime accepts whatever the registry returned. Substitution anywhere in the chain requires noticing.
- **Process records become the "source of truth" instead of signed artifacts.** Jira tickets, wiki deployment records, Slack rollout announcements — evidence of *intent*, not of *outcome*. None are cryptographically tied to the artifact in production.
- **Audit questions require multi-system reconstruction.** "What was deployed on Tuesday at 14:00?" answerable only by joining Git history, CI logs, registry events, deployment records, and the orchestrator's database. Each join is a chance to drift.
- **Mismatch between reviewed and deployed code.** A human approved one commit; the build that shipped came from a different commit (rebase, merge resolution, hotfix on top). Same repo, different code.
- **Deployment systems trust mutable metadata.** Tags get reassigned, manifests get patched, Helm charts get edited in flight. The deployment record points at things that have changed since the deployment.
- **Rollback decisions made without knowing change class.** Stateless? Schema migration? Feature flag? Data-plane? Without that classification on the artifact itself, rollback is a coin flip.

These are the failure modes that surface in incidents, compliance reviews, and post-mortems. Not theoretical — what happens when provenance is "the human PR review plus whatever the build system logs."

## The design pattern I would reject

I would reject designs that treat **provenance as a process artifact** rather than a property of the deployed thing. Specifically:

- **"A human reviewed the PR" as the whole control.** The review may have been thorough; if it isn't bound to the artifact cryptographically, the link is convention, not enforcement.
- **Provenance in Jira, wiki, or disconnected audit notes.** Process records describe what was *supposed* to happen; they don't prove what's running.
- **AI review output not tied to commits or artifacts.** "Consensus: this change is fine" lives in the orchestrator's database. Six months later, an artifact built from that commit is in production with no path back to the verdict.
- **Build systems producing artifacts without signed provenance.** Implicit trust in "the CI system" is a blast-radius statement, not a provenance one — CI compromise invalidates every claim downstream.
- **Deployment systems trusting tags.** `image: myorg/service:v1.4.2` is a string; whatever the registry resolves it to *right now* is what ships.
- **Runtime environments that don't verify artifact identity.** The pod started; therefore the artifact is correct. No admission check, no signature verification — kubelet success is the strongest claim in the chain.
- **Rollback treated as a generic operation.** Same procedure regardless of change class. The artifact carries no information about itself, so rollback decisions happen on tribal knowledge.
- **Audit trails requiring multi-system reconstruction.** The answer takes a week and a security engineer.

The common shape: **trust accumulated by convention rather than carried by the artifact**. That worked when the SDLC was fully human and changes were rare. It does not work when AI participates and change rate goes up.

## Preferred design: signed provenance chain

The preferred shape is a single chain of cryptographically signed assertions, anchored at one end by the change's intent and at the other end by the running artifact:

```
specification / intent
  → pull request
    → AI review output(s)
      → human approval(s)
        → commit hash
          → CI build provenance attestation
            → signed artifact / container digest
              → deployment record (by digest)
                → runtime verification
```

Each arrow is a signed assertion. Each node is content-addressable. The chain is reversible: given a running artifact, you can walk backwards to the specification that produced it, with cryptographic confidence at every step.

The architectural claim is that the **signed artifact bundle becomes the system of record**, not the process artifacts that describe what was supposed to happen. Process records still exist and still matter — but they describe the chain, they don't *constitute* the chain.

Properties this design has to hold:

- **Immutable commit-to-artifact binding.** The build attestation cryptographically commits to the source commit, the build environment, and the resulting artifact digest. Tampering with any of the three invalidates the attestation. SLSA Build L3 (or equivalent) is the right shape.
- **Signed build provenance, signed artifact.** The CI system signs the attestation with a key bound to the build identity. The artifact itself is signed separately. Verifying a deployed artifact requires both checks.
- **AI involvement as a first-class attestation.** When an AI agent generated, modified, or reviewed code, that involvement is captured as a structured signed event — not a PR comment. The multi-model orchestrator's review verdict (Consensus / Majority / Minority / Contested) becomes an attestation bound to the commit, retrievable from the artifact.
- **Deployment by digest, not tag.** The deployment manifest references the content-addressable digest. Tags are human-readable convenience; the deployment is by digest.
- **Admission verification.** The cluster's admission controller refuses to admit artifacts whose signatures don't verify or whose chain doesn't end at an approved commit. SLSA, sigstore, cosign, Kyverno, OPA Gatekeeper all fit here; the tool choice matters less than that verification happens server-side, at admission.
- **Change-class classification.** The attestation carries a structured field — stateless, stateful, migration, data-plane, control-plane — that rollback systems and incident response consume.
- **Compatibility with SLSA-style provenance.** This isn't a parallel standard. SLSA defines build-time provenance levels; the architecture here extends that thinking to AI participation events and runtime verification.

## Deployment and runtime enforcement

A signed chain that nobody checks is a process artifact in disguise. Enforcement happens at four points, in decreasing priority:

- **Admission** is the primary enforcement point. The cluster's admission controller verifies the signature on every pod admission, walks the attestation chain back to a permitted source repository, and checks that the chain includes the required AI-involvement and review attestations for the change class. The cluster will not run an artifact whose chain does not verify, regardless of how it got into the registry.
- **Registry** verifies signatures on push as defense in depth. Easy to bypass with a misconfigured project; useful but not sufficient on its own.
- **Runtime** continuous verification matters for some workloads (re-checking on container restart, refusing to start if the attestation has been revoked). For most workloads, admission is sufficient.
- **Deployment system** itself signs an attestation: "deployed digest X to environment Y at time T, approval Z." The artifact in production carries this deployment attestation; "when did this artifact reach production?" gets a signed answer.

The unifying principle: **don't trust strings.** Not tags, not deployment-system records, not registry resolution, not PR descriptions. Trust signed digests, signed attestations, signed deployment records. The cryptography exists for this exact use case.

## Rollback and change-class semantics

Rollback is one of the operational surfaces where AI-assisted delivery makes existing weaknesses worse.

The traditional rollback procedure is: identify the previous artifact, redeploy it, hope. That works when changes are stateless and the system has been designed for fast revert. It fails on stateful changes (schema migrations, data backfills, irreversible operations), on changes that affect a feature flag with persistent state, on changes that interact with external systems whose state has already moved forward.

AI-assisted code changes raise the stakes: change rate goes up, individual changes may be smaller and more numerous, the per-change cost of "is this safe to revert?" reasoning has to drop or the rollback decision becomes a bottleneck.

The architectural answer is **change-class classification on the artifact**. The provenance attestation includes a structured field — `change_class: stateless | stateful | migration | data-plane | control-plane | mixed` — that the rollback system consults. Rollback semantics differ by class:

- **Stateless:** revert to previous artifact, immediate.
- **Stateful (no migration):** revert to previous artifact, possibly with feature-flag adjustment.
- **Migration:** rollback requires either a forward-only fix or a tested down-migration; the artifact's attestation includes the migration class so the on-call doesn't have to grep for it at 3am.
- **Mixed:** the worst case; requires explicit rollback design, which the artifact's attestation should reference.

Without the classification, rollback is a coin flip. With it, the on-call has a deterministic procedure tied to a specific artifact, signed at build time, not reconstructed from tribal knowledge during the incident.

## Audit model

The question the system has to answer is precise:

> *Given this running artifact, what specification, PR, AI review output, commit, build, approval, and deployment produced it?*

That question, answered cleanly, is most of what compliance, incident response, and post-mortem reviews need. Most of what makes that question hard today is that the answer is a multi-system reconstruction, joining records that may have drifted.

Under the preferred design, the answer is a single query against the artifact:

1. The runtime knows the artifact digest (from admission).
2. The artifact's signature chain produces the build attestation.
3. The build attestation produces the source commit and the build environment.
4. The commit's metadata produces the PR, the AI review verdict (signed and bound to the commit), and the human approval (signed and bound to the commit).
5. The PR references the specification or intent record.
6. The deployment record produces the deployment time, environment, and approval.

Every link is a signed assertion. The answer is mechanical to retrieve, deterministic to verify, and survives reconstruction failures in any of the originating systems — because the chain is on the artifact, not in the systems.

That's a different kind of audit posture than "we have records." It's "the artifact carries its own provenance, signed."

## Open questions

This is the section I'm being honest about. Real designs in this space have to make choices on:

- **What threshold of AI contribution requires explicit attestation?** A line of completion suggested by an IDE assistant is qualitatively different from a function generated by an agent and applied wholesale. The threshold has to be encodable into the attestation pipeline; "any AI assistance whatsoever" is a category mistake, but "only fully AI-authored functions" misses meaningful cases.
- **How should generated tests be represented?** Tests are a form of code, and AI-generated tests have their own correctness questions. Are they attested under the same chain as the production code? Separately? Does the attestation include the AI's confidence in the tests it generated?
- **What counts as meaningful human review?** A reviewer who clicked "approve" without reading is structurally indistinguishable from a reviewer who read carefully. The attestation can record *that* a human approved, not *whether* the approval was meaningful. Pretending otherwise is theater.
- **How long should provenance artifacts be retained?** Indefinitely is expensive; short-term is insufficient for compliance and post-incident review. The retention policy has to be encodable into the attestation system, not enforced by ad-hoc cleanup jobs.
- **How do you avoid making this too heavy for normal development?** Every signed attestation is a CI step, a key management decision, a developer-experience cost. Designs that are correct but unusable will be routed around. Most of the design difficulty is making the correct path the easy path.
- **How do you handle emergency fixes?** Some changes need to ship in 15 minutes during an incident. The attestation system has to accommodate emergency-class changes without creating a bypass that becomes the normal path.
- **How should stateful changes be classified?** "Stateful" is a coarse bucket. A schema migration has different rollback semantics than a feature-flag change with persistent state. The classification taxonomy has to be sharper than two values.
- **Where does provenance enforcement belong: CI, registry, admission, or runtime?** All four can enforce. They have different blast radii, performance characteristics, and failure modes. The right answer is probably "admission, primarily, with registry as defense in depth" — but the trade-offs are real.
- **How do you make audit queries simple enough that they're actually used?** A signed provenance chain that requires three weeks and a security engineer to walk is, operationally, the same as no provenance chain. The query interface — what answers what's running, in seconds, to a non-specialist — is the part of the design that determines whether the system delivers on its claim.

These are the questions a real implementation has to answer. None of them have obvious correct answers. Naming them is the start of the design conversation, not the end.

## How this fits the operational substrate

The provenance layer doesn't stand alone. It composes with the rest of the substrate this portfolio describes:

- The **multi-model code review orchestrator** ([case study 02](./02-code-review-orchestrator.md)) produces the AI review verdicts that the provenance chain needs to be bound to commits. The orchestrator's existing audit record — Consensus / Majority / Minority / Contested classifications, escalation outcomes, prompt and routing versions in effect — is the right shape to be a signed attestation against the commit. That's the integration point.
- The **agent identity and credential plane** ([case study 03](./03-agent-identity-mcp-plane.md)) is the layer that ties AI participation to a specific signed identity. When an agent calls a code-modification tool through a scoped credential, the agent's identity and the action's audit record become inputs to the provenance chain — not "AI was involved here," but "agent X, attested by gateway Y, took action Z against this commit."
- The **orchestration gateway** ([case study 01](./01-ai-orchestration-gateway.md)) is the runtime boundary that hosts the agent runtimes whose participation needs to be attested. The gateway is what makes the agent's actions externally observable; without it, the AI participation is implicit, and provenance has nothing to attest about.

The provenance layer is the consumer at the end of this chain. It's where the operational substrate's audit and identity events become a permanent, cryptographically anchored property of the deployed artifact.

## Closing

AI does not remove the need for software provenance. It makes provenance harder to fake, harder to reconstruct, and more important to design into the delivery path.

The provenance question used to be answerable, loosely, by trusting the human review and the build system. AI participation makes that loose answer worse: more changes, less obvious authorship, reviews whose verdict can no longer be reconstructed from a glance at the diff. The architectural response is not to add a stronger version of the human-process audit. It is to move the system of record from the process to the artifact — to put the cryptographic chain on the deployed thing, anchored at one end by the change's intent and at the other by the running container, with every link verifiable.

That shift is the difference between trusting that the right thing was deployed and being able to *prove* it.

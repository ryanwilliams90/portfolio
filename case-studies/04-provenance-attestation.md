# Cryptographic Provenance for AI-Assisted Code

**Status:** Architecture study, not a production deployment
**Audience:** platform engineering, application security, AI infrastructure governance
**Thesis:** AI-assisted delivery does not require a fundamentally new SDLC. It requires the operational artifact — the signed build, the deployed container, the running instance — to become the system of record for provenance.

> This is a worked example of the provenance, deployment-safety, and release-traceability layer of [Calling the Model Is the Easy Part](../writing/calling-the-model-is-the-easy-part.md). Once AI materially participates in generating, modifying, or reviewing code, "a human reviewed the PR" is no longer enough as the primary provenance claim. The durable claim has to be artifact-centered: given a running workload, the platform should be able to produce a signed chain linking it back to the intent, PR, AI participation record, human approval, source commit, build provenance, artifact digest, deployment decision, and runtime verification result.

## Concrete architecture

The full design lives in the rest of the document, but a single matrix gets the implementation shape on the page first.

| Attestation | Subject | Signer | Key fields | Enforced by |
|---|---|---|---|---|
| **AI participation** | commit / PR / patch digest | orchestration gateway or review orchestrator | agent ID, model class, tool action, prompt-policy version, output digest, review verdict | release verifier |
| **Build provenance** | image digest | CI control plane | source repo, commit, build workflow, builder ID, dependencies | release verifier · admission |
| **Artifact signature** | image digest | CI / release signer | image digest, signer identity | admission controller |
| **Change class** | image digest + source commit | release verifier or change-class classifier | classification (`stateless`, `stateful`, `migration`, `data-plane`, `control-plane`, `mixed`), migration metadata, rollback notes | rollback / incident response |
| **Release verification summary (VSA)** | image digest | release verifier | policy digest, input attestation digests, PASS/FAIL | admission controller |
| **Deployment** | image digest + environment + rollout ID | deployment controller | environment, time, approver, rollout ID, applied change class | audit query · runtime inventory |

The matrix names the operational positions. The rest of the document explains why each row exists, what it defends against, and where the design is uncertain.

## The problem

Many pre-AI delivery systems operated with a loose but often serviceable provenance model: Git history, CI logs, and deployment records were usually enough for day-to-day incident reconstruction. AI-assisted delivery changes the shape of the question.

When an AI participates in generating, modifying, or reviewing code, several previously implicit anchors stop holding:

- **The author of a change is no longer obvious.** A diff produced by an AI agent and lightly edited by a human is, in some sense, both authored. In a meaningful sense, neither.
- **"A human reviewed the PR" is not the same control as before.** Reviewers who know AI participated read the diff differently, and reviewers who don't know weight their attention against assumptions that no longer hold.
- **AI review outputs are themselves outputs.** A multi-model code review system that classifies findings as Consensus / Majority / Minority / Contested is producing a structured audit record. If that record is not tied to the commit, the artifact, and the deployment, it lives nowhere.
- **The deployment chain needs to carry more.** Build provenance, AI involvement metadata, review verdicts, approval attestations — none of that fits in a Git tag, and tags are mutable anyway.

The primary claim here is not that AI-generated code is uniquely defective. Bugs, review misses, and insecure patterns exist in both human- and AI-authored code. The architectural issue is that AI participation weakens implicit assumptions about authorship, review, and traceability — which surfaces during incidents ("what changed?"), audits ("how did this go to production?"), rollbacks ("can we revert without unwinding state?"), and compliance reviews ("can you produce the review record for this artifact?").

## Threat model

The threat model is **loss of trust and traceability** in AI-assisted software delivery. Concrete failure modes:

- **AI-generated code reaches production without a recorded provenance chain.** AI involvement was never captured as a structured event tied to the artifact.
- **Review outputs aren't tied to deployed artifacts.** The multi-model review produced a verdict; the verdict lives in the orchestrator's database; the artifact lives in the registry; nothing links them.
- **Build artifacts aren't traceable back to approved commits.** Whether an artifact's content corresponds to a *specific* approved commit becomes a property of trust in the CI system, not a verifiable property of the artifact itself.
- **Unsigned containers reach runtime.** Deployment pulls by tag, registry resolves to a digest, runtime accepts whatever the registry returned. Substitution anywhere in the chain requires noticing.
- **Process records become the source of truth.** Jira tickets, wiki deployment records, Slack rollout announcements — evidence of *intent*, not of *outcome*. None are cryptographically tied to the artifact in production.
- **Audit questions require multi-system reconstruction.** "What was deployed on Tuesday at 14:00?" answerable only by joining Git history, CI logs, registry events, deployment records, and the orchestrator's database. Each join is a chance to drift.
- **Mismatch between reviewed and deployed code.** A human approved one commit; the build that shipped came from a different commit (rebase, merge resolution, hotfix on top).
- **Deployment systems trust mutable metadata.** Tags get reassigned, manifests get patched, Helm charts get edited in flight.
- **Rollback decisions made without knowing change class.** Stateless? Schema migration? Feature flag? Data-plane? Without a signed change-class attestation bound to the deployed digest, rollback is a coin flip.

These are the failure modes that surface in incidents, compliance reviews, and post-mortems.

## The design pattern I would reject

Designs that treat **provenance as a process artifact** rather than a property of the deployed thing:

- **"A human reviewed the PR" as the whole control.** If the review isn't bound to the artifact cryptographically, the link is convention, not enforcement.
- **Provenance in Jira, wiki, or disconnected audit notes.** Process records describe what was *supposed* to happen; they don't prove what's running.
- **AI review output not tied to commits or artifacts.** "Consensus: this change is fine" lives in the orchestrator's database; six months later, the artifact built from that commit is in production with no path back to the verdict.
- **Build systems producing artifacts without signed provenance.** Implicit trust in "the CI system" is a blast-radius statement, not a provenance one — CI compromise invalidates every claim downstream.
- **Deployment systems trusting tags.** `image: myorg/service:v1.4.2` is a string; whatever the registry resolves it to *right now* is what ships.
- **Runtime environments that don't verify artifact identity.** The pod started; therefore the artifact is correct. No admission check, no signature verification — kubelet success is the strongest claim in the chain.
- **Rollback as a generic operation.** Same procedure regardless of change class. No signed change-class attestation bound to the digest, so rollback decisions happen on tribal knowledge.

The common shape: trust accumulated by convention rather than carried by the artifact. That worked when the SDLC was fully human and changes were rare. It does not work when AI participates and change rate goes up.

## Preferred design: signed provenance chain

The preferred shape is a chain of cryptographically signed assertions, anchored at one end by the change's intent and at the other by the running artifact:

```
specification / intent
  → pull request
    → AI participation attestation(s)
      → human approval(s)
        → commit hash
          → CI build provenance attestation
            → signed artifact / container digest
              → change-class attestation
                → release verification summary (VSA)
                  → deployment request / manifest by digest
                    → admission decision
                      → runtime observation / deployment attestation
```

The order matters: build, classify, verify, request, admit, observe. Admission is what gates the pod's admission to the cluster; the deployment attestation is what the deployment controller emits *after* the rollout, recording what actually happened.

A useful distinction matters here: **artifact-bearing nodes are content-addressed** (commits, image digests, signed binary blobs). **Event-bearing nodes — review verdicts, approvals, deployment decisions, runtime observations — are signed statements *over* immutable identifiers** (commit hashes, image digests, policy digests, deployment IDs). Conflating the two muddles the trust analysis; the chain is built from both kinds.

The architectural claim is that **the artifact digest becomes the lookup key for the signed provenance bundle**. Process records still exist, still matter — but they describe the chain, they don't constitute it. The chain is attached to, or indexed by, the artifact digest; nothing is embedded into the built image after build.

### Properties this design holds

- **Build provenance: SLSA Build L3 as the build-to-artifact target.** SLSA v1.2 is organized into multiple tracks; the Build Track specifically covers increasing trustworthiness of artifact build provenance through a hardened build platform, signed provenance, and tamper resistance during the build. Source-control, AI-review, approval, deployment, and runtime claims need separate attestations layered around that build provenance.
- **AI involvement as a first-class attestation, not a PR comment.** When an AI agent generated, modified, or reviewed code, that involvement is captured as a structured signed event. The multi-model orchestrator's review verdict (Consensus / Majority / Minority / Contested) becomes an attestation bound to the commit, retrievable from the artifact.
- **Wire format: in-toto Statement / DSSE envelope.** The in-toto Statement model binds an attestation to one or more subjects by digest, which is exactly the shape needed here. DSSE is the recommended envelope format — it handles canonical serialization and digital signatures around the Statement payload. cosign produces and verifies in-toto attestations and supports CUE / Rego policy validation, so the tooling for the predicate-and-subject pattern already exists. An AI-participation attestation, for example, is a Statement whose `subject` is the image (or commit) digest and whose `predicate` carries the structured event:

  ```json
  {
    "_type": "https://in-toto.io/Statement/v1",
    "subject": [
      {
        "name": "registry.example.com/payments@sha256:...",
        "digest": { "sha256": "..." }
      }
    ],
    "predicateType": "https://example.com/ai-participation/v1",
    "predicate": {
      "commit": "abc123...",
      "pull_request": "https://github.com/org/repo/pull/42",
      "agent_id": "agent://review-orchestrator/model-router",
      "model_class": "code-review",
      "action": "reviewed",
      "review_verdict": "Consensus",
      "prompt_policy_version": "2026-05-01",
      "output_digest": "sha256:..."
    }
  }
  ```

  Each row in the matrix above corresponds to a Statement of this shape, with a different `predicateType` and `predicate` schema.
- **Storage: OCI registry referrers, with documented fallback.** Attestations are stored as OCI referrers indexed by image digest. The OCI Distribution spec defines a Referrers API for discovering attestations attached to a digest; clients receiving a 404 from the Referrers API must fall back to the referrers tag schema. A real implementation should support both, plus a registry-specific or external attestation store for environments where OCI referrers aren't a usable substrate. The deployment attestation is attached to, or indexed by, the artifact digest — not embedded into the built image after the fact, which would change the digest and break artifact identity.
- **Deployment by digest, not tag.** OCI registries treat a digest as a hash of the artifact manifest or index — assumed immutable — whereas tags are mutable convenience references. The deployment manifest references the digest; the tag is human convenience.
- **Change class as a separate signed predicate.** Rollback semantics aren't really a build-provenance fact — they're a release/operations fact derived from the code, migrations, feature flags, services touched, and deployment context. The change-class attestation is produced by the release verifier (or a dedicated change-class classifier), bound to both the image digest and the source commit, with classification values `stateless | stateful | migration | data-plane | control-plane | mixed`. The build provenance proves what was built; the change-class attestation describes operational rollback semantics; the deployment attestation records which class was applied during rollout.

## Trust model

Signed assertions are only as useful as the policy that decides whose signatures count. The trust model:

- **The AI gateway** may sign AI participation events. It must not sign build provenance.
- **The CI control plane** may sign build provenance. Tenant-controlled build steps must not have access to signing material — that's the SLSA Build L3 separation between the trusted control plane and the user-controlled build script.
- **The release verifier** may sign verification summaries (VSAs).
- **The deployment controller** may sign deployment attestations.
- **Admission** trusts only configured signer/verifier identity pairs, not arbitrary valid signatures. SLSA's verification model is consistent: consumers must verify provenance against expectations, including signer identity, predicate type, subject digest, and result.

Without that separation, a signed assertion proves only that *some* key signed *some* statement about *some* digest. With it, the assertion proves that *the right party* signed *the right statement* about *the right artifact* — which is the property the chain actually depends on.

## Deployment and runtime enforcement

A signed chain that nobody checks is a process artifact in disguise. Enforcement happens at four points, in decreasing priority:

- **Admission** is the primary enforcement point — but it should not reconstruct the entire chain on every pod creation. A **release verifier** evaluates the full attestation bundle once (build provenance, AI participation events, review verdicts, approvals, policy state), emits a signed verification summary (VSA) for the image digest, and **admission verifies the image signature plus the release VSA against a pinned policy identity**. Kyverno's image validation policies show the practical admission-side shape: verify image signatures and attestation signatures, validate extracted attestation payloads against policy.
- **Registry** verifies signatures on push as defense in depth. Easy to bypass with a misconfigured project; useful but not sufficient on its own.
- **Runtime continuous verification** matters for some workloads (re-checking on container restart, refusing to start if the attestation has been revoked). For most workloads, admission is sufficient.
- **Deployment system** itself signs an attestation: image digest + environment + rollout ID + time + approver + change class. Stored as an OCI referrer or in an append-only attestation store, indexed by digest. "When did this artifact reach production?" gets a signed answer.

The unifying principle: **don't trust strings.** Not tags, not deployment-system records, not registry resolution, not PR descriptions. Trust signed digests, signed attestations, signed deployment records.

## Rollback and change-class semantics

AI-assisted code changes raise the stakes on rollback: change rate goes up, individual changes may be smaller and more numerous, the per-change cost of "is this safe to revert?" reasoning has to drop or rollback becomes a bottleneck.

The architectural answer is **a change-class attestation, signed by the release verifier and bound to the image digest and source commit**. Rollback semantics differ by class:

- **Stateless:** revert to previous artifact, immediate.
- **Stateful (no migration):** revert to previous artifact, possibly with feature-flag adjustment.
- **Migration:** rollback requires either a forward-only fix or a tested down-migration; the artifact's attestation includes the migration class so the on-call doesn't have to grep for it at 3am.
- **Mixed:** worst case; requires explicit rollback design, which the artifact's attestation should reference.

Without the classification, rollback is a coin flip. With it, the on-call has a deterministic procedure tied to a specific artifact, signed at release time, not reconstructed from tribal knowledge during the incident.

## Audit model

The question the system has to answer is precise:

> *Given this running artifact, what specification, PR, AI review verdict, commit, build, approval, and deployment produced it?*

Under the preferred design, the answer is a single query against the artifact:

1. The runtime knows the artifact digest (from admission).
2. The artifact's signature chain produces the build attestation.
3. The build attestation produces the source commit and the build environment.
4. The commit's metadata produces the PR, the AI review verdict (signed and bound to the commit), and the human approval.
5. The PR references the specification or intent record.
6. The deployment attestation produces the deployment time, environment, and approver.

Every link is a signed assertion. The answer is mechanical to retrieve, deterministic to verify, and survives reconstruction failures in any of the originating systems — because the chain is attached to, or indexed by, the artifact digest, not held only in the originating systems.

## Minimum viable implementation

A practical first version would not try to solve all provenance questions at once.

1. Build every production container through a trusted CI path that emits SLSA-style build provenance.
2. Sign the resulting image digest.
3. Deploy by digest, never by mutable tag.
4. Require admission-time verification of the image signature and a release verification summary.
5. Emit an AI participation attestation only when an AI system materially generates, modifies, reviews, or approves code-path decisions.
6. Emit a deployment attestation from the deployment controller after production rollout.
7. Provide one query: given a running pod or image digest, return the source commit, PR, review verdict, AI involvement, build, approval, deployment, and rollback class.

The goal of v1 is not complete philosophical authorship attribution. The goal is to make the production artifact *answerable*. Everything in the design above can be added incrementally on top of a v1 that gets these seven things right.

## Open questions

Real designs in this space have to make choices on:

- **What threshold of AI contribution requires explicit attestation?** A line of completion suggested by an IDE assistant is qualitatively different from a function generated by an agent and applied wholesale. The threshold has to be encodable into the attestation pipeline; "any AI assistance whatsoever" is a category mistake, but "only fully AI-authored functions" misses meaningful cases.
- **How should generated tests be represented?** Tests are a form of code, and AI-generated tests have their own correctness questions. Are they attested under the same chain as the production code? Separately? Does the attestation include the AI's confidence in the tests it generated?
- **What counts as meaningful human review?** A reviewer who clicked "approve" without reading is structurally indistinguishable from a reviewer who read carefully. The attestation can record *that* a human approved, not *whether* the approval was meaningful. Pretending otherwise is theater.
- **How long should provenance artifacts be retained?** Indefinitely is expensive; short-term is insufficient for compliance and post-incident review. The retention policy has to be encodable into the attestation system, not enforced by ad-hoc cleanup jobs.
- **How do you avoid making this too heavy for normal development?** Every signed attestation is a CI step, a key management decision, a developer-experience cost. Designs that are correct but unusable will be routed around. Most of the design difficulty is making the correct path the easy path.
- **How do you handle emergency fixes?** Some changes need to ship in 15 minutes during an incident. The attestation system has to accommodate emergency-class changes without creating a bypass that becomes the normal path.
- **How should stateful changes be classified?** A schema migration has different rollback semantics than a feature-flag change with persistent state. The classification taxonomy has to be sharper than two values.
- **Where does provenance enforcement belong?** Admission, primarily, with registry as defense in depth — but the trade-offs are real. Continuous runtime verification is the right call for some workloads.
- **How do you make audit queries simple enough that they're actually used?** A signed provenance chain that requires a security engineer and a week to walk is operationally the same as no provenance chain. The query interface — what answers what's running, in seconds, to a non-specialist — is the part of the design that determines whether the system delivers on its claim.

These are the questions a real implementation has to answer. None have obvious correct answers. Naming them is the start of the design conversation, not the end.

## How this fits the operational substrate

The provenance layer is the consumer at the end of the chain the rest of this portfolio describes:

- The **multi-model code review orchestrator** ([case study 02](./02-code-review-orchestrator.md)) produces the AI review verdicts that the provenance chain binds to commits. The orchestrator's existing audit record — Consensus / Majority / Minority / Contested classifications, escalation outcomes, prompt and routing versions — is the right shape for the AI participation attestation.
- The **agent identity and credential plane** ([case study 03](./03-agent-identity-mcp-plane.md)) ties AI participation to a specific signed identity. When an agent calls a code-modification tool through a scoped credential, the agent's identity becomes input to the provenance chain — not "AI was involved here," but "agent X, attested by gateway Y, took action Z against this commit."
- The **orchestration gateway** ([case study 01](./01-ai-orchestration-gateway.md)) is the runtime boundary that hosts the agent runtimes whose participation needs to be attested. Without it, AI participation is implicit and provenance has nothing to attest about.

This layer is where the operational substrate's audit and identity events become a permanent, cryptographically anchored property of the deployed artifact.

## Closing

AI does not remove the need for software provenance. It makes provenance harder to fake, harder to reconstruct, and more important to design into the delivery path.

The architectural response is not a stronger version of the human-process audit. It is to make the artifact digest the lookup key for the chain — to attach signed, verifiable assertions to the deployed thing, anchored at one end by the change's intent and at the other by the running container, with every link verifiable.

That shift is the difference between trusting that the right thing was deployed and being able to *prove* it.

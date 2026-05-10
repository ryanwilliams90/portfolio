# Engineering Portfolio

Architecture writeups and reference code from a Distinguished Software Engineer working on production AI infrastructure, distributed data systems at scale, and security systems architecture. Day job is at Rapid7; this portfolio reflects the broader work.

Structured as an architecture review, not a marketing site. Case studies lead with constraints, tradeoffs, and failure modes; outcomes follow from those, not the other way round. Where work is at design or proposal stage rather than shipped, that distinction is declared at the top of the document.

Companion code lives in standalone repos linked from the relevant case studies — for example, the [`orchestration-gateway-pattern`](https://github.com/ryanwilliams90/orchestration-gateway-pattern) reference implementation alongside case study 01.

## Flagship case studies

1. **[Production AI Orchestration Gateway](./case-studies/01-ai-orchestration-gateway.md)** — *Shipped.* FastAPI · CrewAI · AWS Bedrock. The async/sync executor-boundary pattern that turns a synchronous, framework-driven agent runtime into an operable production service. **Companion code:** [`orchestration-gateway-pattern`](https://github.com/ryanwilliams90/orchestration-gateway-pattern).
2. **Agent Identity and MCP Credential Plane** — *In progress.* Identity, scoping, and credential issuance for agent runtimes interacting with MCP-compatible tools.
3. **[Multi-Model Code Review Orchestrator](./case-studies/03-code-review-orchestrator.md)** — *Design.* Multi-frontier-model orchestration under enterprise Zero Data Retention constraints, with semantic finding normalization, bounded escalation, and explicit degradation states.
4. **Cryptographic Provenance for AI-Assisted Code** — *In progress.* Attestation and provenance for code produced by AI systems within an SDLC pipeline.

## Contents

- [`about.md`](./about.md) — background and areas of focus
- [`case-studies/`](./case-studies/) — flagship architecture writeups
- [`writing/`](./writing/) — technical essays and engineering notes
- [`projects/`](./projects/) — supporting project summaries
- [`diagrams/`](./diagrams/) — architecture diagrams referenced by case studies

## Supporting work

FastAPI orchestration gateways · CrewAI runtime systems · LiteLLM / Bedrock abstractions · SOC alert dispositioning · detection metadata enrichment · Apache Iceberg + Trino data platforms · retrieval-based anomaly scoring · vulnerability risk scoring.

## Status conventions

Each case study declares one of:

- **Shipped** — running in production, with the operational characteristics described.
- **Prototype** — implemented and exercised, not in production.
- **Design** — architecture and constraints worked through; implementation not yet started or in progress.

# Engineering Portfolio

Senior engineering work focused on production AI infrastructure, orchestration systems, runtime governance, secure agent execution, provenance, and distributed systems reliability.

This repository is structured as an architecture review rather than a marketing site. Case studies emphasize tradeoffs, failure modes, and operational constraints over outcomes alone. Where work is at proposal or design stage rather than shipped, that distinction is made explicit.

## Contents

- [`about.md`](./about.md) — background and areas of focus
- [`case-studies/`](./case-studies/) — flagship architecture writeups
- [`writing/`](./writing/) — technical essays and engineering notes
- [`projects/`](./projects/) — supporting project summaries
- [`diagrams/`](./diagrams/) — architecture diagrams referenced by case studies
- [`assets/`](./assets/) — supporting media

## Flagship case studies

1. **Production AI SDLC Platform Architecture** — orchestration, model routing, runtime boundaries for AI-assisted software delivery.
2. **Agent Identity and MCP Credential Plane** — identity, scoping, and credential issuance for agent runtimes interacting with MCP-compatible tools.
3. **Multi-Model Code Review Orchestrator** — routing, ensembling, and disagreement handling across model providers for code review workloads.
4. **Cryptographic Provenance for AI-Assisted Code** — attestation and provenance for code produced by AI systems within an SDLC pipeline.

## Supporting work

FastAPI orchestration gateways · CrewAI runtime systems · LiteLLM / Bedrock abstractions · SOC alert dispositioning · detection metadata enrichment · Apache Iceberg + Trino data platforms · retrieval-based anomaly scoring · vulnerability risk scoring.

## Status conventions

Each case study declares one of:

- **Shipped** — running in production, with the operational characteristics described.
- **Prototype** — implemented and exercised, not in production.
- **Design** — architecture and constraints worked through; implementation not yet started or in progress.

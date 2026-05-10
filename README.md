# Engineering Portfolio

Architecture writeups and reference code from a Distinguished Software Engineer working on production AI infrastructure, distributed data systems at scale, and security systems architecture. Currently at Rapid7.

Structured as an architecture review, not a marketing site. Case studies lead with constraints, tradeoffs, and failure modes; outcomes follow from those, not the other way round. Each case study declares its status at the top:

- **Shipped** — running in production, with the operational characteristics described.
- **Prototype** — implemented and exercised, not in production.
- **Design** — architecture and constraints worked through; implementation not yet started or in progress.

Companion code lives in standalone repos linked from the relevant case studies — for example, the [`orchestration-gateway-pattern`](https://github.com/ryanwilliams90/orchestration-gateway-pattern) reference implementation alongside case study 01.

## Case studies

1. **[Production AI Orchestration Gateway](./case-studies/01-ai-orchestration-gateway.md)** — *Shipped.* FastAPI · CrewAI · AWS Bedrock. The async/sync executor-boundary pattern that turns a synchronous, framework-driven agent runtime into an operable production service. **Companion code:** [`orchestration-gateway-pattern`](https://github.com/ryanwilliams90/orchestration-gateway-pattern).
2. **[Multi-Model Code Review Orchestrator](./case-studies/03-code-review-orchestrator.md)** — *Design.* Multi-frontier-model orchestration under enterprise Zero Data Retention constraints, with semantic finding normalization, bounded escalation, and explicit degradation states.

Additional case studies on agent identity / MCP credential planes and cryptographic provenance for AI-assisted code are in development and will be added when ready.

## Supporting work

FastAPI orchestration gateways · CrewAI runtime systems · LiteLLM / Bedrock abstractions · SOC alert dispositioning · detection metadata enrichment · Apache Iceberg + Trino data platforms · retrieval-based anomaly scoring · vulnerability risk scoring.

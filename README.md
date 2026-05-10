# Engineering Portfolio

Production AI systems are primarily infrastructure, governance, and runtime coordination problems — not model problems.

This portfolio captures my perspective on production AI infrastructure, with architecture writeups and reference implementations that ground that perspective in real systems.

Distinguished Software Engineer at Rapid7, working on production AI infrastructure, distributed data systems at scale, and security systems architecture.

## Perspective

**[Calling the Model Is the Easy Part](./writing/calling-the-model-is-the-easy-part.md)** — The industry is still optimizing the wrong layer of the AI stack. The hard engineering problems start when AI systems become operationally important and need to be governed, observed, bounded, deployed, debugged, and trusted.

## Evidence

Case studies are examples of how the ideas in the perspective piece manifest in real systems.

Each declares its status at the top — *Shipped* (running in production), *Prototype* (implemented, not yet productionized), or *Architecture study* (architecture and constraints worked through; not yet built or not yet productionized). Companion code lives in standalone repos.

- **[Production AI Orchestration Gateway](./case-studies/01-ai-orchestration-gateway.md)** — *Shipped.* FastAPI · CrewAI · AWS Bedrock. The async/sync executor-boundary pattern that turns a synchronous, framework-driven agent runtime into an operable production service. Demonstrates runtime boundaries, lifespan-scoped configuration, and three-layer observability. **Companion code:** [`orchestration-gateway-pattern`](https://github.com/ryanwilliams90/orchestration-gateway-pattern).

- **[Multi-Model Code Review Orchestrator](./case-studies/02-code-review-orchestrator.md)** — *Architecture study.* Multi-frontier-model orchestration under enterprise Zero Data Retention constraints. Demonstrates bounded escalation, explicit degradation states, semantic finding normalization, and governance of prompts and routing as versioned platform assets.

- **[Agent Identity and MCP Credential Plane](./case-studies/03-agent-identity-mcp-plane.md)** — *Architecture study.* Why agent identity is harder than service identity, what the operational threat model actually is, and the gateway-mediated scoped-execution shape that closes those failure modes. Argues that scoped execution is the operating model for agent platforms, not a security feature retrofitted later.

## Reading order

- **5 minutes:** the [perspective piece](./writing/calling-the-model-is-the-easy-part.md).
- **30 minutes:** the perspective piece, then the [orchestration gateway case study](./case-studies/01-ai-orchestration-gateway.md).
- **A full read:** all three case studies, in order.
- **Want code:** [`orchestration-gateway-pattern`](https://github.com/ryanwilliams90/orchestration-gateway-pattern) — clean-room reference implementation of the gateway pattern.

---

**Contact:** ryan90@gmail.com · [LinkedIn](https://www.linkedin.com/in/ryan-williams-066b5a18/)

**Browse:** [writing/](./writing/) · [case-studies/](./case-studies/)

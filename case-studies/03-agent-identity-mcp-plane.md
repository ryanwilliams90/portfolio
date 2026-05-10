# Agent Identity and MCP Credential Plane

**Status:** Prototype
**Companion code:** [`ryanwilliams90/agent-identity-mcp`](https://github.com/ryanwilliams90/agent-identity-mcp) — working reference implementation of the design described here. Ed25519-signed scoped credentials, verifier structurally separated from tool execution, and a runnable demo of the headline rejection (a credential issued for one action cannot be used to call another).

> This is a worked example of the identity, tool-access, and governance layer described in [Calling the Model Is the Easy Part](../writing/calling-the-model-is-the-easy-part.md). It argues that once agents can interact with real internal systems, scoped execution is the operating model — not a security feature you add later.

## Executive summary

Agents that can call tools, modify code, interact with repositories, or trigger operational workflows need a different identity model from ordinary services. The right unit of control is not just service identity. It is *user + agent + task + tool + action + scope*, evaluated at execution time, not granted broadly at install time.

This note sketches the design I would argue for: a server-side credential plane that issues short-lived, task-scoped credentials at the moment of tool invocation, with raw secrets isolated from the model context and an audit chain that ties every action back to user intent, agent reasoning, and the policy decision that allowed the action through.

## Why agent identity is different from service identity

A service usually has:

- a known owner
- a stable deployment boundary
- a predictable permission set
- a defined purpose
- a relatively predictable execution path

That makes service identity tractable. You provision a service account, scope its permissions to its job, and the question "what is this service allowed to do?" has a static answer that can be audited at provisioning time.

An agent does not behave that way. An agent may act:

- on behalf of a user
- across tools
- across repositories
- across workflows
- across reasoning steps
- with the final tool call not obvious at the start

The execution path is dynamic. The same agent, on the same prompt, can take different paths through the same tool surface based on what the model decides at each step. Authority that's broadly correct at install time may be operationally wrong at execution time, and authority that's narrowly correct for one step may be too narrow for the next.

That changes the question. It is no longer "what service is this?" It becomes:

- *Who initiated the action?*
- *What agent is acting?*
- *What task is it performing?*
- *What repository, system, or environment is in scope?*
- *What tool is being called?*
- *What authority should exist for this specific action, right now?*
- *How is the full chain audited afterwards?*

The unit of control has to shift from "agent identity" (a static fact) to "agent action" (a dynamic event). That's the conceptual move that drives the rest of the design.

## Threat model

The threat model is not only "the model leaks data." That framing is too narrow and pulls attention toward prompt-level controls when the bigger operational risk lives elsewhere.

The bigger risk is agent execution crossing trust boundaries without enough control. The concrete failure modes:

- **Over-broad OAuth grants.** Integrations install with full repo / full org / full account scopes because the OAuth flow makes narrower scopes harder to express. The grant is correct for *some* paths the agent might take, and over-broad for most.
- **Long-lived tokens.** Credentials issued at integration time and held by the agent runtime for days or weeks. Compromise of the runtime is then unbounded in time.
- **Agents holding credentials directly.** The agent process can read raw tokens. A model context that includes a credential, even briefly, is a credential that has potentially been logged, traced, or replayed into a downstream system.
- **Tool calls with more permission than the task requires.** A task that needs to read one file gets a token that can read the whole repository because the integration was installed with that scope.
- **Secrets entering prompts, traces, logs, or model context.** Once a secret is in the model's input or output, you have to assume it's been exposed to every downstream observer of that conversation — logs, traces, evaluation pipelines, debug dumps.
- **Unclear attribution.** When something goes wrong, the audit trail can't separate user intent ("the user asked for X") from agent reasoning ("the agent decided that meant calling Y") from tool execution ("Y was called with these arguments") from downstream side effect ("Y caused Z to happen in production").
- **Compromised third-party integrations.** A breached MCP-style tool server has, by default, the union of every credential ever delegated to it. The blast radius is the entire scope of every agent that ever used the tool.
- **Side effects that are hard to trace.** The agent triggers a deployment, a ticket, a Slack message, a webhook. The action's effect lives in another system, and the link back to the originating user, agent, and task is fragile.
- **Authorization at setup time, not at execution time.** The decision "this agent can do X" is made when the integration is installed. The decision "this specific action, in this specific context, is allowed right now" is not made by anything.
- **Client-side controls treated as strong trust boundaries.** The IDE or chat client enforces what tools the agent "has." A motivated attacker who's reached the runtime ignores the client and calls the underlying API directly.

These aren't theoretical. They are the natural failure modes of treating agents as if they were ordinary API clients.

## The design pattern I would reject

Designs I would push back on:

- Agents given broad standing credentials at integration time.
- An agent treated as a trusted internal service.
- Authorization performed only when the integration is installed.
- Tool access controlled mainly through IDE or chat-client configuration.
- Raw secrets available in the model context (prompts, tool definitions, traces).
- Audit logs that cannot distinguish user, agent, task, tool call, and downstream side effect.
- Credentials scoped to the integration rather than the specific task or action.
- Access granted broadly because "the agent may need it later."

These are the same class of mistake as treating agents like normal API clients. They may work in a demo. They produce the wrong operating model once agents touch repositories, CI/CD, cloud APIs, production systems, ticketing, customer data, or deployment workflows. Every later control fights the original architecture.

## Preferred shape: gateway-mediated scoped execution

The shape I would argue for is *gateway-mediated execution*. Agents do not directly hold broad credentials. Tool calls pass through a server-side control plane that evaluates each action at the moment it's attempted.

The control plane has access to:

- *User* — who initiated the work.
- *Agent* — which runtime is acting, with what attestation.
- *Task* — what the user asked for and what the agent is currently doing.
- *Repository / system* — which target system the action is against.
- *Tool* — which tool is being invoked.
- *Action* — the specific operation the tool is being asked to perform.
- *Environment* — production, staging, sandbox.
- *Policy* — the rules that govern what's allowed.
- *Risk* — derived signals (sensitivity of the target system, blast radius of the action).

For each tool call, the control plane evaluates these inputs and either issues a scoped credential, denies the call, or escalates. Credentials are:

- **Short-lived.** Measured in seconds or minutes, not hours or days.
- **Task-scoped.** Tied to the specific task the user initiated.
- **Tool-scoped.** Usable only with the specific tool being invoked.
- **Action-scoped where possible.** Read-only when read is what's needed; write only with explicit authorization.
- **Issued at execution time.** Not at install time.
- **Auditable.** Every issuance is logged with the full context that produced it.
- **Revocable.** A compromised task or agent can be cut off without revoking the broader integration.

Raw secrets do not enter the model context. The agent receives only the scoped execution capability required for the current action — typically an opaque handle the gateway resolves on the agent's behalf — not the underlying credential.

The design separates, as distinct events:

- *User intent* — what the user asked for.
- *Agent reasoning* — what the agent decided to do about it.
- *Authorization decision* — whether the policy allows it.
- *Tool execution* — the actual call.
- *Downstream side effect* — what changed in the target system.
- *Audit record* — the trail tying all of the above together.

The gateway becomes the enforcement point. It is not "a routing layer that happens to log things." It is the layer where the operational decisions about what an agent is allowed to do, right now, actually live.

## Enforcement boundary

The enforcement point has to be server-side and runtime-side, not client-side. A control that lives in the IDE, the chat client, or the agent runtime itself is, by construction, not a trust boundary against an attacker who has reached the runtime.

This is the same lesson the web platform learned about input validation: client-side controls are useful for UX, not for security. An agent platform that relies on the client to constrain what tools the agent can invoke is making the same category mistake.

The gateway holds the credentials. The gateway evaluates the policy. The gateway issues the scoped capability for the specific call. The agent never has the option of calling the underlying API with a broader credential, because it never had the broader credential in the first place.

## Audit model

For the audit trail to be operationally useful — for incident response, compliance, post-hoc reasoning about what an agent did — the system has to record, per action, attribution to:

- *User* — who initiated.
- *Agent* — which runtime, attested.
- *Task* — what the agent was working on.
- *Repository or system* — what was touched.
- *Tool* — what was called.
- *Action* — what specifically was done.
- *Policy decision* — what rule allowed it.
- *Credential issuance* — what scope was granted, for how long.
- *Result and side effect* — what happened downstream, with a link back to the originating event.

The audit record is not a side effect of the system. It is a primary output. If the system can't answer "why did agent A take action B on behalf of user C, under what authority, and what changed as a result?" then the system cannot be operated safely once it has real production reach.

This also implies the audit chain has to be cryptographically verifiable end-to-end. Otherwise the answer to "what did the agent do?" depends on which logs you trust, and that's not a tenable position once a regulator, an incident responder, or a customer is asking.

## Open questions

This is the section that's genuinely unresolved. Real designs in this space have to make choices on:

- **How granular should scopes be?** Per-tool, per-tool-method, per-target-resource, per-field? There's a real tension between expressiveness and operability — a policy language that can express fine-grained scopes is also a policy language that can become unmaintainable.
- **How do multi-step tasks work?** A single user request may produce a task tree that takes minutes to complete and touches many tools. Does each step get its own credential? Does the task hold a parent credential that issues child credentials? Where's the right level of granularity?
- **How long should task credentials live?** Long enough for a step to complete; short enough that compromise is bounded. The honest answer is "task-shaped, not time-shaped" — but mapping that to existing credential systems (which are time-shaped) is non-trivial.
- **How is delegated authority represented?** When user A asks the agent to do something on behalf of user B, the chain of who-said-what has to be representable in the credential and verifiable downstream.
- **How are emergency or admin actions handled?** Some classes of action need to bypass normal scoping (incident response). The design has to account for this without becoming a backdoor.
- **How does the developer experience stay usable?** Every layer of scoping is friction. A platform that's correct but unusable will be routed around. Most of the design difficulty is making correct behavior the easy path.
- **How do you avoid policy complexity becoming the bottleneck?** A policy engine that no one understands is a policy engine that's wrong. The right policy abstractions probably look more like role-based delegation than like rule-by-rule access control, but that's an unsolved design problem.

These are the questions a real implementation has to answer. None of them have obvious correct answers.

## How this fits the operational substrate

The identity layer doesn't stand alone. It composes with the rest of the operational substrate this portfolio describes:

- The **orchestration gateway** is the runtime boundary that hosts the identity layer's enforcement point. The gateway pattern documented in [case study 01](./01-ai-orchestration-gateway.md) is what makes "evaluate policy at execution time" structurally possible — there's a place to put the enforcement.
- The **multi-model orchestrator** ([case study 02](./02-code-review-orchestrator.md)) is an example of bounded coordination that can plug into the same identity plane: the orchestrator's adversarial prompt framings and routing decisions become events the identity plane can reason about.
- The **observability layer** is where the audit chain lives. Every credential issuance and every policy decision becomes a structured event, correlated with the request and tool-execution telemetry the gateway already emits.
- The **provenance and deployment-safety layer** (in development) is the downstream consumer: when an agent action affects a release artifact or a production change, the credential issuance and the action's audit record become inputs to the provenance chain that gates the deployment.

None of these layers is sufficient on its own. The operational story is that they compose: a gateway with no identity plane is a routing layer; an identity plane with no audit chain is a black box; an audit chain with no provenance is a log file no one reads.

## Closing

Once agents can interact with real internal systems, scoped execution is no longer a security enhancement. It is the operating model.

Identity, tool access, auditability, and governance cannot be retrofitted cleanly after adoption. If an agent platform starts with broad permissions and weak attribution, every later control fights the original architecture. The identity layer has to be designed into the runtime from the beginning, as foundational platform infrastructure — not as a feature added once the platform is in production and the failure modes have already happened.

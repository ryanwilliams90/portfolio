# Production AI Orchestration Gateway

**Status:** Shipped
**Stack:** FastAPI · CrewAI · AWS Bedrock · Kubernetes · LiteLLM-style provider abstraction

## Executive summary

A FastAPI-based orchestration gateway that exposes CrewAI agent workflows as production services backed by AWS Bedrock. The gateway separates the consumer-facing API surface from the orchestration runtime, isolates blocking agent execution behind a thread executor boundary, centralizes Kubernetes-managed secrets at lifespan startup, and abstracts model providers behind a wrapper layer.

The design treats AI orchestration as an operational infrastructure problem rather than a modeling problem. The work that converts an agent workflow into a service — runtime boundaries, secret handling, provider abstraction, deployment ergonomics — is what makes the system operable, and it is most of the system.

## Context

CrewAI workflows were originally developed as project-local runtimes — scripts and notebooks invoked directly. Production use required a service layer with stable API boundaries, predictable runtime behavior, and the operational properties expected of any internal platform: containerized deployment, centralized secret handling, metrics, and CI/CD compatibility.

The brief was narrow: build the service boundary around orchestration, without forking framework internals or coupling consumer APIs to agent execution details. Multiple CrewAI projects had to coexist behind a single gateway without sharing dependencies or configuration.

## Constraints

- **Framework lifecycle.** CrewAI projects must execute through the framework's supported `crewai run` lifecycle. Bypassing it produces subtle initialization differences that surface only under load.
- **Blocking execution.** Agent workflows are long-running and synchronous from the framework's perspective. They violate the assumptions of an async web service if invoked on the event loop directly.
- **Multi-project isolation.** Multiple CrewAI projects must run behind one gateway with isolated dependencies and configuration.
- **Secrets.** Runtime credentials are managed by Kubernetes and must be loaded centrally, not scattered across handlers.
- **Provider portability.** Bedrock is the current model backend, but provider-specific calls cannot leak into orchestration code.
- **Deployment.** Container-first, CI/CD-driven, reproducible builds.

## Architecture

The system is organized as four layers, each with a single responsibility and an explicit boundary to the next.

### 1. API gateway (FastAPI)

The consumer-facing surface. Validates requests, normalizes responses (`ORJSONResponse`), exposes metrics, and routes to orchestration entrypoints. Configuration and secret state are wired up in an async lifespan hook so the API has a fully initialized runtime by the time it accepts traffic.

The gateway never blocks the event loop. Anything that calls into the orchestration runtime crosses the executor boundary first.

### 2. Executor boundary

A `ThreadPoolExecutor` sits between async request handlers and the synchronous orchestration runtime. Blocking and long-running agent execution runs on executor threads; the API layer remains responsive and continues to serve health checks, metrics, and concurrent unrelated requests while a workflow is in flight.

This boundary also makes runtime concurrency, queuing behavior, and saturation observable as concrete properties of a known thread pool, rather than emergent properties of mixing blocking work into asyncio.

### 3. Orchestration runtime (CrewAI)

CrewAI projects execute through the framework's `crewai run` lifecycle rather than direct module imports. This preserves the framework's initialization order and avoids drift between local development and production behavior. Each project is packaged with its own dependencies; the gateway dispatches to the appropriate project entrypoint without sharing process-level state across projects.

### 4. Model provider layer (LiteLLM-style wrapper over Bedrock)

Model invocations go through a thin wrapper that follows LiteLLM-style provider abstraction patterns. Bedrock is the current backend, but call sites in orchestration code see a provider-neutral interface. This is the layer where instrumentation, retries, and (in the future) routing belong.

### Runtime configuration and secrets

Secrets are loaded once during application lifespan from Kubernetes-mounted sources and injected into shared runtime state. Handlers and orchestration code read from that state — they do not reach into the environment, the secret store, or the filesystem on each request. This keeps runtime initialization predictable and makes secret usage centrally auditable.

## Key engineering decisions

### Separate the API surface from the orchestration runtime

The most important boundary in the system is the one between consumer-facing API concerns (validation, error shapes, metrics, auth) and orchestration internals (CrewAI lifecycle, agent state, model calls). Collapsing them — exposing CrewAI primitives directly through HTTP — would have coupled the API contract to framework internals and made provider or framework changes consumer-visible.

### Use an executor boundary for blocking orchestration

Agent workflows do not fit the async-everywhere model. Rather than trying to async-ify a synchronous framework, blocking work runs on a thread pool with explicit concurrency limits. The asyncio layer stays responsive, and the runtime's saturation behavior is a property of a configured thread pool rather than something that emerges under load.

### Respect the framework lifecycle

CrewAI projects run through `crewai run`, not via direct imports. Framework lifecycle hooks exist for reasons that are not always obvious until production — initialization order, plugin registration, telemetry setup. Bypassing them is cheap in development and expensive once a workload is operational.

### Abstract the provider, don't embed it

Bedrock-specific behavior lives behind a wrapper that follows LiteLLM-style patterns. Orchestration code calls a provider-neutral interface. This is not speculative portability — it is where instrumentation, error normalization, and retry logic naturally belong, and keeping it in one place avoids the drift that occurs when each call site invents its own.

### Centralize secret loading at lifespan

Secrets are loaded once at startup, not per-request and not lazily inside handlers. This makes runtime initialization deterministic, removes a class of latency variance, and gives a single place to audit how credentials enter the process.

## Operational considerations

- **Long-running requests.** Workflow execution times exceed standard HTTP timeout expectations. The gateway's request lifecycle, client-side timeouts, and load balancer settings have to agree, or a long-running workflow appears to fail from one side while still executing on another.
- **Orchestration timeouts.** The runtime needs an explicit time budget separate from the HTTP timeout. A workflow that runs forever is worse than one that fails fast.
- **Model invocation latency.** Bedrock latency dominates end-to-end response time for most workflows. The wrapper layer is the right place to surface per-call latency, token counts, and retry counts to the metrics surface.
- **Runtime concurrency.** Concurrency is bounded by the executor's pool size, not the event loop. Pool sizing is a deployment-time decision driven by per-workflow memory and Bedrock concurrency limits, not by request volume alone.
- **Deployment reproducibility.** Each CrewAI project ships in a container with pinned dependencies. Multi-project coexistence in one gateway works because nothing is shared at the dependency level.
- **Metrics surface.** The gateway exposes request-level metrics (latency, status, concurrency) and orchestration-level metrics (workflow duration, model call count, executor saturation). The two together are necessary; either alone gives a misleading picture under load.
- **Failure modes that matter.** Executor saturation manifests as queueing latency rather than errors. Bedrock throttling manifests as elevated tail latency before it manifests as failure. Both need to be visible in dashboards before they become incidents.

## Lessons learned

- **Production AI systems are operational infrastructure problems.** The model is the smallest part of the surface area. Boundaries, lifecycles, secrets, observability, and deployment are most of the work.
- **Async assumptions don't transfer to agent runtimes.** Blocking orchestration on the event loop is the kind of mistake that looks fine in development and causes head-of-line blocking in production. The executor boundary is non-negotiable.
- **Framework lifecycle bypasses are debt.** Running a framework through its supported entrypoint is rarely the most elegant option, and almost always the right one.
- **Provider abstraction earns its keep early.** Even without a second provider, the wrapper layer is where instrumentation, error normalization, and retries belong. Embedding Bedrock calls inline scatters those concerns and makes them inconsistent.
- **Centralized secret loading is a reliability decision.** Per-request secret access is a per-request failure mode and a per-request latency tail.

## Future improvements

Extensibility points the design accommodates but the current implementation does not yet exercise:

- **Structured execution tracing.** Per-workflow trace context propagated through the executor boundary into the runtime and the model wrapper, so a single trace covers the full lifecycle.
- **Distributed task execution.** Moving the executor boundary out of process — to a queue-backed worker pool — for workloads that exceed single-node capacity.
- **Provider-aware failover and routing.** Using the wrapper layer to route by latency, cost, or capability across providers.
- **Request queuing and prioritization.** Explicit admission control at the gateway, with priority classes for different workload types.
- **Cost-aware routing.** Per-call cost accounting in the wrapper layer, feeding routing decisions and budget enforcement.
- **Richer orchestration observability.** Per-step telemetry from inside the agent runtime, surfaced through the same metrics pipeline as gateway and provider metrics.

## Suggested diagram

A layered architecture diagram. Left-to-right primary flow:

```
Client → FastAPI Gateway → Executor Boundary → CrewAI Runtime → Model Wrapper → AWS Bedrock
```

Annotate each arrow with what crosses it:

- Gateway → Executor: synchronous handoff, bounded by pool size.
- Executor → Runtime: `crewai run` lifecycle entry.
- Runtime → Wrapper: provider-neutral model call.
- Wrapper → Bedrock: provider-specific invocation, instrumented.

Side panels (drawn as adjacent boxes connecting into the main flow):

- **Kubernetes secrets** → loaded once at FastAPI lifespan startup, injected into shared runtime state.
- **Metrics / logging** → fed by gateway, executor, and wrapper layers; emphasize that all three contribute.
- **CI/CD + container runtime** → produces the deployable image; per-project dependency isolation noted.

The diagram's job is to make the runtime boundaries visible. The reader should be able to point at exactly where async ends, where blocking begins, and where provider-specific code is contained.

---

### Resume summary

Designed and maintained a FastAPI orchestration gateway for CrewAI agent workloads on AWS Bedrock. Established the runtime boundary between async API handling and synchronous agent execution via a thread-executor isolation pattern, centralized Kubernetes secret loading at application lifespan, and introduced a LiteLLM-style provider wrapper to keep Bedrock-specific behavior out of orchestration code. The platform supports multiple CrewAI projects with isolated dependencies, container-first deployment, and an operational metrics surface spanning gateway, executor, and provider layers.

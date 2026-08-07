# Production Notes — What Changes at Real Organizational Scale

This project makes deliberate scope decisions to stay learnable and buildable solo. This document is your answer key for the inevitable interview question: **"What would you do differently for a production system with a real team and real scale?"** Being able to name these — and explain *why* they weren't needed for v1 — is a stronger signal than having built them all, because it shows judgment about scope, not just technical breadth.

## Auth
- **Now:** 30-minute JWT, no refresh token, re-login required.
- **At scale:** rotating refresh tokens in httpOnly cookies (advanced prep Q43), a revocation list (Redis) for logout-before-expiry, and SSO/OIDC for enterprise customers (advanced prep Q57).

## Multi-tenancy
- **Now:** single-user personal boards.
- **At scale:** a `tenant_id` on every document, enforced at the query layer (not just the app layer) via MongoDB's field-level access patterns, or a schema-per-tenant approach if isolation requirements are stricter (e.g., enterprise clients requiring dedicated infrastructure).

## Agentic AI
- **Now:** hand-rolled ReAct loop in `agent.py`, ~150 lines, so every mechanism is visible.
- **At scale:** migrate to **LangGraph** — the 2026 industry-standard runtime for stateful, production agents (durable checkpoints, human-in-the-loop primitives, built-in state persistence). The migration is mechanical: our `tools.py` definitions map directly to LangGraph tool nodes; our manual "loop until done or max steps" becomes a `StateGraph` with conditional edges. Because we built the raw version first, you can explain BOTH what LangGraph is doing for you AND how to debug it when the abstraction leaks — a stronger interview position than only knowing the framework.
- **Vector search:** our in-memory cosine-similarity index (`rag.py`) works for one resume per user. At the scale of "search across all users' documents" or "millions of chunks," you'd move to a real vector database (Pinecone, Weaviate, or MongoDB Atlas's own vector search feature — notably, staying in MongoDB Atlas for vector search would avoid adding a whole new infrastructure dependency, which is itself worth mentioning).

## Real-time
- **Now:** WebSocket connections held in a single backend process's memory.
- **At scale:** once running multiple backend instances behind a load balancer, you need a shared pub/sub layer (Redis Pub/Sub, or a managed service) so a broadcast from instance A reaches a client connected to instance B (advanced prep Q38's scaling note, applied here).

## Observability
- **Now:** basic `print`/logging statements.
- **At scale:** structured logging (JSON logs) shipped to a log aggregator, distributed tracing (a trace ID threaded through REST → service → DB → AI call, per advanced prep Q64), and dashboards on p95/p99 latency, not averages.

## Rate limiting & abuse prevention
- **Now:** none — acceptable for a personal single-user tool.
- **At scale:** per-user rate limits on the AI chat endpoint specifically (AI calls cost real money per the advanced prep's Q55 cost-optimization discussion), and general API rate limiting (e.g., via a reverse-proxy layer or `slowapi`) to prevent abuse.

## Testing
- **Now:** core unit + integration tests covering the Must-Have features.
- **At scale:** contract tests between frontend and backend (catching a breaking API change before deploy), load testing the AI endpoint specifically (since it's the slowest, most expensive path), and a staging environment mirroring production for pre-release validation.

## CI/CD
- **Now:** a single GitHub Actions workflow running tests.
- **At scale:** separate build/test/deploy stages, canary or blue-green deployment for the backend (so a bad deploy affects a small percentage of traffic before full rollout), and automatic rollback on elevated error rates post-deploy.

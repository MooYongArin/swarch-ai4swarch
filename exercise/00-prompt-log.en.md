**Language:** [ภาษาไทย](00-prompt-log.md) | English

# Prompt log

**Name / Student ID:**
**Agent(s) used:** e.g. GitHub Copilot (Agent mode), Gemini Code Assist, Gemini CLI, Claude Code, Codex, Cursor, Cline: …
**Model(s):** …
**Plan / tier:** e.g. Copilot Student, Copilot Free, Gemini free tier: …

Log every request you send to the agent. "Files the agent read" is what the agent reported, or what you saw it open.

| # | Task / run | Agent + model | Message sent (short) | Files the agent read | Stayed within the allowed evidence? | Useful? (1–5) | Notes |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code, Claude Opus 5.5 (claude-opus-5-5) | `prompts/1-…` run A, evidence: `protos/demo.proto` | `prompts/1-recover-architecture.md`, `system/online-boutique/protos/demo.proto` | Yes | 3 | Output: `exercise/outputs/1A-architecture.md`. 9 services found, 0 dependencies (proto shows no callers); frontend, languages, data stores marked "Not evidenced"; likely calls labelled [Inferred] |
| 2 | 1-B | Claude Code, Claude Opus 5.5 (claude-opus-5-5) | `prompts/1-…` run B, evidence: `protos/demo.proto` + `kubernetes-manifests/` | `prompts/1-recover-architecture.md`, `system/online-boutique/protos/demo.proto`, all 13 files in `system/online-boutique/kubernetes-manifests/` (README.md, kustomization.yaml, 11 service *.yaml) | Yes (only listed the manifests folder to find its files) | 3 | Output: `exercise/outputs/1B-architecture.md`. 10 gRPC services + frontend as HTTP entry (frontend-external LB), optional loadgenerator, redis-cart (Redis, emptyDir, so data is not persistent). 17 dependencies, mostly from `*_SERVICE_ADDR` env vars. Flagged `shoppingassistantservice` address with no manifest. Languages still not evidenced |
| 3 | 2-a | Claude Code, Claude Opus 5.5 (claude-opus-5-5) | `prompts/2-…` run a, context: none, requirement: R-12 (first attempt sent with placeholder; agent asked for the input, then resent) | `prompts/2-write-adr.md`, `docs/adr/template.md` | Yes (read nothing from `system/`) | 3 | Output: `exercise/outputs/2a-adr.md`. Options: A direct sync calls, B event + broker (chosen), C orchestrator. Broker technology left undecided; open questions list missing latency/availability targets, MailFast limits, team broker skills, deciders, privacy, whether fraud scoring must block orders. Current flow marked [Assumption]/[Inferred] |
| 4 | 2-b | Claude Code, Claude Opus 5.5 (claude-opus-5-5) | `prompts/2-…` run b, context: `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `src/checkoutservice/main.go`, requirement: R-12 | `prompts/2-write-adr.md`, `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `system/online-boutique/src/checkoutservice/main.go` | Yes | 4 | Output: `exercise/outputs/2b-adr.md`. Chose RabbitMQ (options: sync chain, goroutines, RabbitMQ, Kafka). Used scenario numbers (500 ms, 200 orders/s, MailFast 50/s). Invented: "no team Kafka experience", "Locust" |
| 5 | 3 | Claude Code, Claude Opus 5.5 (claude-opus-5-5) | `prompts/3-…`, adr: `docs/adr/0002-publish-order-events-via-rabbitmq.md` | `prompts/3-review-adr.md`, `docs/adr/0002-publish-order-events-via-rabbitmq.md`, `docs/scenario.md` | Yes | 5 | Output: `exercise/outputs/3-review.md`. Verdict: accept with changes. Found charge-then-fail risk, ADR inconsistencies (review ran before all my edits were applied), email consumer concurrency (~75 in flight), missing dedup for email/analytics, missing "PostgreSQL as queue" option |
| 6 | | | | | | | |
| 7 | | | | | | | |

> Written by: Claude Code (VS Code extension), model Claude Opus 5.5 (`claude-opus-5-5`)

# ADR-R01: Use synchronous gRPC with one shared `demo.proto` contract for all service-to-service calls

- **Status:** Reconstructed
- **Date:** Not evidenced in the provided files. Copyright headers say 2018 (`src/checkoutservice/main.go`, `src/frontend/main.go`) and 2020 (`protos/demo.proto`), so the decision dates from 2018–2020 at the latest **[Inferred]**.
- **Deciders:** Not evidenced. The copyright holder is Google LLC (header of all three files); who made the decision is unknown.

## Context and problem

Online Boutique is split into many small services that must call each other to serve a single shop request:

- `protos/demo.proto` defines **9 gRPC services** in one file, under one package `hipstershop`: `CartService`, `RecommendationService`, `ProductCatalogService`, `ShippingService`, `CurrencyService`, `PaymentService`, `EmailService`, `CheckoutService`, `AdService`.
- Shared message types are reused across services: `Money`, `CartItem`, `Address`, `Empty`, `CreditCardInfo`, `OrderResult` (`demo.proto`). For example, `Money` is used by `Product`, `GetQuoteResponse`, `ChargeRequest` and `OrderItem`.
- `frontend` opens gRPC client connections to 7 services (`frontendServer` struct and `mustConnGRPC` calls, `src/frontend/main.go` lines 141–147) and serves end users over HTTP (`http.ListenAndServe`, line 171).
- `checkoutservice` opens gRPC connections to 6 services (`mustConnGRPC`, `src/checkoutservice/main.go` lines 118–123). `PlaceOrder` calls them **one after another and waits for each reply**: cart → product catalog + currency (per item) → shipping quote → currency → payment → shipping → empty cart → email (`PlaceOrder`, `prepareOrderItemsAndShippingQuoteFromCart`, lines 230–311).
- No message broker, queue or event type appears in any of the three files.

The problem: how should these services communicate, and how should the interfaces between them be defined and shared?

## Decision drivers

Ordered by likely importance. Since there is no original ADR, the order is **[Inferred]**.

1. **Modifiability / interoperability:** one strongly typed contract that clients and servers can generate code from.
2. **Simplicity of the request flow:** a user's request (for example "place order") needs an immediate answer (`PlaceOrderResponse` returns `OrderResult`).
3. **Observability:** calls must be traceable end to end (`otelgrpc` stats handlers on both client and server).
4. **Performance:** low per-call overhead, because one checkout causes many internal calls.
5. **Operability on Kubernetes:** standard health checking (`grpc_health_v1` registered in `checkoutservice`).

## Considered options

1. **Synchronous gRPC with a single shared `demo.proto`** (what the code shows)
2. Synchronous gRPC with **one `.proto` file per service**, owned by each service team
3. Synchronous **REST/JSON over HTTP** (for example with OpenAPI specs)
4. **Asynchronous messaging** through a message broker (events/commands), at least for side effects such as email and shipping

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. gRPC + one shared proto | One source of truth; shared types (`Money`, `Address`) defined once; typed generated stubs (`pb.New*ServiceClient`); compact binary encoding; built-in health protocol and OTel instrumentation | Every service depends on one file, so a change in one service's API touches the shared contract (high coupling at design time); synchronous chains couple availability and add up latency; gRPC is not callable directly from a browser, so a gateway (`frontend`) is still needed |
| 2. gRPC + proto per service | Clear ownership; services version their APIs independently | Shared types (`Money`, `Address`) must be duplicated or put in a common proto anyway; more files and build steps to manage |
| 3. REST/JSON | Universal tooling, easy to debug with curl and browsers, no code generation required | Weaker typing unless OpenAPI is enforced; larger payloads and slower parsing; health checks and tracing have to be set up per framework |
| 4. Async messaging | Decouples availability (email/shipping can fail or be slow without blocking checkout); absorbs load spikes | Adds a broker to run and monitor; eventual consistency; harder to return an immediate `OrderResult`; more complex debugging |

## Decision

**Chosen: Option 1.** All internal calls use synchronous unary gRPC, and every interface is defined in the single file `protos/demo.proto`.

Likely reasons **[Inferred]**:
- Defining all services in one file makes the whole system's interface visible in one place, which suits a demo or reference application, and lets cross-cutting types like `Money` be shared without import plumbing.
- Synchronous request/response matches the shop's user flows, which all need an immediate reply, and keeps `PlaceOrder` a plain sequential function.
- gRPC gives standard health checks and OpenTelemetry tracing across languages with little code (`otelgrpc.NewServerHandler`, `otelgrpc.NewClientHandler`, `healthpb.RegisterHealthServer`).

## Consequences

- **Positive:** One contract, generated clients/servers, compile-time type checking of calls (quality attribute: modifiability, interoperability).
- **Positive:** Distributed traces across the call chain, because context is propagated on every gRPC call (`otel.SetTextMapPropagator`, `otelgrpc` handlers) (quality attribute: observability).
- **Positive:** Compact binary protocol over persistent connections created once at startup (`mustConnGRPC` in `main`) (quality attribute: performance).
- **Negative:** Checkout latency is the **sum** of ~8+ sequential calls, plus 2 calls per cart item (`prepOrderItems` loop: `GetProduct` + `Convert`) (quality attribute: performance).
- **Negative:** Checkout availability depends on all of cart, catalog, currency, shipping and payment being up; any one failing fails the order (`return nil, status.Errorf(...)` in `PlaceOrder`) (quality attribute: availability).
- **Negative:** The shared proto is a design-time coupling point; every service compiles against the same `hipstershop` package (quality attribute: modifiability, independent deployability).
- **Negative:** Connections use plaintext (`insecure.NewCredentials()` in `mustConnGRPC` in both files) (quality attribute: security).
- **Risks and mitigations:**
  - *Cascading slowness:* no per-call deadline is set in `PlaceOrder`, and `convertCurrency` uses `context.TODO()` instead of the request `ctx` (`src/checkoutservice/main.go` line 360), so that call ignores caller cancellation and is a gap in trace propagation. Mitigation: pass `ctx` through and set deadlines/retries.
  - *Partial failure:* the card is charged before shipping (`chargeCard` then `shipOrder`); if `ShipOrder` fails the payment is not reversed (no compensation in `PlaceOrder`). Mitigation: a saga or compensating refund.
  - *Non-critical calls already treated as best-effort:* the `EmptyCart` error is discarded (`_ = cs.emptyUserCart`) and an email failure is only logged (`log.Warnf`). These are candidates for moving to async messaging (Option 4).
  - *Proto evolution:* use only additive, backward-compatible field changes. Field number 4 is skipped in `PlaceOrderRequest` (`email = 5`), which may be a removed field **[Inferred]**.
  - *Not every dependency is gRPC:* `frontend` reads `SHOPPING_ASSISTANT_SERVICE_ADDR` but makes no `mustConnGRPC` call for it (lines 139–147), and that service is not in `demo.proto`. So the "all services use gRPC" rule has at least one exception; its protocol is not evidenced in the provided files.

## Validation

- Load test of `/cart/checkout` that measures p95/p99 `PlaceOrder` latency against cart size, to confirm the sequential chain fits the latency budget.
- Fault injection (stop `currencyservice` or `shippingservice`) to measure how checkout availability is affected.
- Review traces to confirm every hop appears (and to detect the `context.TODO()` gap).
- Review the decision again if services are owned by separate teams, or if `demo.proto` changes often break unrelated services.

## Claim check

| # | Claim | Type | Evidence |
|---|---|---|---|
| 1 | All service interfaces are defined in one proto file, package `hipstershop` | Evidenced | `protos/demo.proto` (`package hipstershop`, 9 `service` blocks) |
| 2 | Shared message types (`Money`, `CartItem`, `Address`, `Empty`) are reused across services | Evidenced | `demo.proto` (`Money` used in `Product`, `GetQuoteResponse`, `ChargeRequest`, `OrderItem`) |
| 3 | All RPCs are unary request/response (no streaming) | Evidenced | `demo.proto`: no `stream` keyword in any `rpc` |
| 4 | `frontend` calls 7 backends over gRPC | Evidenced | `src/frontend/main.go` `mustConnGRPC` lines 141–147 |
| 5 | `frontend` serves users over HTTP | Evidenced | `src/frontend/main.go` `http.ListenAndServe`, `mux.NewRouter` |
| 6 | `checkoutservice` calls 6 backends over gRPC | Evidenced | `src/checkoutservice/main.go` `mustConnGRPC` lines 118–123 |
| 7 | `PlaceOrder` makes calls sequentially and blocks on each reply | Evidenced | `src/checkoutservice/main.go` `PlaceOrder`, `prepareOrderItemsAndShippingQuoteFromCart` |
| 8 | No message broker is used | Evidenced (within scope) | No broker client or import in the three files; other services were not checked |
| 9 | Each service gets its own generated copy of the proto code | Evidenced (partly) | `checkoutservice` imports `.../src/checkoutservice/genproto`; how it is generated is not evidenced |
| 10 | gRPC was chosen for tracing and health checks | Inferred | Usage shown by `otelgrpc` handlers and `healthpb.RegisterHealthServer`; the rationale is not stated |
| 11 | One shared file was chosen for visibility and type reuse in a demo app | Inferred | Not stated in any file |
| 12 | Synchronous calls were chosen because user flows need immediate replies | Inferred | `PlaceOrderResponse` returns `OrderResult` synchronously; the motive is not stated |
| 13 | Options 2–4 were considered | Assumption | No record of alternatives in the evidence |
| 14 | gRPC/protobuf is more compact and faster than REST/JSON | Assumption | General industry knowledge |
| 15 | Async messaging would decouple availability but add eventual consistency | Assumption | General industry knowledge |
| 16 | gRPC cannot be called directly from browsers, so a gateway is needed | Assumption | General knowledge; consistent with `frontend` serving HTTP |
| 17 | Checkout latency grows with the number of cart items | Evidenced | `prepOrderItems` loop calls `GetProduct` + `convertCurrency` per item |
| 18 | Checkout fails if any critical dependency fails | Evidenced | `PlaceOrder` returns errors from prep, `chargeCard`, `shipOrder` |
| 19 | Email and empty-cart are best-effort | Evidenced | `_ = cs.emptyUserCart(...)`; `log.Warnf` on `sendOrderConfirmation` error |
| 20 | Payment is not reversed if shipping fails | Evidenced | `PlaceOrder`: `chargeCard` before `shipOrder`, no refund call |
| 21 | `convertCurrency` drops the request context | Evidenced | `src/checkoutservice/main.go` line 360 `context.TODO()` |
| 22 | Internal gRPC traffic is unencrypted | Evidenced | `insecure.NewCredentials()` in `mustConnGRPC` (both files) |
| 23 | The shared proto creates design-time coupling between services | Inferred | Follows from claim 1; impact on teams is not shown |
| 24 | Skipped field number 4 in `PlaceOrderRequest` is a removed field | Inferred | `demo.proto` `PlaceOrderRequest` (`address = 3`, `email = 5`) |
| 25 | Shopping assistant is not a gRPC dependency of `frontend` | Evidenced (partly) | `SHOPPING_ASSISTANT_SERVICE_ADDR` is read but has no `mustConnGRPC` call and is not in `demo.proto`; its actual protocol is not evidenced |
| 26 | Decision date 2018–2020, made by Google | Inferred | Copyright headers only |
| 27 | Connections are created once at startup and reused | Evidenced | `mustConnGRPC` called in `main`; `*grpc.ClientConn` fields reused in handlers |

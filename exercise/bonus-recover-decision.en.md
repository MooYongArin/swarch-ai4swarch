**Language:** [ภาษาไทย](bonus-recover-decision.md) | English

# Bonus (optional, take-home): Recover an unwritten decision (approximately 15 min)

> **Lecture link:** T5 (Dhar et al. generate decisions from code and diagrams to reduce *knowledge vaporization*), AICH4.
> **Question:** when AI explains *why* a system was built this way, how much of the explanation is evidence and how much is a confident guess?

## Run

Start a new chat or session in your agent and send:

```
Follow prompts/bonus-recover-decision.md
decision: Why do all services communicate synchronously over gRPC, using one shared demo.proto?
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/src/checkoutservice/main.go, system/online-boutique/src/frontend/main.go
```

Want the ADR in Thai? Add a line `language: Thai`.

The agent writes the reconstructed ADR, including its **Claim check** table, to [`outputs/bonus-recovered-adr.md`](outputs/bonus-recovered-adr.md).

## Audit the claim check

Choose **five** claims from the agent's table: at least two it marked *Evidenced* and two it marked *Inferred*. Check each one yourself.

| # | Claim (short) | AI says | You say | Why (file path or reasoning) |
|---|---|---|---|---|
| 1 | #3: All RPCs are unary request/response (no streaming) | Evidenced | Evidenced (agree) | `protos/demo.proto`: Ctrl+F "stream" gives 0 matches; all 15 `rpc` definitions take one request and return one response |
| 2 | #21: `convertCurrency` drops the request context (`context.TODO()`, line 360) | Evidenced | Evidenced (agree) | `src/checkoutservice/main.go` L359: function receives `ctx context.Context`; L360: calls `Convert(context.TODO(), …)` instead of `ctx`, so cancellation and trace context are not passed on |
| 3 | #20: Payment is not reversed if shipping fails | Evidenced | Evidenced (agree) | `src/checkoutservice/main.go` `PlaceOrder`: `chargeCard` at L252, then `shipOrder` at L258; on shipping error L259–260 only returns `codes.Unavailable` with no refund call; "refund" has 0 matches in the file |
| 4 | #11: One shared proto file was chosen for visibility and type reuse in a demo app | Inferred | Inferred (agree) | `protos/demo.proto` comments are only the license header, section dividers and field descriptions; no file states why one shared file was used. The single file and reused `Money` type are facts, but the motive is a guess (the name "demo.proto" hints at a demo but doesn't prove intent) |
| 5 | #24: Skipped field number 4 in `PlaceOrderRequest` is a removed field | Inferred | Inferred (agree) | `protos/demo.proto` L228–235: numbers go 3 → 5, so 4 is skipped (fact). But there is no `reserved 4;` (0 matches for "reserved"), so nothing shows a field was ever removed; it could also just have been skipped. The reason is a guess |


**Count:** of all the agent's claims, how many are *Evidenced* according to the agent? 17 According to you? 14

## Reflect

1. Did the agent ever label a guess as *Evidenced*? Give an example, or explain how you checked that it didn't.

Mostly no. All 5 claims I checked had the right label: the 3 Evidenced ones (#3, #20, #21) were confirmed in `demo.proto` and `checkoutservice/main.go`, and the 2 Inferred ones (#11, #24) really were guesses, with no comment, doc or `reserved` line stating a reason. How I check is in the table

2. A new developer reads only this reconstructed ADR. What wrong belief could they form?

They could believe the "Likely reasons" (visibility in a demo, immediate replies, tracing/health checks) were the original architects' real reasons, when no file states them. The "Considered options" section also reads as if Options 2–4 were really compared, but the Claim check marks that as an Assumption (#13). They might also think *every* service uses gRPC, though the shopping assistant is an exception (#25).

3. What can AI recover from code, and what can only the original architects know? Use one example from your ADR.

AI can recover **what** the system does and **how**: all RPCs are unary, all interfaces are in one `demo.proto`, `PlaceOrder` calls services one by one, and the card is charged before shipping with no refund (`main.go` L252–260). Only the original architects know **why**. For example, #11 says one shared proto was chosen "for visibility in a demo app", but nothing in the code or comments says so; it could equally be team convenience or polyglot code generation.

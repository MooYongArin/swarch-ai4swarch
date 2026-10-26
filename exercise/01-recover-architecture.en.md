**Language:** [ภาษาไทย](01-recover-architecture.md) | English

# Task 1: Recover the architecture (approximately 15 min)

> **Lecture link:** T7 architecture recovery, AICH2 (traceability), AICH3 (context), and the JSS finding that 93% of studies never validate AI output.
> **Question:** how much of an architecture can AI recover, and how much does it depend on the evidence it reads?

**Don't look at the original Online Boutique repository during this task:** its README shows the architecture diagram you are recovering.

For each run, start a **new chat or session** in your agent and send the message below. The agent follows [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) and writes its result to `exercise/outputs/1<run>-architecture.md`.

**Run A: interfaces only**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interfaces + deployment config**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

While the agent works, watch which files it opens. If it reads files outside the evidence you allowed, write that down in your prompt log.

## Results

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

Open each file with **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) to see the diagram.

## Verify Run B yourself (the important part)

Check **at least five** dependencies in Run B's table against the files. Open the cited file and find the evidence. You may also check the service's source code, e.g. `src/checkoutservice/main.go`.

| Caller → Callee | AI's evidence | Verified? ✅ / ❌ / ❓ | What you found |
|---|---|---|---|
| Customer → frontend | kubernetes-manifests/frontend.yaml (Service frontend-external, type: LoadBalancer, port http 80 → 8080) | ✅ | `frontend.yaml` L124–137: Service `frontend-external` is type LoadBalancer, port 80 → 8080, matching `containerPort: 8080` (L49). HTTP confirmed by `httpGet` probes. Shows the frontend is public; that the caller is a "Customer" is the AI's interpretation. |
| loadgenerator → frontend | kubernetes-manifests/loadgenerator.yaml (FRONTEND_ADDR=frontend:80; init container runs wget http://${FRONTEND_ADDR}) | ✅ | `loadgenerator.yaml` L69–70 and L82–83: FRONTEND_ADDR=frontend:80, which matches internal ClusterIP Service `frontend`. Init container `frontend-check` (L41–59) wgets the frontend until HTTP 200, a startup wait. But `kustomization.yaml` L24 comments loadgenerator out, so it's not deployed by default, and it's test traffic, not part of the shop. |
| frontend → productcatalogservice | kubernetes-manifests/frontend.yaml (PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550) | ✅ | `frontend.yaml` L69–70: PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550. Callee `productcatalogservice.yaml` L69–81: Service `productcatalogservice`, ClusterIP, port `grpc` 3550 → 3550; grpc readiness/liveness probes on 3550. Name and port match. Env var shows configuration only, not which RPCs are called. |
| frontend → currencyservice | kubernetes-manifests/frontend.yaml (CURRENCY_SERVICE_ADDR=currencyservice:7000) | ✅ | `frontend.yaml` L71–72: CURRENCY_SERVICE_ADDR=currencyservice:7000. Callee `currencyservice.yaml` L70–82: Service `currencyservice`, ClusterIP, port `grpc` 7000 → 7000; grpc probes on 7000. Name and port match. |
| frontend → cartservice | kubernetes-manifests/frontend.yaml (CART_SERVICE_ADDR=cartservice:7070) | ✅ | `frontend.yaml` L73–74: CART_SERVICE_ADDR=cartservice:7070. Callee `cartservice.yaml` L70–82: Service `cartservice`, ClusterIP, port `grpc` 7070 → 7070; grpc probes on 7070. Name and port match. |


- **Wrong or missing** in Run B (false dependencies, missing ones, containers to question): No false dependencies found in the 5 checked. Questionable container: loadgenerator (commented out in kustomization.yaml L24, test traffic only). Missing: env vars show configuration, not actual calls or which RPCs; languages not evidenced. Handled well: shoppingassistantservice correctly left out (not deployed, kustomization.yaml L35).

## Reflect (2–3 sentences each)

1. Compare A and B. What could the agent **not** know from the `.proto` file alone, and why?
From demo.proto alone, the agent could list the 9 gRPC services and their RPCs, but it found 0 dependencies and could not see the frontend, the redis-cart data store, the optional loadgenerator. Because a proto is an interface contract: it describes what each service offers, not who calls it, what storage sits behind it, or how it's deployed.
2. Which **one** mistake would mislead a new team member most if the diagram were published without checking?
redis-cart as a normal database: if the diagram doesn't show that it uses emptyDir, a newcomer would assume carts are stored permanently, but they're lost when the pod restarts. This is an availability risk.

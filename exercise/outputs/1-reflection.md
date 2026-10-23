# Task 1 reflection: Run A (proto only) vs Run B (proto + manifests)

Compared files:

- `exercise/outputs/1A-architecture.md`, Run A, evidence `system/online-boutique/protos/demo.proto`
- `exercise/outputs/1B-architecture.md`, Run B, evidence `demo.proto` + `system/online-boutique/kubernetes-manifests/`

## 1. Side-by-side

| Aspect | Run A (proto only) | Run B (proto + manifests) |
|---|---|---|
| Containers | 9 gRPC services | 9 gRPC services + `frontend` + `redis-cart` + `loadgenerator` (12) |
| Customer entry point | Not evidenced; `Person` has no arrow | `frontend` behind a `LoadBalancer` (`frontend.yaml`, Service `frontend-external`) |
| Dependencies | 0 drawn. 6 `checkoutservice` calls **[Inferred]** from shared message types | 17 `Rel`, each backed by a `*_ADDR` env var |
| Data stores | None. Cart store **[Inferred]** | `redis-cart` (`redis:alpine`, `emptyDir`, so not persistent) |
| Ports / deployment | Not evidenced | Ports per service, probes, Service types |
| Implementation language | Not evidenced | Still not evidenced (bare image names) |
| External systems | Not evidenced | Still not evidenced |

What Run A inferred, checked against Run B:

| Run A **[Inferred]** claim | Confirmed by Run B? |
|---|---|
| Checkout calls Payment, Email, Shipping, Currency, Cart, Product Catalog | Yes, all six (`checkoutservice.yaml` env vars) |
| A separate UI exists and calls the services | Yes (`frontend`, 7 backend calls) |
| Cart Service keeps state in some store | Yes (`redis-cart`) |
| Not inferred at all | `recommendationservice` → `productcatalogservice`; `loadgenerator` → `frontend` |

Reasoning from shared message types got the checkout fan-out right. It could not find calls that share no message type, such as recommendation → catalog, which only uses `product_id` strings.

## 2. What the agent could not know from the `.proto` alone, and why

The underlying reason is the same for every row: **a `.proto` file is an interface contract.** It declares what each server *offers* (services, RPCs, messages). It says nothing about who uses those interfaces, how they are deployed, or what sits behind them.

| Unknowable from `demo.proto` | Why the proto can't show it | Where it showed up (Run B) |
|---|---|---|
| **Who calls whom** | The proto defines servers only. Clients are not declared, because any program can import the stubs and call any RPC. | `*_ADDR` env vars in `frontend.yaml`, `checkoutservice.yaml`, `recommendationservice.yaml` |
| **The web frontend / entry point** | `frontend` serves HTTP and is a gRPC *client* only, so it has no `service` block in the proto. | `frontend.yaml` (port 8080, `LoadBalancer`) |
| **Data stores** | Storage is a private implementation detail behind the interface. `CartService` exposes `GetCart`, not where carts live. | `cartservice.yaml` (`redis-cart`, `REDIS_ADDR`) |
| **Durability of state** | Volume types are a deployment concern, not an interface concern. | `cartservice.yaml` (`emptyDir`) |
| **Ports, replicas, exposure, probes** | These are deployment topology. The proto has no network or runtime config. | Every manifest |
| **Non-gRPC components** (test tools, Redis) | Anything that isn't a gRPC server never appears in a proto. | `loadgenerator.yaml`, `redis-cart` |
| **Optional / planned services** | Feature flags and components are wiring, not contract. | `SHOPPING_ASSISTANT_SERVICE_ADDR` in `frontend.yaml`; commented-out kustomize components |
| **Which RPC each caller uses** | Neither the proto nor the manifests show this. A connection is not a method call. | Still unknown. Needs caller source, e.g. `src/checkoutservice/` |
| **Implementation language** | `option go_package` only sets where generated Go stubs go. It says nothing about any server's language. | Still unknown. Needs `src/<service>/` |
| **External providers** (card processor, mail) | `PaymentService.Charge` / `EmailService.SendOrderConfirmation` define the contract, not whether a real third party sits behind it. | Still unknown. Needs `src/paymentservice/`, `src/emailservice/` |

The last three rows matter too. Adding manifests did **not** settle language, RPC usage or external systems. Each kind of evidence answers a different set of questions.

## 3. The one mistake that would mislead a new team member most

**The Run A diagram, published as is, shows nine independent gRPC services with no entry point, no arrows and no data store.**

A new team member reading only the diagram, without the Gaps table, would take three wrong lessons from it:

1. **"Services are decoupled peers."** In C4, a missing arrow reads as "no dependency". In reality `checkoutservice` calls six services *synchronously* to place an order (`checkoutservice.yaml`), and `frontend` calls seven. Both are hubs. So an outage in `paymentservice` or `currencyservice` breaks checkout. The diagram hides the system's main availability risk and the blast radius of every change.
2. **"Customers talk to the gRPC services directly."** The `Person` floats with no arrow, and `frontend` is missing. The real exposure surface is a single HTTP `LoadBalancer` (`frontend.yaml`, `frontend-external`), with all gRPC services internal (`ClusterIP`). Someone reasoning about security, API compatibility or where to add auth would start from the wrong place.
3. **"The system is stateless."** `redis-cart` is missing, and with it the fact that cart data sits on an `emptyDir` volume and is lost when the pod is rescheduled (`cartservice.yaml`). That is the only data store in the deployment and a real durability concern.

**Why this is worse than Run B's mistakes:** Run A's prose is careful (it says "None evidenced" and lists the gaps). But a diagram travels without its caveats, and **missing evidence turns into evidence of absence** once it's drawn as an empty picture. Run B's errors are smaller and local:

- `loadgenerator`, a test tool that is commented out of `kustomization.yaml`, is drawn inside the system boundary as if it were production.
- `Rel` labels such as "Gets shipping quote" name specific RPCs that are not evidenced.

Each of those misleads about one box or one arrow. Run A's diagram misleads about the shape of the whole system.

**Fix before publishing:** don't publish a proto-only container diagram as the architecture. Either add the deployment evidence (Run B), or title it "Service interfaces (no runtime dependencies evidenced)" and print the Gaps next to it.

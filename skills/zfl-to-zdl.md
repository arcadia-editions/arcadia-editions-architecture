---
name: zfl-to-zdl
version: 1.1.0
description: >
  Generate one ZDL domain model per system from a ZFL business flow. Map
  actor starts, event-driven policies, and direct synchronous calls into ZDL
  services, commands, events, lifecycle state, transitions, and transport
  annotations.
triggers:
  - "zfl-to-zdl"
  - "generate ZDL from ZFL"
  - "domain model from flow"
  - "ZenWave ZDL"
  - ".zfl"
applies_to:
  - claude
  - codex
  - any agent with file read/write access
---

# Skill: ZFL to ZDL

Generate one `domain-model.zdl` per system declared in a ZFL flow.

This skill is for scaffold generation, not final domain design. The output should
be structurally valid, grounded in the flow, and conservative about assumptions.

## Inputs

Read these inputs before generating anything:

1. The target ZFL flow file.
2. Any `@zdl("...")` mappings declared inside the `systems` block.
3. `zenwave-architecture.yml` if present, to resolve repository paths and base packages.
4. The canonical ZDL grammar URL:
   `https://raw.githubusercontent.com/ZenWave360/dsl-kotlin/refs/heads/main/src/commonMain/antlr/io.zenwave360.language.antlr/Zdl.g4`
5. Existing `domain-model.zdl` files in target repos, if they already exist.

## Reference resolution order

Use references in this order:

1. The canonical ZDL grammar URL:
   `https://raw.githubusercontent.com/ZenWave360/dsl-kotlin/refs/heads/main/src/commonMain/antlr/io.zenwave360.language.antlr/Zdl.g4`
2. Existing local `domain-model.zdl` files in target or sibling repos.
3. The embedded ZDL example at the end of this skill.
4. Any user-provided example.

Do not search for or depend on a local `Zdl.g4`. Use the canonical grammar URL as the grammar source.
The grammar is the source of truth for syntax.
Examples are references for structure, idiom, and naming, not for overriding grammar.
Prefer existing local `domain-model.zdl` files over the embedded example for style and naming conventions.

## Invocation

Natural language is enough:

> Generate ZDL models from this ZFL, one per system, using the `@zdl(...)` targets,
> with lifecycle, transitions, commands, events, and transport annotations.

## Execution order

1. Read the ZFL and enumerate all systems.
2. Resolve the output path for each system.
3. Extract every command, event, response, and direct call relevant to each system.
4. Infer the aggregate root, lifecycle states, and transitions for each bounded context.
5. Generate one `domain-model.zdl` per system.
6. Validate the result against the checklist before writing.
7. Report written files and any assumptions that were not explicit in the flow.

## Output path resolution

Use the first matching rule:

1. If the system has `@zdl("path/to/domain-model.zdl")`, use that exact path relative to the current working directory where the command is being run.
2. Else if `zenwave-architecture.yml` maps the service repository and ZDL spec path, use that.
3. Else default to `{kebab-case-system-name}/domain-model.zdl` relative to the current path where the command is being run.

Treat the current working directory as the root for `@zdl(...)` resolution. For example,
if the agent is invoked from the repository root, `@zdl("customer-api/model.zdl")`
must write to `./customer-api/model.zdl` under that root.

Do not overwrite an existing file blindly. If a target already exists, read it first and preserve valid existing structure unless the user explicitly asked for replacement.

## API reference URL resolution

When generating entries for an `apis` block, resolve the AsyncAPI contract URL
in this order:

1. Apicurio Registry URL, when a registry base URL, group, artifact, and version
   can be inferred from `zenwave-architecture.yml`, environment-specific docs,
   or user input.
2. HTTP(S) URL, when the service publishes its AsyncAPI contract at a stable
   endpoint or documentation URL.
3. Local file path, only as a last resort when no registry or HTTP URL is known.

Use the newer one-line syntax for each API entry:

```zdl
apis {
    asyncapi client OrdersCheckoutApi "https://registry.example.com/apis/registry/v3/groups/orders.checkout/artifacts/orders-checkout-asyncapi/versions/1.0.0/content"
    asyncapi client PaymentsProcessingApi "https://payments.example.com/asyncapi.yml"
    asyncapi client CatalogInventoryApi "catalog-inventory-api/asyncapi.yml"
}
```

For Apicurio Registry v3, use the artifact content route:

```text
{registryBaseUrl}/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/versions/{versionExpression}/content
```

Prefer stable architecture identifiers over display names:

- `groupId`: use the producer service id from `zenwave-architecture.yml` when available, for example `orders.checkout.orders-checkout`.
- `artifactId`: use a deterministic AsyncAPI artifact id, preferably `{producer-service-kebab}-asyncapi`.
- `versionExpression`: use the producer service version when available, otherwise `latest`.

Do not invent an Apicurio or HTTP host. If the host cannot be discovered from
the repository context or user input, fall back to the local AsyncAPI artifact
path from `zenwave-architecture.yml`, for example `orders-checkout-api/asyncapi.yml`.

## Mapping rules

### 1. One ZDL per system

Each `system` block becomes one `domain-model.zdl`.

The file should contain:

- `config`
- one aggregate root entity
- one lifecycle enum
- input types as needed
- one primary service for that bounded context
- event definitions for the events the service emits

### 2. Aggregate and entity naming

Derive the aggregate root from the bounded context, not mechanically from the service class name.

Examples:

- `OrdersCheckout` -> `Order`
- `CatalogProducts` -> `Product`
- `PaymentsProcessing` -> `Payment`
- `FulfillmentShipping` -> `Shipment`
- `NotificationsConsumer` -> `Notification`

Add only the minimum domain fields needed to support:

- identity
- lifecycle state
- timestamps implied by events
- business values clearly implied by the flow

Do not invent a rich domain model unless the user explicitly asked for enrichment.

### 3. Event-driven commands

Map every policy of this form:

```zfl
when SomeEvent do someCommand {
    service SomeSystem.SomeService
    emits EventA
    emits EventB
}
```

to a ZDL service command on that system.

Classification:

- If the trigger is an `@actor` start event, model the command as actor-facing and annotate it with REST semantics.
- If the trigger is another service's event:
  - add an `api` to `apis` block using the API reference URL resolution rules, for example `apis { asyncapi client OtherServiceApi "https://registry.example.com/apis/registry/v3/groups/orders.checkout/artifacts/orders-checkout-asyncapi/versions/1.0.0/content" }`.
  - name client APIs as `{ProducerSystemName}Api`, for example `OrdersCheckoutApi`.
  - annotate the service command with `@asyncapi(api: OtherServiceApi, channel: <EventName>Channel)`.
  - if the event is consumed from the same service's own AsyncAPI, omit `api` and use `@asyncapi(channel: <EventName>Channel)`.
- If it's an standalone `do someCommand { }` block map them as direct calls with REST semantics: `emits` becomes and event, `response` becomes the REST response. See direct calls below.

The emitted outcomes become:

- `withEvents EventA` for one outcome
- `withEvents [EventA | EventB]` for multiple outcomes

### 4. Direct calls

Map every synchronous step of this form:

```zfl
when StartOrderCheckout do startOrderCheckout {
    service OrdersCheckout.OrdersCheckoutService
    call reserveStock
    on StockReserved emits OrderCreated
    on StockUnavailable emits StockUnavailable
}

do reserveStock {
    service CatalogProducts.CatalogProductsService
    response StockReserved
    response StockUnavailable
}
```

as two separate concerns:

1. The orchestration command remains on the calling service.
2. The called operation becomes a direct service command on the called service.

Direct calls map to ZDL service commands with `@post`, not `@asyncapi`.

Implications:

- `startOrderCheckout` belongs to `OrdersCheckoutService`.
- `reserveStock` belongs to `CatalogProductsService`.
- `reserveStock` is synchronous, so model it as a direct command with `@post`.
- The `response` values become the possible outcomes of the direct command.
- The caller's `on <Response> emits <Event>` clauses become the caller's outward withEvents.

Use this pattern:

```zdl
@rest("/products")
service CatalogProductsService for (Product) {
    @post("/stock/reservations")
    reserveStock(ReserveStockInput) ReserveStockResponse
}
```

### 5. Responses and events

Treat `response` names in a direct call definition as domain outcomes that should still be modeled explicitly in ZDL.

In practice:

- define them as events in the called service file when they represent published business facts
- use them in `withEvents`
- keep naming consistent with the flow

If the flow clearly uses a response only as a private synchronous result and not as a published contract, keep it minimal and do not invent extra messaging annotations beyond what the ZDL example and grammar support.

A `response` typicaly maps to an `output` object in the ZDL, if multiple responses are possible, use an `enum` in the `output` response type:

```zdl
@rest("/products")
service CatalogProductsService for (Product) {
    @post("/stock/reservations")
    reserveStock(ReserveStockInput) StockReserved
}
output StockReserved {
    productId String
}
@output
enum StockReserved { RESERVED(1), UNAVAILABLE(2) }
```

### 6. Lifecycle and transitions

Read the flow as a state machine.

For each service:

- identify the first command that creates or takes ownership of the aggregate
- infer the initial lifecycle state
- infer subsequent states from the happy path and compensations
- annotate state-changing methods with `@transition`

Guidance:

- Use `@transition(to: STATE)` for the creation step.
- Use `@transition(from: STATE_A, to: STATE_B)` for normal progression.
- Use `@transition(from: [STATE_A, STATE_B], to: STATE_C)` when multiple source states are valid.
- If one command can end in multiple distinct states, use multiple `@transition` annotations.

Do not force every event into a state transition. Query methods and pure notification handlers may have no transition.

### 7. Transport annotations

Use transport annotations by interaction type:

- actor-facing command -> `@rest` on the service and `@post` on the method
- direct synchronous inter-service call -> `@post` on the method
- event-triggered command -> `@asyncapi(api: OtherServiceApi, channel: <EventName>Channel)`
- self-consuming event-triggered command -> `@asyncapi(channel: <EventName>Channel)`

For pure consumer services, omit `@rest` if they have no actor-facing or direct-call surface.

### 8. Naming conventions

Use existing repository naming if available from `@zdl(...)` or `zenwave-architecture.yml`.

Otherwise:

- base package: `io.arcadiaeditions.{domain}.{subdomain}`
- event topic: `{domain}.{subdomain}.{event-name-kebab-case}.[event|command|response].v1`
- events channel: `{PascalCaseEvent}Channel`
- AsyncAPI client name: `{ProducerSystemName}Api`, for example `OrdersCheckoutApi`

Prefer the naming already present in the provided example over invented alternatives.

## Quality checklist

Before writing each file, verify:

- Every command owned by the service appears as a service method.
- Every emitted event owned by the service appears as an event block.
- Every direct call is modeled as a synchronous command with `@post`.
- Event-triggered commands are not mislabeled as direct calls.
- State-changing methods have `@transition` annotations.
- Lifecycle enum values cover every state referenced by `@lifecycle` and `@transition`.
- `withEvents` matches the outcomes declared in the flow.
- Output paths honor `@zdl(...)` when present.
- Existing files were preserved or updated intentionally, not overwritten by accident.

## Writing style for generated ZDL

- Stay close to the grammar and example.
- Prefer simple, valid scaffolds over speculative completeness.
- Keep one aggregate root per file unless the user explicitly asked for a richer model.
- Use ASCII only.
- Keep comments sparse and only where structure would otherwise be unclear.

## Report back

When done, report:

- files created or updated
- direct calls discovered and how they were mapped
- assumptions made about aggregate names, states, or fields

## Embedded ZDL example

Use the following example as the default syntax and style reference when no better
local example exists. Prefer its structure and idioms, but keep generated output
minimal and specific to the target flow.

```zdl
config {
    title "Order Fulfillment Example"
    basePackage "io.zenwave360.example.orderfulfillment"
}

@aggregate
@lifecycle(field: status, initial: DRAFT)
entity Order {
    @naturalId
    orderNumber String required unique maxlength(36)
    status OrderStatus required
    totalAmount BigDecimal required
}

enum OrderStatus {
    PLACED(1),
    PAID(2),
    SHIPPED(3),
    CANCELLED(4)
}

input PlaceOrderInput {
    items OrderItem[] minlength(1)
    currency String required maxlength(3)
}

input PayOrderInput {
    paymentReference String required
}

input ShipOrderInput {
    trackingNumber String required
}

@rest("/orders")
service OrderService for (Order) {

    @post
    @transition(to: PLACED)
    placeOrder(PlaceOrderInput) Order withEvents OrderPlaced

    @post("/{orderNumber}/pay")
    @transition(from: PLACED, to: PAID)
    payOrder(@natural id, PayOrderInput) Order withEvents OrderPaid

    @post("/{orderNumber}/ship")
    @transition(from: PAID, to: SHIPPED)
    shipOrder(@natural id, ShipOrderInput) Order withEvents OrderShipped

    @post("/{orderNumber}/cancel")
    @transition(from: [DRAFT, PLACED], to: CANCELLED)
    cancelOrder(@natural id) Order withEvents OrderCancelled

    @get("/{orderNumber}")
    getOrder(@natural id) Order?
}

@asyncapi({ channel: "OrderPlacedChannel", topic: "orderfulfillment.order-placed.event.v1" })
event OrderPlaced {
    id Long
    version Integer
}

@asyncapi({ channel: "OrderPaidChannel", topic: "orderfulfillment.order-paid.event.v1" })
event OrderPaid {
    id Long
    version Integer
}

@asyncapi({ channel: "OrderShippedChannel", topic: "orderfulfillment.order-shipped.event.v1" })
event OrderShipped {
    id Long
    version Integer
    trackingNumber String
}

@asyncapi({ channel: "OrderCancelledChannel", topic: "orderfulfillment.order-cancelled.event.v1" })
event OrderCancelled {
    id Long
    version Integer
}
```

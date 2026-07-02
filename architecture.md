# Small Business Service Platform - Architecture (DDD)

## 0) DDD Quality Score (Current)

Current score: **8.6 / 10**

Why not 10 yet:

- Some terms still overlap (`order`, `ticket`, `job`) and need final standardization with business stakeholders.
- Cross-context contracts are defined conceptually but not yet formalized as versioned API/event schemas.
- Invariants are defined, but conflict handling and compensation policies need implementation-level detail.

What gets us to 10/10:

- Approve and freeze a v1 ubiquitous language glossary.
- Define versioned published language for context integrations.
- Add executable architecture tests for context boundaries and state transition rules.

---

## 1) Domain Vision and Scope

Build a modular management system for small businesses, starting with coffee shops, with a reusable operational core for other service businesses (for example hair salons).

Primary business outcomes:

- Fast, reliable order intake from guest-facing screen.
- Controlled handoff from cashier/payment to execution team.
- Clear production queue with explicit status transitions.
- Flexible staffing and permissions (role changes, multi-role users).
- Configurable product catalog and guest-facing content.

Strategic domain distillation:

- **Core domain**: Service execution flow (queueing, status transitions, handoff, failure handling).
- **Supporting subdomains**: Catalog management, storefront content, reporting.
- **Generic subdomains**: Authentication, RBAC, payment gateway adapters, notification channels.

---

## 2) Ubiquitous Language (v1)

Use these terms in code, docs, UI copy, and conversations.

### Core nouns

- **ServiceOrder**: Customer-requested unit of work (coffee order, salon visit request).
- **OrderLine**: A purchasable/requested line item within a ServiceOrder.
- **Cart**: Pre-checkout collection of requested lines.
- **CheckoutSession**: Payment and cashier adjustment process for one ServiceOrder.
- **FulfillmentTicket**: Execution artifact consumed by preparation/service staff.
- **QueueEntry**: Position of a ticket/customer in a waiting line.
- **HandoverTask**: Delivery task to counter, table, or service station.
- **StaffMember**: User assigned to operational roles.
- **RoleAssignment**: Time-bounded assignment of permissions to a StaffMember.
- **CatalogItem**: Sellable item (product) or booked work (service).
- **ServiceStation**: Execution resource (kitchen station, bar station, salon chair).

### Core verbs

- **placeOrder**: Confirm cart into ServiceOrder.
- **amendOrder**: Cashier edits order before execution lock.
- **capturePayment**: Record successful payment at counter.
- **startFulfillment**: Move ticket from `new` to `in_progress`.
- **markReady**: Move ticket to `ready`.
- **completeHandover**: Mark delivery/service completion.
- **failFulfillment**: End ticket as failed with mandatory reason.
- **assignRole / revokeRole**: Manage staff permissions.

### Status terms

- **OrderStatus**: `draft | placed | paid | cancelled`
- **FulfillmentStatus**: `new | in_progress | ready | delivered | failed`
- **QueueStatus**: `waiting | called | seated | no_show | cancelled`

---

## 3) Bounded Contexts and Context Map

Start as a **modular monolith** in Laravel. Context boundaries are code/module boundaries first, deployment boundaries later.

## Contexts

1. **CatalogContext**
    - Owns: CatalogItem, variants/options, availability rules, base pricing.
2. **OrderingContext**
    - Owns: Cart, ServiceOrder creation, guest checkout initiation.
3. **CheckoutContext**
    - Owns: CheckoutSession, cashier amendments, payment capture records.
4. **FulfillmentContext**
    - Owns: FulfillmentTicket, queue logic, execution transitions, failure reasons.
5. **HandoverContext**
    - Owns: delivery-to-counter/table/service-station completion flow.
6. **IdentityAccessContext**
    - Owns: StaffMember, RoleAssignment, permission policies.
7. **BackofficeContext**
    - Owns: admin configuration, storefront content blocks/pages.

## Context map (relationship pattern)

- OrderingContext -> CatalogContext: **Conformist** (reads published catalog view).
- CheckoutContext -> OrderingContext: **Customer/Supplier** (order snapshot and amendment policies).
- FulfillmentContext -> CheckoutContext: **Anti-Corruption Layer** (consumes payment/order signals mapped into ticket terms).
- HandoverContext -> FulfillmentContext: **Customer/Supplier** (acts on ready tickets).
- BackofficeContext -> CatalogContext: **Customer/Supplier** (authoring and publishing flow).
- All contexts -> IdentityAccessContext: **Open Host Service + Published Language** for authz checks.

Integration rule:

- No context reads another context's internal tables directly.
- Cross-context communication via application services + domain/integration events.

---

## 4) Domain Model: Aggregates, Invariants, Events

## 4.1 Aggregates by context

### OrderingContext

- **ServiceOrder (AR)**
    - Entities: OrderLine
    - Value Objects: OrderId, Money, CustomerNote, FulfillmentTarget
    - Invariants:
        - Cannot place empty order.
        - Price snapshot frozen at placeOrder time.
        - Cancellation blocked after fulfillment started.

### CheckoutContext

- **CheckoutSession (AR)**
    - Value Objects: PaymentMethod, PaidAmount, ChangeAmount
    - Invariants:
        - `capturePayment` allowed once per session.
        - Manual/card reference required for non-cash methods.
        - Amendment allowed only before payment captured (configurable policy).

### FulfillmentContext

- **FulfillmentTicket (AR)**
    - Value Objects: TicketNumber, FailureReason, StationId
    - Invariants:
        - Transitions allowed only: `new -> in_progress -> ready -> delivered` or `new|in_progress -> failed`.
        - `failed` requires FailureReason.
        - `delivered` only from `ready`.

- **QueueEntry (AR)** (used strongly in salon mode)
    - Value Objects: QueuePosition, ArrivalTime
    - Invariants:
        - One active queue entry per order/customer target.
        - Position changes emit reorder events.

### HandoverContext

- **HandoverTask (AR)**
    - Value Objects: Destination(counter/table/station), CompletedAt
    - Invariants:
        - Task created only for `ready` ticket.
        - Completing handover closes task idempotently.

### IdentityAccessContext

- **StaffMember (AR)**
    - Entities: RoleAssignment
    - Invariants:
        - Staff can hold multiple active roles.
        - Role change is append-only history (no silent overwrite).

### CatalogContext

- **CatalogItem (AR)**
    - Entities: CatalogVariant, OptionGroup
    - Value Objects: Price, AvailabilityWindow
    - Invariants:
        - Inactive items cannot be added to new carts.
        - Variant pricing cannot be negative.

## 4.2 Domain events (minimum set)

- `ServiceOrderPlaced`
- `ServiceOrderAmendedByCashier`
- `PaymentCaptured`
- `FulfillmentTicketCreated`
- `FulfillmentStarted`
- `FulfillmentMarkedReady`
- `FulfillmentFailed`
- `HandoverCompleted`
- `QueueEntryCreated`
- `QueuePositionChanged`
- `StaffRoleAssigned`
- `StaffRoleRevoked`
- `CatalogItemPublished`

Event policy:

- Domain events are internal per context.
- Integration events crossing contexts use explicit payload contracts and versioning.

## 4.3 State machine profiles

### Coffee shop profile

- Flow: `Cart -> ServiceOrder(placed) -> CheckoutSession(paid) -> FulfillmentTicket(new/in_progress/ready) -> HandoverTask(delivered)`

### Salon profile

- Flow: `Cart/Booking -> ServiceOrder(placed) -> QueueEntry(waiting/called/seated) -> FulfillmentTicket(in_progress/ready) -> CheckoutSession(paid if post-service) -> HandoverCompleted(done)`

Both profiles reuse same core abstractions with policy toggles.

---

## 5) First 10 User Stories (MVP)

1. As a **guest**, I can browse active catalog items by category and search term.
2. As a **guest**, I can add/remove items in cart and see subtotal updates.
3. As a **guest**, I can place an order and receive an order number.
4. As a **cashier**, I can amend an unpaid placed order (quantity/remove/add line).
5. As a **cashier**, I can capture payment as cash or card and mark order paid.
6. As a **kitchen/service staff**, I can view new tickets sorted by queue priority/time.
7. As a **kitchen/service staff**, I can transition ticket `new -> in_progress -> ready`.
8. As a **kitchen/service staff**, I can fail a ticket with mandatory reason.
9. As a **waiter/front staff**, I can mark ready ticket as delivered to destination.
10. As a **manager**, I can assign/revoke roles so one person can hold cashier+waiter permissions.

Acceptance baseline for all stories:

- Auditable timestamp + actor identity for every state transition.
- Authorization policy checked in application layer.

---

## 6) Laravel Implementation Plan (Phased)

## Phase 1 - Foundation (1-2 weeks)

- Establish modular structure:
    - `app/Domain/<Context>/...`
    - `app/Application/<Context>/...`
    - `app/Infrastructure/<Context>/...`
    - `app/Interfaces/Http/...`
- Add architectural conventions:
    - Domain has no Laravel/Eloquent dependencies.
    - Repositories are interfaces in Domain, implementations in Infrastructure.
- Introduce shared value objects (`Money`, `OrderId`, `TicketNumber`).
- Setup RBAC with `spatie/laravel-permission` in IdentityAccessContext.

Deliverable:

- Skeleton modules + first architecture tests for dependency direction.

## Phase 2 - Ordering + Checkout MVP (2-3 weeks)

- Implement CatalogContext read model for guest storefront.
- Implement Cart + ServiceOrder placement.
- Implement CheckoutSession with cashier amendment and payment capture.
- Emit `ServiceOrderPlaced` and `PaymentCaptured` integration events.

Deliverable:

- End-to-end flow: guest places order, cashier captures payment.

## Phase 3 - Fulfillment + Handover MVP (2-3 weeks)

- Implement FulfillmentTicket aggregate and transition rules.
- Implement queue board for kitchen/service staff.
- Implement HandoverTask completion for counter/table delivery.
- Add failure handling with typed failure reasons.

Deliverable:

- Full operational pipeline from paid order to delivered/failed.

## Phase 4 - Backoffice and Configurability (2 weeks)

- Catalog admin CRUD with publish/unpublish workflow.
- Storefront content editor (sections/blocks) with versioned publish.
- Business policy settings (prepay required, amendment window, queue policy).

Deliverable:

- Manager-controlled configuration and content updates.

## Phase 5 - Hardening and Scale Readiness (ongoing)

- Add idempotency keys to transition commands.
- Add optimistic locking/version on aggregate roots.
- Add outbox pattern for reliable event publishing.
- Add SLA metrics: lead time per status, fail ratio, queue waiting time.
- Prepare selected contexts for extraction if scale requires microservices.

Deliverable:

- Production-grade reliability and measurable operations.

---

## 7) Suggested Code Layout (Initial)

```text
app/
  Domain/
    Ordering/
    Checkout/
    Fulfillment/
    Handover/
    Catalog/
    IdentityAccess/
    Shared/
  Application/
    Ordering/
    Checkout/
    Fulfillment/
    Handover/
    Catalog/
    IdentityAccess/
  Infrastructure/
    Persistence/
    Messaging/
    Auth/
  Interfaces/
    Http/
    Console/
```

---

## 8) Architectural Guardrails

- Do not pass Eloquent models into Domain layer.
- Do not expose aggregate internals across contexts.
- Enforce state transitions only through aggregate methods.
- Keep events past-tense and immutable.
- Prefer Value Objects over primitive argument lists.

---

## 9) Open Decisions (to finalize with stakeholders)

- Payment timing policy per business type: prepay vs post-service.
- Amendment policy after payment: forbidden vs manager override flow.
- Queue priority strategy: FIFO only vs weighted priorities.
- Service station assignment strategy: manual vs auto-dispatch.
- Cancellation/refund policy and accounting integration depth.

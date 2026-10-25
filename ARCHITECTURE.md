# NexusCore Architecture Reference

## Status and scope

**Status:** In development and continuously updated.  
**Evidence snapshot:** 2026-10-25.  
**Purpose:** Explain the request and integration architecture of NexusCore without copying tenant data, credentials, connection strings, secrets, or deploy-time values.

This is a source-derived reference, not a production runbook or an authority for live configuration. The code and deployed configuration remain the source of truth.

## System shape

NexusCore is a React/Vite operational workspace backed by an ASP.NET Core YARP API Gateway and bounded-context services. The local composition includes SQL Server, Redis, RabbitMQ, the web application, gateway, and service hosts for Master, HR, Trade, Accounting, Manufacturing, Ecommerce, and Field Sales.

```text
Browser workspace / storefront
  -> React + Vite
  -> API Gateway (YARP)
  -> route policy and identity-claim propagation
  -> bounded context service
  -> service-owned persistence boundary
  -> outbox and broker consumers when cross-context work is required
```

The diagram in `index.html` is intentionally a readable abstraction: it shows the request path and the principal boundaries, not every internal endpoint or controller.

## Verified request lifecycle

1. The browser sends an API request to the gateway.
2. YARP matches an explicit route and selects a destination cluster.
3. Protected routes require the configured bearer policy. Several routes transform identity claims into `X-Tenant-Id` and `X-Employee-Id` request headers for the target service.
4. The destination service runs its own host pipeline. HR, Trade, Accounting, and Manufacturing visibly compose authentication, tenant-integrity validation, authorization, tenant middleware, tenant-lifecycle enforcement, tenant requirement, and employee-access enforcement.
5. The bounded context validates its command/query, writes only to its own persistence boundary, and returns its result.
6. When integration is needed, the owning service records an outbox message with the business transaction. Consumers process the message through the broker rather than sharing another service's database.

### Important boundary rule

Gateway claim headers are transport context, not proof of authorization by themselves. Each bounded context must enforce its own policy and tenant controls. A newly introduced route is not covered merely because a nearby gateway route is protected.

## Service catalogue

| Service | Primary responsibility | Gateway ingress observed | Notes |
| --- | --- | --- | --- |
| Master | Identity, tenant registry, permissions, billing, platform administration | identity/master route family | Host composes authentication, authorization, security headers and rate limiting. |
| HR | Employee, attendance, payroll and organization workflows | `/api/v1/hr`, `/api/v1/workspace` | Tenant/lifecycle/employee gates are visible in the service pipeline. |
| Trade | Inventory, procurement, sales, POS and storefront operations | `/api/v1/inventory`, `/api/v1/procurement`, `/api/v1/sales`, `/api/v1/pos`, storefront routes | Trade MassTransit and outbox registration are visible. |
| Accounting | Journal, report and posting-route operations | `/api/v1/accounting`, `/api/v1/finance` | Accounting MassTransit consumers and tenant/lifecycle/employee gates are visible. |
| Manufacturing | Manufacturing workflows | `/api/v1/manufacturing` | Explicit permission registration plus tenant/lifecycle/employee gates are visible. |
| Ecommerce | Storefront summaries and imports | `/api/v1/ecommerce` | Protected endpoints use the JWT tenant claim. Treat every newly exposed endpoint as a tenancy-review event. |
| Field Sales | Field sales workflow and supervisor actions | `/api/v1/field` | Controllers default to authenticated access; rate limits distinguish field writes and sensitive decisions. |

## Data ownership and integration

### Synchronous data

Each bounded context is responsible for its own write model and persistence access. Cross-context behavior must use a published contract, approved internal API, or an asynchronous message. Do not query another service's database directly to bypass its business rules, tenancy boundary, audit, or invariants.

### Asynchronous data

RabbitMQ is the broker for asynchronous integration. The outbox pattern keeps the integration intent durable with the business transaction, allowing retry after broker interruption and keeping an event from being silently lost after the write succeeds.

An outbox is not a license for duplicate side effects: consumers still require idempotency, traceability, failure visibility, and a recovery path.

### Runtime services

SQL Server stores relational application data. Redis supports distributed runtime concerns. Their presence in the deployment topology does not change service data ownership.

## Security and tenancy guardrails

- Default to authenticated routes. Explicitly document every public exception and its input protection.
- The gateway and service host are separate enforcement points. A service must fail closed when tenant or employee context is absent where it is required.
- Use the authenticated claim context as the source for propagated identity headers; never trust browser-supplied tenant or employee headers.
- Keep public webhook/capture flows isolated, replay-resistant, and validated using their dedicated contracts rather than weakening normal API authorization.
- Do not place secrets, connection strings, tenant IDs, user data, production URLs, or support tokens in this repository.

## Accounting boundary

Accounting owns journal and posting-route execution. Commercial and operational services may request accounting work through defined contracts, but they do not own the general ledger. A posting route must resolve deterministically for the document scenario; ambiguous or incomplete routes must reject before a journal is created.

## Operational assumptions

| Assumption | Why it matters | Documentation action when it changes |
| --- | --- | --- |
| A gateway route maps to one intended service cluster | Prevents accidental service exposure | Update the ingress catalogue and interactive diagram. |
| Tenant context is validated inside tenant-aware services | Prevents cross-tenant access | Update the lifecycle and security sections; add test evidence. |
| Cross-context messages are durable and observable | Prevents silent integration loss | Update the outbox/consumer contract and failure path. |
| Services own their data boundaries | Protects domain invariants and auditability | Record any approved exception as an architecture decision. |

## What this reference does not claim

- It does not certify a production deployment.
- It does not enumerate every endpoint, permission, queue, database table, or event schema.
- It does not state that all service pipelines are identical; each host must be read independently.
- It does not replace automated security, contract, integration, or runtime tests.

## Evidence map

| Topic | Source file |
| --- | --- |
| Local deployment topology | `docker-compose.yml` |
| Gateway route-to-cluster mapping | `services/gateway/NexuseCore.ApiGateway/appsettings.json` |
| Gateway runtime pipeline | `services/gateway/NexuseCore.ApiGateway/Program.cs` |
| Master host pipeline | `services/master/NexuseCore.Master.Api/Program.cs` |
| HR host pipeline | `services/hr/NexuseCore.HR.Api/Program.cs` |
| Trade host pipeline and messaging | `services/trade/NexuseCore.Trade.Api/Program.cs` |
| Accounting host pipeline and consumers | `services/accounting/NexuseCore.Accounting.Api/Program.cs` |
| Manufacturing host pipeline | `services/manufacturing/NexuseCore.Manufacturing.Api/Program.cs` |
| Ecommerce host endpoints | `services/ecommerce/NexuseCore.Ecommerce.Api/Program.cs` |
| Field Sales host pipeline and rate limits | `services/field-sales/NexusCore.FieldSales.Api/Program.cs` |

## Documentation update contract

Update this repository in the same change as any of the following:

1. A gateway route, destination cluster, claim transformation, or authorization policy changes.
2. A bounded context gains, loses, or changes a public or internal integration surface.
3. A service changes its tenant, lifecycle, employee-access, or permission enforcement.
4. A new persistent store, cache, broker, outbox, consumer, or external integration is introduced.
5. A business-critical ownership rule changes, especially around accounting, inventory, payroll, or identity.

For every update, review the diagram, the service catalogue, the evidence map, and the status/snapshot date together. Do not update only the visual while leaving contradictory written guidance behind.

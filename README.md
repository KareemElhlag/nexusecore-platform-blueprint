# NexusCore Architecture Explorer

Standalone documentation repository for explaining the NexusCore ERP request architecture.

## Documentation status

**In development and continuously updated.** This is an explanatory architecture view, not the deployment source of truth. Update the diagram and the evidence list whenever a service boundary, gateway route, request guard, datastore, or integration contract changes.

The current view is based on the NexusCore source configuration inspected on 2026-10-25. It deliberately avoids tenant data, live endpoint values, credentials, and deployment secrets.

## Open locally

Open `index.html` in a modern browser. No build step, service, credential, or copy of the ERP source is required.

## What the diagram shows

- Browser requests flowing through the React/Vite frontend and YARP API Gateway.
- Authentication, authorization, tenant identity propagation, lifecycle checks, and employee-access checks.
- Primary bounded contexts: Master, HR, Trade, Accounting, Manufacturing, Ecommerce, and Field Sales.
- Synchronous persistence via isolated tenant databases and asynchronous integration via RabbitMQ outbox/consumers.

## Detailed reference

Read [`ARCHITECTURE.md`](ARCHITECTURE.md) for the verified request lifecycle, service ownership and ingress catalogue, guard order, data/integration boundaries, operational guardrails, evidence map, and documentation update contract.

## Evidence sources in NexusCore

- `docker-compose.yml`
- `services/gateway/NexuseCore.ApiGateway/appsettings.json`
- `services/gateway/NexuseCore.ApiGateway/Program.cs`
- `services/master/NexuseCore.Master.Api/Program.cs`
- `services/hr/NexuseCore.HR.Api/Program.cs`
- `services/trade/NexuseCore.Trade.Api/Program.cs`
- `services/accounting/NexuseCore.Accounting.Api/Program.cs`

The explorer intentionally represents architectural boundaries and request flow; it does not expose live endpoints, tenant data, credentials, or deployment secrets.

---
id: RFC-0004
title: "Migrate the public API from REST to GraphQL"
status: Superseded
author: Donald Gifford
created: 2026-05-17
---
<!-- markdownlint-disable-file MD025 MD041 -->

# RFC 0004: Migrate the public API from REST to GraphQL

**Status:** Superseded
**Author:** Donald Gifford
**Date:** 2026-05-17

<!--toc:start-->
- [Summary](#summary)
- [Problem Statement](#problem-statement)
- [Proposed Solution](#proposed-solution)
- [Design](#design)
  - [Sample schema sketch](#sample-schema-sketch)
  - [Façade migration](#faade-migration)
- [Alternatives Considered](#alternatives-considered)
- [Supersession](#supersession)
- [Implementation Phases](#implementation-phases)
- [Risks and Mitigations](#risks-and-mitigations)
- [Success Criteria](#success-criteria)
- [References](#references)
<!--toc:end-->

## Summary

Proposed to replace the public REST API (`api.example.com/v1/*`) with a single GraphQL endpoint (`api.example.com/graphql`). **Superseded** in 2026-Q1 by the decision to standardize on OpenAPI 3.1 + REST per the counter-proposal that emerged from the [RFC-0002 rejection](./0002-use-grpc-for-all-internal-service-to-service-communication.md).

> [!IMPORTANT]
> This RFC is **superseded**. The active strategy is OpenAPI 3.1 + REST for the public surface. This document is preserved for historical context; the reasoning that drove the GraphQL pitch is still useful background for anyone evaluating a fresh attempt later.

## Problem Statement

The current REST surface has three pain points for partner integrators:

1. **Over-fetching.** A typical product-detail page request pulls `~140 KB` of JSON for fields the client uses `~12 KB` of.
2. **Round-trip chains.** Fetching an order with its line items and product details requires 3 sequential calls; partner SLAs are eaten by RTT.
3. **Versioning drift.** We currently maintain `v1` + `v2` for 22 endpoints and have a public sunset date for `v1` we're going to miss for the third time.

## Proposed Solution

- Single endpoint: `POST /graphql`.
- Schema-first using **GraphQL SDL** + `graphql-codegen` for typed clients.
- Apollo Server in front of the existing REST handlers (façade pattern for the migration window).

## Design

### Sample schema sketch

```graphql
type Order {
  id: ID!
  status: OrderStatus!
  lineItems: [LineItem!]!
  customer: Customer!
}

type Query {
  order(id: ID!): Order
  orders(customerId: ID!, after: String, first: Int): OrderConnection!
}
```

### Façade migration

For the first 6 months, GraphQL resolvers call existing REST handlers in-process. After traffic shifts, resolvers migrate to direct data-layer calls.

## Alternatives Considered

- **REST + sparse fieldsets (`?fields=`).** Solves over-fetching without the schema lift; less expressive than GraphQL but cheaper.
- **OpenAPI 3.1 + generated typed clients.** What we ultimately picked. See *Supersession*.
- **JSON:API.** Considered; ecosystem maturity has stalled and tooling is thinner than either alternative.

## Supersession

In 2026-Q1, after the architecture review on the [gRPC proposal (RFC-0002)](./0002-use-grpc-for-all-internal-service-to-service-communication.md), the platform team produced a counter-proposal: **standardize on OpenAPI 3.1 + REST** for both internal and public surfaces, with `oapi-codegen` (Go), `openapi-typescript`, and `openapi-generator` (Java) producing typed clients.

The OpenAPI direction won on these dimensions:

| Dimension | GraphQL | OpenAPI 3.1 + REST |
|-----------|---------|--------------------|
| Solves over-fetching | Yes | Partial (sparse fieldsets) |
| Solves N+1 round-trips | Yes | Partial (resource embedding) |
| Caching (CDN edge) | Hard (POST) | Easy (GET + standard headers) |
| Partner onboarding cost | High (new tooling) | Low (familiar) |
| Server complexity | High (resolver wiring) | Low (HTTP handlers) |
| Schema sharing internal ↔ public | Same | Same |
| Tooling maturity in our stack | Patchy | Mature |

Caching at the CDN edge turned out to be the decisive factor — our partner traffic profile is read-heavy and benefits significantly from edge caching, which GraphQL POST queries make awkward.

## Implementation Phases

N/A — superseded.

## Risks and Mitigations

N/A — superseded.

## Success Criteria

N/A — superseded.

## References

- Superseded by the OpenAPI 3.1 standardization (internal ADR, 2026-Q1).
- Related: [RFC-0002](./0002-use-grpc-for-all-internal-service-to-service-communication.md)

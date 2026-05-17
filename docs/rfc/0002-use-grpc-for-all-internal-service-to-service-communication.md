---
id: RFC-0002
title: "Use gRPC for all internal service-to-service communication"
status: Rejected
author: Donald Gifford
created: 2026-05-17
---
<!-- markdownlint-disable-file MD025 MD041 -->

# RFC 0002: Use gRPC for all internal service-to-service communication

**Status:** Rejected
**Author:** Donald Gifford
**Date:** 2026-05-17

<!--toc:start-->
- [Summary](#summary)
- [Problem Statement](#problem-statement)
- [Proposed Solution](#proposed-solution)
- [Design](#design)
  - [Sketched transport map](#sketched-transport-map)
- [Alternatives Considered](#alternatives-considered)
- [Decision (REJECTED)](#decision-rejected)
- [Implementation Phases](#implementation-phases)
- [Risks and Mitigations](#risks-and-mitigations)
- [Success Criteria](#success-criteria)
- [References](#references)
<!--toc:end-->

## Summary

Proposal to make **gRPC** the mandatory transport for all internal service-to-service calls, retiring our mix of REST + JSON over HTTP/1.1. **Rejected** at the 2026-05-12 architecture review.

## Problem Statement

Our internal API surface is inconsistent:

- 14 services speak REST + JSON (mostly via Spring + Express).
- 4 services expose hand-rolled binary protocols (legacy from the 2021 monolith decomposition).
- 2 services use GraphQL internally (mistakenly — see [RFC-0004](./0004-migrate-the-public-api-from-rest-to-graphql.md)).

This causes hand-coded client stubs in 6 languages, no shared contract definition, and an 8-12% JSON-parsing overhead on the hot path of `order-svc` at p99.

## Proposed Solution

- All new services define their RPCs in **protobuf**.
- `buf` enforces lint + breaking-change checks in CI.
- Generated stubs land in a per-language `proto-stubs/` mono-package.
- All call paths use `grpc-go`, `grpc-java`, or `@grpc/grpc-js`.

> [!WARNING]
> This RFC was rejected. Reasoning preserved here for posterity — do not implement.

## Design

### Sketched transport map

| Caller | Callee | Today | Proposed |
|--------|--------|-------|----------|
| `order-svc` | `inventory-svc` | REST/JSON | gRPC unary |
| `notification-svc` | `user-svc` | REST/JSON | gRPC unary |
| `analytics-svc` | `event-stream` | Kafka | gRPC server-stream |

## Alternatives Considered

- **OpenAPI 3.1 + generated REST clients.** Lighter onboarding. Lower performance gains but sufficient for our P99 budgets.
- **Async messaging only.** Won't replace the synchronous calls that drive 70% of internal traffic.
- **Status quo.** Operationally painful but the cost of switching is also non-trivial.

## Decision (REJECTED)

> [!CAUTION]
> Vote: 2 in favor, 6 against, 1 abstain.

Reasoning from the review notes:

1. **Latency win is real but small.** The 8-12% JSON overhead on `order-svc` is dwarfed by downstream DB and external-API time. Switching to protobuf would shave ~6ms off a 240ms p99 — under the 25ms noise floor.
2. **Operational cost is high.** Adopting gRPC means: new ingress (HTTP/2, ALPN, header-size tuning), new client load-balancing strategy, new debugging tooling. The team has ~12 weeks of capacity in H2; this would consume most of it.
3. **OpenAPI is the better near-term investment.** Standardizing on OpenAPI + `oapi-codegen` (Go) + `openapi-typescript` gets us shared contracts, generated clients, and breaking-change CI gates at ~1/4 the migration cost.
4. **gRPC may still be right for the streaming-analytics path.** A follow-up RFC scoped to that one workload may make sense if the analytics team confirms the unary-vs-streaming distinction matters for their SLO.

## Implementation Phases

N/A — rejected.

## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| n/a  | n/a    | n/a        | n/a        |

## Success Criteria

N/A — rejected.

## References

- 2026-05-12 architecture review minutes (internal)
- Related: [RFC-0004](./0004-migrate-the-public-api-from-rest-to-graphql.md)

---
id: RFC-0008
title: "Use Bun as the runtime for production Node.js services"
status: Draft
author: Donald Gifford
created: 2026-05-17
---
<!-- markdownlint-disable-file MD025 MD041 -->

# RFC 0008: Use Bun as the runtime for production Node.js services

**Status:** Draft
**Author:** Donald Gifford
**Date:** 2026-05-17

<!--toc:start-->
- [Summary](#summary)
- [Problem Statement](#problem-statement)
- [Proposed Solution](#proposed-solution)
- [Design](#design)
  - [Compatibility gates](#compatibility-gates)
  - [Container image](#container-image)
  - [Observability](#observability)
- [Alternatives Considered](#alternatives-considered)
- [Implementation Phases](#implementation-phases)
  - [Phase 1: notification-svc spike (2 weeks)](#phase-1-notification-svc-spike-2-weeks)
  - [Phase 2: TBD](#phase-2-tbd)
  - [Phase 3: TBD](#phase-3-tbd)
- [Risks and Mitigations](#risks-and-mitigations)
- [Success Criteria](#success-criteria)
- [References](#references)
<!--toc:end-->

## Summary

Evaluate moving our Node.js production services from `node` to `bun` as the runtime. **Draft** — collecting evidence before promoting to Proposed.

> [!NOTE]
> Draft. Several unknowns are flagged inline as `TODO`. This RFC isn't ready for review until they're closed.

## Problem Statement

We run 3 production Node.js services (`notification-svc`, `webhook-fanout`, `frontend-bff`). Pain points:

- Cold-start times have crept up as bundle sizes grew. `frontend-bff` is now at 1.2s cold start; we'd like to halve it.
- The `node`/`npm`/`tsx`/`ts-node`/`vitest` toolchain has 7 moving parts in dev; ramp-up for new hires is friction.
- We've had 2 production incidents in the last 6 months traced to subtle `fetch` polyfill differences between `undici@5` and `undici@6`.

## Proposed Solution

- Bun as both **runtime** and **dev toolchain** for the 3 Node services.
- Keep TypeScript source; let Bun's native TS execution handle dev runs.
- Keep npm-published deps; Bun resolves the existing `package.json` + `package-lock.json`.

> [!CAUTION]
> Bun's Node.js compatibility is high but not 100%. Several Node-API-using native modules (e.g. some Prisma adapters) are still on the gap list. We must inventory production dependencies before committing.

## Design

### Compatibility gates

For each of the 3 services, run before any production commitment:

1. `bun install` against the existing lockfile — must succeed without warnings.
2. Full unit + integration test suite under Bun — must pass at parity with Node.
3. Production-shape load test — must meet or beat current Node latency at 95th percentile.

TODO: enumerate native deps. Known concerns:

- `@grpc/grpc-js` (used in 2 services) — believed compatible as of Bun 1.1.
- `@prisma/client` (used in `notification-svc`) — needs verification.

### Container image

```dockerfile
FROM oven/bun:1-alpine AS build
WORKDIR /app
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile --production
COPY . .
RUN bun build ./src/index.ts --target=bun --outfile=dist/server.js

FROM oven/bun:1-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
USER bun
EXPOSE 8080
ENTRYPOINT ["bun", "run", "dist/server.js"]
```

### Observability

TODO: confirm OTel SDK works under Bun. The Node auto-instrumentations may not all hook the right modules in Bun's resolver.

## Alternatives Considered

- **Stay on Node.** Safe. Doesn't address cold-start or toolchain pain.
- **Deno.** More mature than Bun in some areas; smaller ecosystem in others. The migration cost from Node-shaped code is much higher than from Bun's near-drop-in target.
- **Migrate the 3 services to Go.** Largest performance win and consolidates onto our primary language. Effort is approximately 2 engineer-quarters per service. Worth keeping on the table.

## Implementation Phases

TODO. Will depend on the compatibility-gate findings.

### Phase 1: `notification-svc` spike (2 weeks)

- Smallest service. Lowest risk.
- Run full integration suite under Bun.
- Production canary at 1% for one week.

### Phase 2: TBD

### Phase 3: TBD

## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Native module incompatibility | High | Medium | Compatibility gate before commitment; fall back to Node per service |
| OTel auto-instrumentation gaps | High | Medium | Validate trace + metric output during the spike |
| Bun version churn in production | Medium | High | Pin a specific Bun version; only upgrade behind a Renovate PR with the full test suite green |
| Performance regression vs Node | Medium | Low | Load test gate is mandatory; rollback path documented |
| Team unfamiliarity with Bun debugging tools | Medium | High | Internal training + runbook for `bun` profiling tools |

## Success Criteria

TODO. Notional:

- Cold-start time for `frontend-bff` drops below 600ms.
- All 3 services run on Bun in production with zero unplanned rollbacks for 8 weeks.
- Developer-survey ramp-up time on Node services drops by at least 25%.

## References

- [Bun documentation](https://bun.sh/docs)
- [Bun Node.js compatibility tracking](https://bun.sh/docs/runtime/nodejs-apis)
- TODO: pending native-module compatibility audit.

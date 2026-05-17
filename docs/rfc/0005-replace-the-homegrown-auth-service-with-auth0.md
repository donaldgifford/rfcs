---
id: RFC-0005
title: "Replace the homegrown auth service with Auth0"
status: Draft
author: Donald Gifford
created: 2026-05-17
---
<!-- markdownlint-disable-file MD025 MD041 -->

# RFC 0005: Replace the homegrown auth service with Auth0

**Status:** Draft
**Author:** Donald Gifford
**Date:** 2026-05-17

<!--toc:start-->
- [Summary](#summary)
- [Problem Statement](#problem-statement)
- [Proposed Solution](#proposed-solution)
- [Design](#design)
  - [Token shape](#token-shape)
  - [Migration of existing users](#migration-of-existing-users)
  - [SSO for enterprise tenants](#sso-for-enterprise-tenants)
  - [Session model](#session-model)
- [Alternatives Considered](#alternatives-considered)
- [Implementation Phases](#implementation-phases)
  - [Phase 1: Spike](#phase-1-spike)
  - [Phase 2: TBD](#phase-2-tbd)
  - [Phase 3: TBD](#phase-3-tbd)
- [Risks and Mitigations](#risks-and-mitigations)
- [Success Criteria](#success-criteria)
- [References](#references)
<!--toc:end-->

## Summary

Replace the homegrown auth service (`identity-svc`, ~14k lines of Go) with **Auth0** as the IdP. Issue Auth0-minted JWTs to all clients, validate at the gateway. Keep a thin profile/preferences service for non-auth user data.

> [!NOTE]
> Draft. Sections marked `TODO` need fleshing out before this is ready for review.

## Problem Statement

`identity-svc` has been a chronic source of incidents:

- 4 production outages in 2025 tied to the session-store cache being misconfigured.
- TOTP rollout took 11 months and shipped with a known issue around clock-skew on Android.
- We have no SAML support; two enterprise deals are blocked on it.
- The team that originally built it has fully turned over.

The team's appetite to invest further in `identity-svc` is approximately zero, and the maintenance cost is approximately one full-time engineer.

## Proposed Solution

- Auth0 as the OIDC provider.
- Universal Login for human flows.
- Gateway middleware validates JWTs from Auth0's JWKS.
- `identity-svc` decommissioned after migration; `profile-svc` (new, thin) takes over preferences + non-auth profile fields.

## Design

### Token shape

TODO: confirm `aud`, `iss`, claim set with the platform team. Sketch:

```json
{
  "iss": "https://example.us.auth0.com/",
  "sub": "auth0|abc123",
  "aud": ["https://api.example.com"],
  "scope": "openid profile email",
  "https://example.com/roles": ["customer"]
}
```

### Migration of existing users

TODO: Auth0 supports lazy migration via the [Custom Database connection](https://auth0.com/docs/authenticate/database-connections/custom-db) on first login. Need to spike this to confirm the password-hash format works.

### SSO for enterprise tenants

TODO: SAML + OIDC enterprise connections, one per tenant. Tenant onboarding flow needs design.

### Session model

TODO: stateless JWTs vs Auth0-managed sessions. Initial preference: stateless access tokens (5 min) + refresh tokens (30 day, rotating).

## Alternatives Considered

- **Stay on `identity-svc` + invest.** Rejected — see *Problem Statement*.
- **WorkOS / Clerk / Stytch.** Worth a comparison spike before this is approved.
- **Build SAML support into `identity-svc`.** ~3 months of work for one enterprise feature; doesn't address the underlying maintenance burden.

## Implementation Phases

TODO: phases will depend on the lazy-migration spike outcome.

### Phase 1: Spike

- Lazy migration with our existing password hash format.
- One internal-only application moved end-to-end.

### Phase 2: TBD

### Phase 3: TBD

## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Password-hash format incompatibility | High | Medium | Spike before commitment; fall back to forced reset if needed |
| Vendor lock-in / pricing volatility | Medium | Medium | OIDC is portable; design with switchable IdP in mind |
| Auth0 outage = full login outage | High | Low | Stale-token grace + degraded read-only fallback |
| Migration data loss | High | Low | Dual-source-of-truth window with reconciliation |

## Success Criteria

TODO.

## References

- TODO: pricing / SLA from Auth0 enterprise sales.
- TODO: comparison matrix vs WorkOS, Clerk, Stytch.
- TODO: enterprise SSO requirements from the two deals currently blocked.

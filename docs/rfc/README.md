# Requests for Comments (RFCs)

This directory contains RFCs documenting high-level proposals for major features
and system redesigns.

## What are RFCs?

RFCs document **high-level problem definitions and solution strategies**. Each
RFC focuses on:

- **Problem Statement**: The issue being addressed with evidence
- **Proposed Solution**: High-level approach and architecture
- **Implementation Phases**: Overview of how the solution will be built
- **Alternatives**: Other approaches that were considered
- **Risks and Success Criteria**: What could go wrong and how we measure success

## Creating a New RFC

```bash
docz create rfc "Your RFC Title"
```

## RFC Status

- **Draft**: Initial draft, not yet ready for review
- **Proposed**: Ready for review and feedback
- **Accepted**: Approved and ready for implementation
- **Rejected**: Not moving forward with this proposal
- **Superseded**: Replaced by another RFC

<!-- BEGIN DOCZ AUTO-GENERATED -->
## All RFCs

| ID | Title | Status | Date | Author | Link |
|----|-------|--------|------|--------|------|
| RFC-0001 | Adopt PostgreSQL as the primary application data store | Accepted | 2026-05-17 | Donald Gifford | [0001-adopt-postgresql-as-the-primary-application-data-store.md](0001-adopt-postgresql-as-the-primary-application-data-store.md) |
| RFC-0002 | Use gRPC for all internal service-to-service communication | Rejected | 2026-05-17 | Donald Gifford | [0002-use-grpc-for-all-internal-service-to-service-communication.md](0002-use-grpc-for-all-internal-service-to-service-communication.md) |
| RFC-0003 | Standardize on OpenTelemetry for traces, metrics, and logs | Accepted | 2026-05-17 | Donald Gifford | [0003-standardize-on-opentelemetry-for-traces-metrics-and-logs.md](0003-standardize-on-opentelemetry-for-traces-metrics-and-logs.md) |
| RFC-0004 | Migrate the public API from REST to GraphQL | Superseded | 2026-05-17 | Donald Gifford | [0004-migrate-the-public-api-from-rest-to-graphql.md](0004-migrate-the-public-api-from-rest-to-graphql.md) |
| RFC-0005 | Replace the homegrown auth service with Auth0 | Draft | 2026-05-17 | Donald Gifford | [0005-replace-the-homegrown-auth-service-with-auth0.md](0005-replace-the-homegrown-auth-service-with-auth0.md) |
| RFC-0006 | Adopt an event-driven architecture using NATS JetStream | Proposed | 2026-05-17 | Donald Gifford | [0006-adopt-an-event-driven-architecture-using-nats-jetstream.md](0006-adopt-an-event-driven-architecture-using-nats-jetstream.md) |
| RFC-0007 | Move CI from Jenkins to GitHub Actions | Accepted | 2026-05-17 | Donald Gifford | [0007-move-ci-from-jenkins-to-github-actions.md](0007-move-ci-from-jenkins-to-github-actions.md) |
| RFC-0008 | Use Bun as the runtime for production Node.js services | Draft | 2026-05-17 | Donald Gifford | [0008-use-bun-as-the-runtime-for-production-nodejs-services.md](0008-use-bun-as-the-runtime-for-production-nodejs-services.md) |
<!-- END DOCZ AUTO-GENERATED -->

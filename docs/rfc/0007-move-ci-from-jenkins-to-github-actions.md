---
id: RFC-0007
title: "Move CI from Jenkins to GitHub Actions"
status: Accepted
author: Donald Gifford
created: 2026-05-17
---
<!-- markdownlint-disable-file MD025 MD041 -->

# RFC 0007: Move CI from Jenkins to GitHub Actions

**Status:** Accepted
**Author:** Donald Gifford
**Date:** 2026-05-17

<!--toc:start-->
- [Summary](#summary)
- [Problem Statement](#problem-statement)
- [Proposed Solution](#proposed-solution)
- [Design](#design)
  - [Runner topology](#runner-topology)
  - [Reusable workflow example](#reusable-workflow-example)
  - [Secrets management](#secrets-management)
- [Alternatives Considered](#alternatives-considered)
- [Implementation Phases](#implementation-phases)
  - [Phase 1: Greenfield + reusable workflow library (2 weeks)](#phase-1-greenfield--reusable-workflow-library-2-weeks)
  - [Phase 2: Migrate active services (8 weeks)](#phase-2-migrate-active-services-8-weeks)
  - [Phase 3: Long tail + shutdown (4 weeks)](#phase-3-long-tail--shutdown-4-weeks)
- [Risks and Mitigations](#risks-and-mitigations)
- [Success Criteria](#success-criteria)
- [References](#references)
<!--toc:end-->

## Summary

Replace our self-hosted Jenkins CI cluster with **GitHub Actions** as the primary CI platform. Build agents run on a mix of GitHub-hosted runners (default) and self-hosted runners (for jobs needing GPU, ARM cross-compile, or `secrets-aware` deploys to the internal VPC).

## Problem Statement

Jenkins is showing its age:

- **24 plugins** pinned to specific versions; Renovate cannot upgrade them safely.
- **Pipeline-as-code-but-not-really** — `Jenkinsfile` Groovy is a per-repo dialect that mixes well-typed steps with `sh ''' ... '''` blocks; we have ~3,400 lines of Groovy that nobody enjoys maintaining.
- **Queue contention** at peak (10am–11am, 4pm–5pm) regularly pushes PR build wait times past 30 minutes.
- **No first-class GitHub integration** — required-checks gating, PR comments, deploy reviews all flow through the GitHub App + bespoke glue.
- The Jenkins controller runs on an EOL Ubuntu 18.04 AMI; the upgrade to 22.04 has been on the backlog for 14 months.

> [!WARNING]
> The Jenkins controller is a single point of failure. We do not currently have a tested DR plan; a full restore is estimated at 4–6 hours by the SRE team. This RFC's acceptance includes shutting down that liability.

## Proposed Solution

- GitHub Actions as the CI platform.
- Standard runner image: `ubuntu-24.04`. Self-hosted ARC (Actions Runner Controller) for VPC-bound or GPU jobs.
- Reusable workflows live in a central `.github/workflows/` repo (`org/ci-workflows`); per-repo workflows compose them.

## Design

### Runner topology

| Runner | Concurrency cap | Use case |
|--------|-----------------|----------|
| `ubuntu-24.04` (GitHub-hosted) | 100 | Default — most builds, tests, lints |
| `ubuntu-24.04-arm` (GitHub-hosted) | 20 | ARM cross-builds |
| Self-hosted `internal-vpc` (ARC) | 40 | Deploys to internal-VPC services, secrets-aware |
| Self-hosted `gpu-l4` (ARC) | 8 | ML inference test suite |

### Reusable workflow example

```yaml
# .github/workflows/release.yml (in org/ci-workflows)
on:
  workflow_call:
    inputs:
      go-version: { type: string, default: "1.26" }
      service-name: { type: string, required: true }

jobs:
  build:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ inputs.go-version }}
      - run: go test ./...
      - run: go build -o bin/${{ inputs.service-name }} ./cmd/${{ inputs.service-name }}
```

Per-repo workflow:

```yaml
# .github/workflows/ci.yml (in any service repo)
on: [push, pull_request]
jobs:
  release:
    uses: org/ci-workflows/.github/workflows/release.yml@v1
    with:
      service-name: order-svc
```

### Secrets management

- Repository secrets for service-scoped credentials.
- Organization secrets for shared infra credentials (limited to specific environments).
- OIDC federation to AWS via `aws-actions/configure-aws-credentials` — no long-lived keys.

> [!TIP]
> Pin reusable workflows to a tag (`@v1`), not `@main`. The whole point of having a central workflows repo is the ability to roll forward intentionally.

## Alternatives Considered

- **Stay on Jenkins + upgrade.** Closes the Ubuntu-EOL gap but doesn't address the Groovy maintenance burden or queue contention. Estimated 6 weeks of work, primarily plugin compatibility.
- **Buildkite.** Strong UX, great pipeline-as-code story. Higher per-build cost at our scale; no decisive feature win over GitHub Actions for our use cases.
- **CircleCI.** Similar trade-off to Buildkite; less attractive than GitHub Actions because we don't get the same first-class GitHub integration.
- **GitLab CI.** Would require moving SCM hosting too; out of scope for this RFC.

## Implementation Phases

### Phase 1: Greenfield + reusable workflow library (2 weeks)

- Set up `org/ci-workflows` repo with `release`, `library-publish`, `terraform-plan`, and `terraform-apply` reusable workflows.
- New services use Actions from day one.

### Phase 2: Migrate active services (8 weeks)

- Top 12 services by build volume migrate first. Required check on `main` flips from Jenkins to Actions per service as it lands.

### Phase 3: Long tail + shutdown (4 weeks)

- Remaining ~30 services migrate.
- Jenkins moves to read-only for 30 days, then the cluster is decommissioned.

## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| GitHub-hosted runner cost overrun | Medium | Medium | Per-org spending limit + monthly review; route long-running builds to self-hosted |
| Self-hosted ARC operational burden | Medium | Medium | SRE ownership + runbook; autoscaling baseline tuned in week 1 |
| Workflow migration regressions | Medium | High | Per-service migration PR with `before/after` build-time comparison required in PR description |
| OIDC misconfiguration → AWS access loss | High | Low | Test OIDC role in staging account first; phased rollout |

## Success Criteria

- 100% of services on GitHub Actions within 14 weeks of acceptance.
- p99 PR build wait time below 5 minutes (currently 30+).
- Jenkins cluster decommissioned within 16 weeks.
- Zero unplanned CI downtime caused by Jenkins controller during the migration window.

## References

- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [Actions Runner Controller (ARC)](https://github.com/actions/actions-runner-controller)
- Related: [RFC-0003](./0003-standardize-on-opentelemetry-for-traces-metrics-and-logs.md)

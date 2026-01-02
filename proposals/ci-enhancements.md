# Meta
[meta]: #meta
- Name: Kyverno CI Pipeline Enhancements
- Start Date: Jan 1, 2026
- Update data (optional): 
- Author(s): @JimBugwadia

# Table of Contents
[table-of-contents]: #table-of-contents
- [Meta](#meta)
- [Table of Contents](#table-of-contents)
- [Overview](#overview)
- [Definitions](#definitions)
- [Motivation](#motivation)
- [Proposal](#proposal)
  - [In Scope](#in-scope)
  - [Out of Scope](#out-of-scope)
- [Implementation](#implementation)
  - [1. Centralize Image Building](#1-centralize-image-building)
  - [2. Implement Caching](#2-implement-caching)
  - [3. Workflow Dependencies](#3-workflow-dependencies)
  - [4. Job Parallelization Improvements](#4-job-parallelization-improvements)
  - [5. Build Environment Optimization](#5-build-environment-optimization)
  - [7. Conditional Execution](#7-conditional-execution)
  - [8. Multi-State Pipeline](#8-multi-state-pipeline)
    - [Overall Strategy](#overall-strategy)
    - [Handling failures in the Post-Merge Pipeline](#handling-failures-in-the-post-merge-pipeline)
    - [Initial Implementation](#initial-implementation)
  - [Link to the Implementation PRs](#link-to-the-implementation-prs)
- [Migration (OPTIONAL)](#migration-optional)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Prior Art](#prior-art)
- [Unresolved Questions](#unresolved-questions)
- [CRD Changes (OPTIONAL)](#crd-changes-optional)

# Overview
[overview]: #overview

Dramatically speed up CI times while retaining core value of comprehensive testing.

# Definitions
[definitions]: #definitions

* `CI`: Continious Integration

# Motivation
[motivation]: #motivation

It currently takes several hours to merge a PR in the [kyverno/kyverno](https://github.com/kyverno/kyverno/) repository.

As an example, [PR #14590](https://github.com/kyverno/kyverno/pull/14590) took ~6 hours 12 minutes to complete conformance checks. All conformance checks are performed across 3 Kubernetes versions, for every change.

This is a frustrating developer experience that kills productivity and increases barriers to contributing. These ineffeciencies cause a huge pile up of open PRs, that sometimes have been around for months. 

Our goal should be for CI checks to complete and the mean `time to merge` be 5 minutes or less.

# Proposal

Getting to 5 minutes will not be easy and will require several techniques. Here is what is in-scope for this KDP, and other items that are currently out-of-scope and will be considered separately.

## In Scope
   
1. **Reusing Images**: Images are build several times in various CI jobs, and each run takes ~4 mins. This can be optimized.
   
2. **Caching**: We are not caching Go modules and other artifacts. This adds several minutes in each job.
   
3. **Multi-Stage CI Pipelines**: We don't need to run all CI checks for each change. A multi-stage pipeline can run fast & critical on each PR and then trigger longer and more expensive checks when the changes are merged in main. To prevent cascading failures, the CI can be blocked when there is a  

## Out of Scope

1. **Chainsaw Enhancements**: Chainsaw relies on external clusters. It can be updated to use the Kubernetes fake client and envtest tools.

2. **Replacing Chainsaw tests with unit tests**: Unit tests a several times faster (>100X) than e2e tests. Several Chainsaw tests can be replaced by unit tests, especially for core feature capabilities.

# Implementation

## 1. Centralize Image Building

Create a reusable workflow that builds images once and makes them available to all workflows:**New workflow: `.github/workflows/build-images.yaml`**

- Triggered on PRs and pushes
- Builds all images using `ko-build-all`
- Uploads images as artifacts (`kyverno.tar`)
- Outputs image digests/tags for reuse
- Uses GitHub Actions cache for ko build cache

**Update dependent workflows:**

- `conformance.yaml`: Remove `prepare-images` job, download artifacts from `build-images` workflow
- `load-testing.yml`: Remove `prepare-images` job, download artifacts from `build-images` workflow
- `images-build.yaml`: Can be deprecated or refactored to use the centralized workflow

## 2. Implement Caching

**Ko Build Cache:**

- Cache `KOCACHE` directory (`/tmp/ko-cache`) using `actions/cache`
- Key: `ko-cache-${{ runner.os }}-${{ hashFiles('go.sum', '.ko.yaml', '**/Dockerfile') }}`
- Restore cache before building, save after building

**Go Module Cache:**

- Cache `${{ env.GOMODCACHE }}` and `${{ env.GOCACHE }}` in `setup-build-env` action
- Key: `go-mod-${{ runner.os }}-${{ hashFiles('**/go.sum') }}`

**Docker Layer Cache:**

- Use `docker/build-push-action` with cache-from/cache-to if migrating from ko
- Or cache Docker buildx cache directory

## 3. Workflow Dependencies

**Use `workflow_run` trigger:**

- Make `conformance.yaml` and `load-testing.yml` depend on `build-images` workflow completion
- Download artifacts from the completed workflow run
- This ensures images are built once and reused

## 4. Job Parallelization Improvements

**Image Building:**

- Build images in parallel using matrix strategy (6 images = 6 parallel jobs)
- Each job builds one image, uploads as separate artifact
- Download and combine artifacts when needed

**Test Execution:**

- Already well parallelized with matrix strategies
- Consider grouping related test suites to reduce job overhead

## 5. Build Environment Optimization

**Setup Build Env Action:**

- Add Go module caching to `.github/actions/setup-build-env/action.yaml`
- Cache tool installations (ko, kind, etc.) if they don't change frequently
- Use `actions/setup-go@v6` with built-in caching

**Reduce Setup Time:**

- Pre-install common tools in composite actions
- Use action version pinning (already done) to leverage GitHub's action cache

## 7. Conditional Execution

**Path-based Triggers:**
- Only build images if relevant files changed (already partially done in `helm-test.yaml`)
- Use `paths` filter for image building workflow
- Skip image builds if only documentation changed

**Skip Unnecessary Steps:**
- Skip Trivy scans in `images-build.yaml` if images will be scanned in `images-publish.yaml`
- Consolidate security scanning to one location

## 8. Multi-State Pipeline

### Overall Strategy

The basic idea is to have two separate workflow files triggered by different events: 
* `pull_request` for the fast checks, and; 
* `push` (targeting the main branch) for the slower, post-merge checks.

This separation ensures that developers get immediate feedback while keeping the repository's main line stable without slowing down the development cycle.

1. The `Fast CI Workflow (fast-ci.yml)`. This workflow triggers on every Pull Request. It should focus on "fail-fast" mechanisms like linting, unit tests, and security scanning.

```yaml
name: Fast CI Checks
on:
  pull_request:
    branches: [ main ]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install dependencies
        run: npm ci
      - name: Run Linter
        run: npm run lint
      - name: Run Unit Tests
        run: npm test -- --shard=1/2 # Example of parallelizing
```

2. The `Slow CI Workflow (slow-ci.yml)`. This workflow triggers only after code is successfully merged into the main branch. This is where you run heavy end-to-end (E2E) tests, performance benchmarks, or complex integration suites.

```yaml
name: Post-Merge Heavy Checks
on:
  push:
    branches: [ main ]

jobs:
  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Environment
        run: ./setup-heavy-env.sh
      - name: Run Integration Suite
        run: npm run test:e2e
  
  deploy-staging:
    needs: e2e-tests
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Staging
        run: ./deploy.sh staging
```

**Key Strategies for Efficiency**

| Feature | PR Pipeline (Fast) | Post-Merge Pipeline (Slow) |
|---------|--------------------|----------------------------|
| Goal | Developer feedback in < 5 mins | Deep validation & Deployment |
| Trigger | pull_request | push (to main) |
| Typical Tasks | Linting, Unit Tests, Type Checking | E2E Tests, Stress Tests, Security Audits |
| Cost | High frequency, low resource | Low frequency, high resource |


### Handling failures in the Post-Merge Pipeline

Since the second pipeline runs after a merge has already occurred, it cannot retroactively "un-merge" that code. Instead, we can use a strategy that locks the front door for any subsequent PRs until the "main" branch is healthy again.

**The "Broken Master" Check**

The most effective way to block subsequent PRs is to add a Status Check to the PR pipeline that queries the health of the main branch. If the last run of your "Post-Merge Pipeline" failed, this check fails, effectively blocking the "Merge" button on all open PRs.

We can use a community action like bennycode/stop-merging or write a simple script using the GitHub CLI (gh):

```yaml
# Add this job to your Fast CI (PR) workflow
jobs:
  check-main-health:
    runs-on: ubuntu-latest
    steps:
      - name: Check if Main is Green
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          # Get the status of the last 'Slow CI' run on main
          STATUS=$(gh run list --workflow "slow-ci.yml" --branch main --limit 1 --json conclusion -q '.[0].conclusion')
          
          if [ "$STATUS" != "success" ]; then
            echo "❌ Main branch is currently broken (Slow CI failed). Merging is blocked."
            exit 1
          fi
          echo "✅ Main branch is healthy."
```

Action Required: In Settings > Branches  add check-main-health as a Required Status Check for your main branch protection rule.


### Initial Implementation

To start with, we can move the following tests to the post-merge pipeline:
1. **Policy repo tests**: these tests use the sample policies and rely on an external repository. Many times 
2. **Load tests**: load tests do not need to be executed 

## Link to the Implementation PRs

* https://github.com/kyverno/kyverno/issues/14290

# Migration (OPTIONAL)

Not Applicable.

# Drawbacks

The Multi-Stage Pipeline introduces the possibility that issues are not caught in the PR pipeline.

# Alternatives

Stay with the status quo.

# Prior Art

Not Applicable.

# Unresolved Questions

1. We need to investigate the GitHub Merge Queue feature. See slack thread: https://kubernetes.slack.com/archives/C032MM2CH7X/p1767336106402949. It may be able to complement the multi-stage CI approach.

# CRD Changes (OPTIONAL)

Not Applicable.
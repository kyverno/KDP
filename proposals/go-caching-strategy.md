# Meta
[meta]: #meta
- Name: Go CI Caching Strategy
- Start Date: 2025-03-13
- Update date (optional): 2025-03-13
- Author(s): (Github usernames)
- Supersedes: N/A

# Table of Contents
[table-of-contents]: #table-of-contents
- [Meta](#meta)
- [Table of Contents](#table-of-contents)
- [Overview](#overview)
- [Definitions](#definitions)
- [Motivation](#motivation)
- [Proposal](#proposal)
- [Implementation](#implementation)
  - [Cache inheritance and isolation across branches](#cache-inheritance-and-isolation-across-branches)
- [Migration (OPTIONAL)](#migration-optional)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Prior Art](#prior-art)
- [Unresolved Questions](#unresolved-questions)
- [CRD Changes (OPTIONAL)](#crd-changes-optional)

# Overview
[overview]: #overview

This KDP documents Kyverno’s GitHub Actions strategy for caching Go module and build output. We use custom composite actions (go/setup, go/restore-cache, go/save-cache) to separate module cache (GOMODCACHE) and build cache (GOCACHE), control cache keys and who saves, and stay within GitHub’s cache limits while maximizing hit rates and avoiding workflows overwriting each other’s useful cache.

# Definitions
[definitions]: #definitions

- **GOMODCACHE** – Directory where the Go toolchain stores downloaded modules (e.g. `~/go/pkg/mod`). Output of `go env GOMODCACHE`.
- **GOCACHE** – Directory where the Go toolchain stores build and test cache (e.g. `~/.cache/go-build`). Output of `go env GOCACHE`.
- **Cache key** – Immutable identifier for a GitHub Actions cache entry. Exact key match is required to restore that exact entry; partial keys (restore-keys) allow prefix matching for fallback.
- **restore-keys** – List of key prefixes used when no exact key match exists. The cache action uses the most recent cache whose key starts with one of these prefixes ([GitHub Docs](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows)).
- **Cache prefix** – User-chosen string (e.g. `gomod`, `gobuild`, `gobuild-unit-tests`) that namespaces cache keys so different workflows or job types don’t collide.
- **CACHEDATE** – Date in `YYYY-MM-DD` form included in the cache key so entries can be refreshed over time (e.g. daily) while still reusing recent caches.
- **Cache scope (branch)** – GitHub scopes cache entries to a branch (or, for pull requests, to the PR merge ref). Restore can see caches from the current branch and from the default branch; PR runs can also see caches from the PR’s base branch. Caches created on a PR are only restorable by that same PR ([GitHub: Restrictions for accessing a cache](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows#restrictions-for-accessing-a-cache)).

# Motivation
[motivation]: #motivation

- **Faster CI**: Restoring module and build cache reduces `go mod download` and rebuild time on every run.
- **Controlled saving**: Only designated jobs (e.g. Caching workflow on `main`, tests on `main`) write cache, so shorter or narrower jobs don’t overwrite richer caches ([Dan Peterson](https://danp.net/posts/github-actions-go-cache/)).
- **Separate caches**: Module cache and build cache have different lifecycles and key strategies; combining them in a single key (as `actions/setup-go` does by default) can lead to one workflow saving a cache that is a poor fit for another (e.g. build-only vs test-with-race).
- **Respect GitHub limits**: Default cache is 10 GB per repository with 7-day retention; keys and save policy must avoid unbounded growth and thrashing ([GitHub dependency caching](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows)).
- **Predictable keys**: Keys must be consistent between restore and save (same prefix, OS, arch, Go version, date, and content hash) so that the same job type reuses the right cache.

# Proposal

- **Two cache namespaces**: (1) **gomod** – module cache only; (2) **gobuild** (and optionally workflow-specific prefixes like **gobuild-unit-tests**) – build cache only. Workflows choose which to restore and/or save via `go/setup` inputs and explicit `go/save-cache` steps.
- **Explicit restore/save**: We do not use `actions/setup-go`’s built-in cache (`cache: false`). We use our own `go/restore-cache` and `go/save-cache` so key format, restore-keys, and which job saves are under our control.
- **Key format**:  
  `{cache-prefix}-{runner.os}-{ARCH}-{OS}-{GOVERSION}-{CACHEDATE}-{hashFiles('**/go.sum')}`  
  - **ARCH**: runner arch lowercased.  
  - **OS**: `ImageOS` when set (e.g. on some hosted runners), otherwise `Unknown`.  
  - **GOVERSION**: `go env GOVERSION`.  
  - **CACHEDATE**: `date +%Y-%m-%d` so keys rotate by day and we can get “fresh enough” cache via restore-keys.  
  - **hashFiles('**/go.sum')**: Ensures key changes when dependencies change.
- **Restore-keys**: Two fallback prefixes so we can hit the latest cache for the same prefix/OS/arch/Go version even if the exact date or go.sum hash differs:  
  - `{prefix}-{runner.os}-{ARCH}-{OS}-{GOVERSION}-{CACHEDATE}-`  
  - `{prefix}-{runner.os}-{ARCH}-{OS}-{GOVERSION}-`
- **Who saves**:
  - **Caching workflow** (on `main`): One job restores nothing, runs `go mod download` (and tidy/install-tools), then saves **gomod**. Another job restores **gomod** only, runs full build, then saves **gobuild**. This keeps module cache full and build cache populated by a full build.
  - **Tests workflow**: Restores **gobuild-unit-tests** (distinct prefix), runs unit tests, saves **gobuild-unit-tests** only on `push` to `main`. So test runs benefit from and refresh test-build cache without overwriting the main **gobuild** cache.
  - Other workflows (lint, verify-codegen, conformance, etc.) only restore (with configurable prefixes); they do not save, avoiding cache pollution and staying under the 10 GB limit.
- **Optional restore**: `go/setup` accepts `gomod-cache-prefix` and `gobuild-cache-prefix`; set to `''` to disable that restore (e.g. Caching workflow disables both on the gomod job and only saves after downloading).

# Implementation

The strategy is implemented in the Kyverno repo as follows.

- **`.github/actions/go/install`**  
  Installs Go via `actions/setup-go` with `cache: false`, then exports `gomod-cache` and `gobuild-cache` (GOMODCACHE and GOCACHE paths) as outputs.

- **`.github/actions/go/restore-cache`**  
  Inputs: `path`, `cache-prefix`. Computes GOVERSION, CACHEDATE, ARCH, OS (with `ImageOS` defaulting to `Unknown`), builds the key and restore-keys as above, and calls `actions/cache/restore`.

- **`.github/actions/go/save-cache`**  
  Inputs: `path`, `cache-prefix`. Uses the same key formula (with the same variables step) and calls `actions/cache/save`. Restore and save keys are identical for the same inputs and day.

- **`.github/actions/go/setup`**  
  Runs `go/install`, then conditionally runs `go/restore-cache` for gomod and/or gobuild using install outputs and the configured prefixes. Exposes `gomod-cache` and `gobuild-cache` (pass-through from install) for later `go/save-cache` steps.

- **`.github/workflows/cache.yaml`** (runs on push to `main`)  
  - **gomod job**: Setup with both cache prefixes disabled; `go mod download` (and tidy, install-tools); save gomod cache.  
  - **gobuild job**: Setup with only gomod restore enabled (gobuild disabled); install ko; full build; save gobuild cache.

- **`.github/workflows/tests.yaml`**  
  Setup with `gobuild-cache-prefix: gobuild-unit-tests` (gomod uses default restore). After unit tests, save gobuild cache with prefix `gobuild-unit-tests` only when `github.event_name == 'push'` and `github.ref == 'refs/heads/main'`.

Other workflows (lint, verify-codegen, conformance, images-build, etc.) call `go/setup` with default or overridden prefixes and do not run `go/save-cache`, so they only consume cache and stay within the 10 GB limit.

## GitHub Actions cache constraints we respect

- **Immutability**: Each key creates at most one cache entry; we never “update” an entry, only create new keys (e.g. new CACHEDATE or new go.sum hash) and use restore-keys to match recent entries ([actions/cache tips](https://github.com/actions/cache/blob/main/tips-and-workarounds.md)).
- **Size and retention**: Default 10 GB per repository; eviction is least-recently-used. We limit who saves (Caching + tests on main only) and use a small set of prefixes to avoid unbounded key growth ([GitHub dependency caching](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows)).
- **Key stability**: Restore and save use the same key formula so that a job that saves is able to restore the same key on a later run when inputs (date, go.sum) align; restore-keys handle the case when only a newer or older day’s cache exists.

### Cache inheritance and isolation across branches

GitHub does **not** store caches in a single global pool per repository. Each cache entry is associated with the **branch (or ref) that created it**. Restore then follows a fixed search order and scope rules ([GitHub: Cache key matching](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows#cache-key-matching), [Restrictions for accessing a cache](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows#restrictions-for-accessing-a-cache)):

1. **Search order**: The cache action looks for a matching key (exact, then prefix/restore-keys) **in the branch of the current run first**, then **in the default branch** (e.g. `main`). So a run on a feature branch or PR will fall back to the default branch's caches if there is no hit in the current branch.

2. **Who can restore what**:
   - **Default branch runs** can restore caches created on the default branch.
   - **Feature-branch runs** (e.g. `push` to `feature-x`) can restore caches from that branch and from the default branch. So they **inherit** main's caches.
   - **Pull request runs** can restore caches from: the **PR merge ref** (current), the **default branch**, and the **base branch** of the PR. So a PR targeting `main` can use caches created on `main`; a PR from `feature-b` into `feature-a` can use caches from `main`, `feature-a`, and `feature-b`.
   - **Child/sibling isolation**: A run on the default branch **cannot** restore a cache created on a feature branch. A run on branch A **cannot** restore a cache created on a sibling branch B (same base). So caches created on PR or feature branches are **isolated** from main and from other branches.

3. **Who can create (save) what**:
   - Caches **saved** on a **push** to a branch are stored for that branch (e.g. `main` or `feature-x`).
   - Caches **saved** during a **pull_request** run are stored for the **PR merge ref** (`refs/pull/<id>/merge`). Those caches are **only restorable by re-runs of the same PR**; they are not visible to the base branch or to other PRs. That avoids PRs polluting the default branch cache but also means PR caches are not shared across PRs from the same source branch.

**How our strategy uses this:** We **do not include the branch name** (e.g. `github.ref`) in the cache key. We **only save** from the **Caching workflow** and from **tests on push to `main`**. So all saved caches live on the default branch. As a result:

- **PRs and feature branches** only **restore**: they see no cache on their own ref, then fall back to `main` and get the gomod/gobuild (or gobuild-unit-tests) cache produced by main. They **inherit** main's cache without ever creating branch-specific or PR-specific entries.
- **Main** both restores and saves; it refreshes the cache that everyone else uses.
- We avoid creating caches on PRs, so we never create the limited-scope PR merge ref caches that would be isolated to a single PR and useless to other runs.

## Link to the Implementation PR

Implementation lives in the Kyverno repository under `.github/actions/go/` and `.github/workflows/cache.yaml`, `.github/workflows/tests.yaml`. There is no single “implementation PR” for this KDP; it describes the current design and rationale.

# Migration (OPTIONAL)

N/A. This KDP describes the existing design. Migrations from other caching approaches (e.g. switching from `actions/setup-go` with `cache: true` to this strategy) would involve: (1) Adding `go/setup` with desired prefixes and optionally disabling built-in setup-go cache; (2) Adding `go/save-cache` only where we want to write cache (e.g. Caching workflow, tests on main); (3) Ensuring workflows that need cache paths use `steps.<setup-step-id>.outputs.gomod-cache` and `gobuild-cache`.

# Drawbacks

- **Complexity**: More moving parts than using `actions/setup-go` with `cache: true`; contributors must understand prefixes and which jobs save.
- **Key churn**: Including CACHEDATE and go.sum in the key creates new keys frequently; restore-keys mitigate but we still store multiple entries per prefix over time, using more of the 10 GB.
- **Runner variance**: `ImageOS` is not set on all runners; we use `Unknown` as default so keys are consistent but may not distinguish actual OS variants on some environments.
- **No automatic trim**: We do not currently trim GOCACHE (e.g. by deleting old build artifacts before save) as suggested in [Dan Peterson’s article](https://danp.net/posts/github-actions-go-cache/); doing so could reduce cache size and eviction pressure at the cost of extra logic and maintainability.

# Alternatives

- **Use `actions/setup-go` with `cache: true`**: Simpler, but one key combines module and build cache and any job can save; a fast build job can overwrite a cache that tests need, and we lose control over key shape and retention ([Dan Peterson](https://danp.net/posts/github-actions-go-cache/)). We rejected this to avoid “the faster-running artifacts workflow was sneaking in and saving a cache item” and leaving CI without the right build output.
- **Single cache, single key**: One path and one key for both GOMODCACHE and GOCACHE. Simpler but couples module and build lifecycle and increases the chance that one workflow’s save hurts another; we prefer separate gomod and gobuild (and optional workflow-specific gobuild prefixes).
- **No date in key**: Keys would only change when Go version or go.sum change. Cache could become very stale; we include CACHEDATE so that we can refresh periodically and still use restore-keys to get the latest available.
- **Save from every workflow**: Would maximize local hit rate but quickly exhaust the 10 GB limit and risk eviction of the caches that help most; we restrict saving to the Caching workflow and tests on main.

# Prior Art

- **Dan Peterson, “Better GitHub Actions caching for Go”** ([danp.net](https://danp.net/posts/github-actions-go-cache/)): Describes problems with `setup-go`’s combined cache (one workflow saving and another missing modules/build output), suggests separating restore/save control, using a date in the key for freshness, and trimming the build cache before save. Our design follows the same ideas: we disable setup-go cache, use our own keys with a date component, and limit who saves so the “more involved CI workflow” (and our Caching workflow) own cache production.
- **GitHub Actions cache documentation**: [Caching dependencies to speed up workflows](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows) and [actions/cache](https://github.com/actions/cache) (including [tips-and-workarounds.md](https://github.com/actions/cache/blob/main/tips-and-workarounds.md)): Immutability, restore-keys, and the 10 GB default limit and eviction behavior inform our key design and save policy.
- **actions/setup-go**: Uses the same underlying cache backend; we set `cache: false` and replicate key logic in our actions so we can split gomod/gobuild and control which jobs save.

# Unresolved Questions

- Should we add a GOCACHE trim step (e.g. delete build cache files older than 90 minutes) before save in the Caching or tests workflow to reduce stored size and stay under 10 GB longer?
- Should CACHEDATE be optional (e.g. input to restore/save) so some workflows can use a key without date for longer-lived, less granular cache?
- Do we want to document or enforce a maximum number of cache prefixes to avoid accidental proliferation and quota pressure?
- Should `ImageOS` be passed explicitly as an input to restore-cache/save-cache instead of relying on the environment variable and `Unknown` default for consistency across runners?

# CRD Changes (OPTIONAL)

None. This KDP only concerns CI workflow and action design, not Kubernetes CRDs or Kyverno policy API.

# Meta
[meta]: #meta
- Name: Global Context Entry Caching Enhancements
- Start Date: 2024-12-17
- Author(s): @KhaledEmaraDev

# Table of Contents
[table-of-contents]: #table-of-contents
- [Meta](#meta)
- [Table of Contents](#table-of-contents)
- [Overview](#overview)
- [Definitions](#definitions)
- [Motivation](#motivation)
- [Proposal](#proposal)
- [Implementation](#implementation)
- [Migration (OPTIONAL)](#migration-optional)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Prior Art](#prior-art)
- [Unresolved Questions](#unresolved-questions)
- [CRD Changes (OPTIONAL)](#crd-changes-optional)

# Overview
[overview]: #overview

This proposal aims to enhance the performance of Kyverno's Global Context Entry feature by introducing caching mechanisms for JMESPath expressions. Currently, these expressions are evaluated on every lookup, which is expensive. This proposal explores two approaches: one that moves JMESPath evaluation to write time by storing it in the `GlobalContextEntry` CRD, and another approach using a cache keyed by the Global Context Entry name and the `jmesPath` expression. This document also introduces a `ttl` attribute for both solutions.

# Definitions
[definitions]: #definitions

*   **Global Context Entry:** A Kyverno CRD that caches Kubernetes resources or API call results.
*   **JMESPath:** A query language for JSON data, used here to extract specific parts of the returned API data or resource.
*   **TTL (Time To Live):**  The duration for which data is considered valid in a cache before it is refreshed or invalidated.
*   **Write Time:** The time when data is initially stored or cached. In the case of the Global Context Entry this is the time when the data from Kubernetes or the API Call is retrieved.
*   **Lookup Time:** The time when data is retrieved from a cache or source.

# Motivation
[motivation]: #motivation

-   **Why should we do this?** Evaluating JMESPath expressions on every lookup is inefficient and can lead to performance bottlenecks, especially with complex expressions or frequent access. This impacts Kyverno's overall performance and scalability, particularly in larger clusters.
-   **What use cases does it support?** This change will improve the performance of policies that utilize the Global Context Entry, especially those policies that use frequent lookups.
-   **What is the expected outcome?** The expected outcome is improved policy performance due to less resource usage when making calls to the Global Context Entry.

# Proposal
[proposal]: #proposal

This proposal details two potential caching mechanisms for the Global Context Entry feature. Both mechanisms aim to improve performance by reducing the frequency of JMESPath evaluations. Additionally, they propose the addition of a `ttl` attribute.

**Mechanism 1: JMESPath Evaluation at Write Time**

In this approach, the `jmesPath` expression is moved from the `globalReference` context definition to the `GlobalContextEntry` CRD definition under either the `kubernetesResource` or the `apiCall` spec. This means that when the GlobalContextEntry data is fetched and stored, the `jmesPath` expression is applied **once** and the resulting data is cached.

```yaml
apiVersion: kyverno.io/v2alpha1
kind: GlobalContextEntry
metadata:
  name: deployments
spec:
  kubernetesResource:
    group: apps
    version: v1
    resource: deployments
    namespace: fitness
  jmesPath: length(@) # <== JMESPath moved here
  ttl: 60s # <== Added TTL attribute
```
or
```yaml
apiVersion: kyverno.io/v2alpha1
kind: GlobalContextEntry
metadata:
  name: deployments
spec:
  apiCall:
    urlPath: "/apis/apps/v1/namespaces/fitness/deployments?labelSelector=app=blue"
    refreshInterval: 10s
  jmesPath: length(@) # <== JMESPath moved here
  ttl: 60s # <== Added TTL attribute
```

The `globalReference` definition in the policy context would now only need the name of the GlobalContextEntry.

```yaml
context:
  - name: deployments
    globalReference:
      name: deployments
```

**Mechanism 2: Caching with Combined Key**

In this approach, we retain the current design of applying the `jmesPath` on lookup and introduce a cache. The cache's key is a combination of the GlobalContextEntry's `name` and the `jmesPath` expression. The cache's value is the result of applying the `jmesPath` expression.

```yaml
context:
  - name: deployments
    globalReference:
      name: deployments
      jmesPath: length(@)
      ttl: 60s # <== TTL attribute added to globalReference.
```
With this approach, we are now caching the result of `length(@)` for GlobalContextEntry `deployments`. When this entry is requested with a different `jmesPath`, then the cache will be a miss and will re-evaluate.

**TTL Attribute**

For both mechanisms, a Time-To-Live (TTL) attribute is introduced. This can either be added under the `GlobalContextEntry` or under `globalReference`, depending on the approach. This attribute specifies the duration for which the cached data is considered valid. After the TTL expires, the cached data is refreshed from Kubernetes or the API endpoint. For the first approach, the `ttl` is applied to the whole Global Context Entry. For the second approach, it's on the specific cached entry that's a result of that `jmesPath` being applied.

**User Experience**

*   **Mechanism 1 (JMESPath at Write Time):** For existing users of the Global Context Entry, this change will involve moving the `jmesPath` expression from policy context to the `GlobalContextEntry`. It simplifies the policy, as it only requires the GlobalContextEntry name and it's less verbose. New users will learn to configure the `jmesPath` within the GlobalContextEntry from the start.
*   **Mechanism 2 (Combined Key Cache):** For existing users, this is largely transparent.  They might have to specify a `ttl` if they wish to. New users will learn to use the `ttl` within the context definition.

# Implementation
[implementation]: #implementation

**Mechanism 1 (JMESPath at Write Time):**

1.  **CRD Modification:** Modify the `GlobalContextEntry` CRD to include a `jmesPath` field. A `ttl` field would be added to the top-level of the CRD spec.
2.  **Data Retrieval:** When fetching data from Kubernetes or an API endpoint, the specified `jmesPath` expression is immediately applied to the retrieved data before caching it.
3.  **Cache Storage:** The result of the `jmesPath` evaluation is stored in the cache, keyed by the Global Context Entry name.
4.  **Policy Context Retrieval:** When the `globalReference` is used, the cached result is returned.
5. **TTL Management:** The cache entry will be invalidated and updated if the `ttl` has expired.

**Mechanism 2 (Combined Key Cache):**

1. **Cache Implementation:** Implement a cache that stores the result of `jmesPath` evaluations. This cache will be keyed by `GlobalContextEntry.name` and the `jmesPath` expression used to evaluate the data.
2. **Data Retrieval:** When a `globalReference` is used, the cache is checked for an existing entry using the GlobalContextEntry's `name` and the `jmesPath` provided.
3. **Cache Hit:** If a cache hit is found and the entry is not expired, return the cached value.
4. **Cache Miss:** If a cache miss occurs, fetch the data from Kubernetes or an API endpoint, evaluate the `jmesPath`, store the result in the cache with a `ttl` configured.
5. **TTL Management**: The cache entry will be invalidated and updated if the `ttl` has expired.

**General Implementation Notes:**

*   The cache implementation can use an in-memory cache with a cleanup process based on the configured TTL.
*   Error handling for invalid `jmesPath` expressions should be added, and these errors should be reported to the user via the Kyverno events and in the logs.
*  Metrics must be updated to track cache hit rate.

# Migration
[migration-optional]: #migration-optional

**Mechanism 1:** Requires changes to existing GlobalContextEntries. It is not transparent and will require user intervention. Users will need to move the `jmesPath` to the GlobalContextEntry and remove it from the policy context. This approach would also require a new Kyverno release to update the CRD to reflect this change.
**Mechanism 2:** Is largely transparent as a new `ttl` is added to the globalReference which is optional and will not affect existing users.

# Drawbacks
[drawbacks]: #drawbacks

**Mechanism 1:**
*   The primary drawback is that it is less flexible. Each `GlobalContextEntry` can only hold one specific view of the data, determined by the `jmesPath`. If a policy needs a different `jmesPath` expression on the same data, a new `GlobalContextEntry` must be created.
* Requires user intervention and is not backwards compatible.

**Mechanism 2:**
* Adds complexity to the caching mechanisms.

# Prior Art
[prior-art]: #prior-art

*   Kyverno already uses caching for Kubernetes resource lookups. This proposal extends that by integrating JMESPath results within the cache system.
*  Other systems like data pipelines or caching services often employ similar strategies of evaluating expressions and results and storing with an optional TTL.

# Unresolved Questions
[unresolved-questions]: #unresolved-questions

*  Should we have an additional global cache for cached data? It's currently per-controller, does this work?

# CRD Changes (OPTIONAL)
[crd-changes-optional]: #crd-changes-optional

**Mechanism 1:** Requires a change to the GlobalContextEntry CRD and a new Kyverno release.

**Mechanism 2:** No CRD changes are required but a new field is added to the policy CRD.

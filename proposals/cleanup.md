# Meta

- Name: Clean-up
- Start Date: 2022-07-30
- Author(s): chipzoller
- Supersedes: N/A

# Table of Contents

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

A clean-up ability of Kyverno allowing it to clean-up (delete) resources in a Kubernetes cluster based upon certain criteria.

# Definitions

- TTL: Time-to-live. The amount of time a resource may exist until it is cleaned up.

# Motivation

The initial idea for this feature came from issue [3483](https://github.com/kyverno/kyverno/issues/3483) authored by user luisdavim.

As long as there has been Kubernetes, there has been the inevitability that some resources--most especially those created directly by human users--are left derelict at some point without a good way to identify what and where they are. As Kubernetes continues to expand with its myriad of possible custom resources, this problem grows over time. The ultimate problems this poses are few: 1) Kubernetes' key-value store, which has fairly limited storage capacity, may become overwhelmed by resources which should be deleted; and 2) these resources, in the case of Pods, may result in consumption of excess resources driving up cost and reducing workload consolidation abilities. The solution to these problems is to identify the derelict resources and remove (i.e., delete) them.

There are two main use cases associated with this.

1. Explicit, per-resource clean-up declaration which may be set by a user either upon creation or retroactively.

2. Implicit, bulk clean-up of resources on a scheduled basis which meet a certain criteria.


# Proposal

In this proposal, Kyverno may function as a clean-up controller allowing it to identify and scavenge resources. There are two proposed capabilities which would be used to effect a clean-up.

1. Use of two specific annotations which may be set on any resource:
   - `kyverno.io/ttl`: Sets the time-to-live a resource may have which functions as a count-down timer starting from the time at which the annotated resource was either created or when the annotation was assigned. Value is a string and units are given in either minutes, hours, or days. Ex., `kyverno.io/ttl: 30m`, `kyverno.io/ttl: 30d`
   - `kyverno.io/expires`: Sets the absolute date and time at which clean-up should proceed. Must conform to ISO 8601 standards. Ex., `kyverno.io/expires: 2022-08-04T00:30:00Z`, `kyverno.io/expires: 2022-09-30` 
2. Use of a new rule type tentatively called `cleanup` which will execute in background mode on a schedule given in cron format. The rule will reuse existing Kyverno policy expression paradigms and capabilities to minimize introduced complexity.

The functional requirements of both capabilities are defined below.

```
1. Annotations:
    a. kyverno.io/ttl
        i. must support minimally 1m. No seconds. Duration units are m, h, or d.
        ii. Must allow both assignment to new (incoming) resources as well as assignment to existing resources
        iii. Must allow alteration after the fact
        iv. Must allow deletion to cancel the future clean-up action
        v. Must provide mechanism by which cluster operator (Kyverno CLI) can query across a cluster and return all resources which will be cleaned up
    b. kyverno.io/expires
        i. Must support absolute time in future in at least one standardized time format (ISO 8601 recommended)
        ii. Must allow both assignment to new (incoming) resources as well as assignment to existing resources
        iii. Must allow alteration after the fact
        iv. Must allow deletion to cancel the future clean-up action
        v. Must provide mechanism by which cluster operator (Kyverno CLI) can query across a cluster and return all resources which will be cleaned up
    c. General:
        i. Must log clean-up of any and all resources in, minimally, the Kyverno log
        ii. Must support clean-up of both Namespaced and cluster-scoped resources (including custom)
        iii. Must only pay attention to these defined annotations (i.e., user cannot define a custom annotation as a replacement for these built-in, special Kyverno annotations)
        iv. If both ttl and expires are specified, either Kyverno ignores the resource or it operates on the sooner of the two. Validate policy may be written which alternatively prevents this condition (account for both creation and updating a resource to add the second annotation)
2. Rule:
    a. Must support definition in both Policy and ClusterPolicy resources.
    b. Must support a cron-style schedule around which field validation must be placed (ex., ensuring it uses only a desired subset of the total cron functionality; ensuring it does not run too frequently; etc.)
    c. Must only work in background mode and therefore must ensure background=true for this rule type
    d. Must use existing Kyverno match/exclude and other policy authoring mechanisms to define logic
    e. Must support clean-up of both Namespaced and cluster-scoped resources (including custom)
    f. Must only clean up resources which have neither ttl nor expires annotations assigned
    g. Must show, in some mechanism OTHER THAN the Kyverno log, any and all resources cleaned up by this rule. This should be at least events written to the policy containing the rule responsible for the clean up action.
    h. Must support testing in the Kyverno CLI
    i. Should be capable of checking and reporting, as a warning back to the user at time of policy creation, if Kyverno does not possess the necessary permissions to clean up the resources as defined in the rule's match block.
    j. Should be able to determine unmounted PVs and unreferenced PVCs
    k. Should be able to determine Pods not behind a Service
    l. Should support a dry run functionality so users may determine what resources would be cleaned up at the time the rule executes (Kyverno apply command)
```

In order to solve for the need of users to introspect an existing cluster to determine which resources may be effected by either or both capability, the following CLI use cases must be solved for.

```
CLI Solutions for Annotations Use Case

	• Although most, if not all, of these could be answered with a direct JMESPath query, the CLI could offer a better UX to users but would still internally result in JMESPath queries against the Kubernetes API server.
	• Command may be `kyverno cleanup <inputs>`

	1. "Show me all the resources in the cluster that will be cleaned up in the next 6 hours."
	2. "Show me all the Deployments in the cluster which will be cleaned up by 5pm today."
	3. "Show me all the resources in the foo Namespace which will be cleaned up between now and Friday at 5pm."
	4. "Show me all the resources which were created with a ttl of greater than 7 days."
	5. "Show me all the resources which are set to expire during the month of May of this year."

CLI Solutions for Rule Use Case
	1. "Show me all the resources in the cluster which would be cleaned up when this policy executes."
```

For the rule use case, the following are some pseudo-policy which have been put together to show how some leading use cases might be written as Kyverno policy with a new rule type. Note that, similar to mutate for existing resources, data must be accessible to Kyverno on the existing resources in order to make a policy decision. This proposal recycles the use of the `target` [variable system](https://kyverno.io/docs/writing-policies/mutate/#variables-referencing-target-resources) for that purpose.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: cleanup-examples
spec:
  rules:
  # Cleans up Deployments and Statefulsets without the label `application` at midnight every day.
  - name: cleanup-deploys-no-labels
    match:
      all:
      - resources:
          kinds:
          - Deployment
          - StatefulSet
    exclude:
      all:
      - resources:
          selector:
            matchLabels:
              application: "?*"
    cleanup:
      schedule: "0 0 * * *"
      conditions: {}
  # Cleans up any bare Pods every 5 days
  - name: cleanup-bare-pods
    match:
      any:
      - resources:
          kinds:
          - Pod
    cleanup:
      schedule: "0 0 */5 * *"
      conditions:
        all:
        - key: "{{ target.metadata.ownerReferences[] || `[]` }}"
          operator: Equals
          value: []
  # Clean up all resources in the `foo` Namespace on the first of every month
  - name: cleanup-everything-in-foo
    match:
      any:
      - resources:
          kinds:
          - "*"
          namespaces:
          - foo
    cleanup:
      schedule: "0 * 1 * *"
      conditions: {}
  # Cleans up unbound PVCs every Sunday at 07:00
  - name: cleanup-unbound-pvcs
    match:
      any:
      - resources:
          kinds:
          - PersistentVolumeClaims
    # context:
    # - name: allpvcs
    #   apiCall:
    #     urlPath: "/api/v1/persistentvolumeclaims"
    #     jmesPath: "items[?status.phase == 'Unbound'] || `[]`"
    # preconditions:
    #   all:
    #   - key: "{{allpvcs}}"
    #     operator: NotEquals
    #     value: []
    cleanup:
      schedule: "0 7 * * 0"
      conditions:
        all:
        - key: "{{target.status.phase}}"
          operator: Equals
          value: Unbound
```

These new proposed clean-up abilities would be assisted/augmented by Kyverno's present ability to mutate and validate resources. Some example use cases of Kyverno's present abilities which can be assistative are shown below.

* The desired criteria may be codified into a mutate policy allowing Kyverno to assign the requisite annotation.
    * Ex., write the ttl annotation to all Deployments in the foo Namespace
* A validate policy may be constructed to prevent tampering with the annotation once a resource has been created with it assigned.
    * Ex., block changes to or removals of ttl and/or expires
* A validate policy may be constructed to prevent the use of a cleanup annotation based upon certain criteria.
    * Ex., prevent the use of either annotation for the foo Role
* A validate policy may be constructed preventing both the use of kyverno.io/ttl and kyverno.io/expires from being set on the same resource.
    * Ex., block a resource if it has both ttl and expires set
* A validate policy may be written to enforce certain formatting/standards for these annotations
    * Ex., ttl does not attempt to use seconds
    * Ex., ttl cannot be greater than 90 days
    * Ex., expires uses the correct time format
    * Ex., expires is not a date/time in the past
    * Ex., expires cannot be after the end of this year


# Implementation

A couple implementations have been suggested for the annotation/label methodology. They can roughly be categorized into two separate approaches: Use of a custom resource; use of existing Kubernetes constructs.

## Custom Resources

1. Kyverno could leverage either the existing UpdateRequest (UR) CR (with modifications) in order to support a type of "ledger" to know when to remove resources. The benefit of this approach is that it doesn't require any new CR and CRD to be invented and reduces complexity. The potential downside is the UR may not be suitable for such a job without significant modifications.

2. Kyverno could create a new CR specifically for this clean-up use case. As resources come in with either annotation, the CR can be updated with the clean-up time. Kyverno would manage this CR as it does other CRs. The benefit of this approach is we can craft a CR exactly as needed to fit this purpose. The potential downside is it is more work and more complexity.

## Existing Kubernetes Constructs

Kyverno could alternatively leverage a CronJob resource to perform the deletions. By creating a CronJob and keeping it updated upon create/update/delete of these source resources or annotations/labels, the CronJob can be created with a matching schedule which then deletes them. The CronJob may set an ownerReference to the "parent" resource so that deleting of it causes deletion of the CronJob. (This last statement has not been completely proven.) The benefit of this approach is it reduces technical complexity required to implement an end-to-end solution by leveraging existing resource types. The potential downside is it creates additional resources in a cluster which may be undesirable. It also brings with it complexity as Kyverno's creation of these CronJobs may violate users' validate rules which will either need to be excepted or some other method found to exempt them.

## Link to the Implementation PR

N/A

# Migration (OPTIONAL)

N/A

# Drawbacks

* Makes Kyverno code base more complex

* Places additional strain on the Kubernetes API server and Kyverno itself as some rule processing may need to scavenge the entire cluster. On very large clusters, this could be significant. Care must be taken to not negatively impact cluster operators or existing Kyverno operations.

# Alternatives

* Only implement one of these mechanisms and not both, ex., clean-up of annotated resources only and no new rule; new rule only and no support for clean-up of annotated resources

* Do not provide CLI support initially (strongly discouraged)

* Do not implement this and Kyverno is incapable of solving for these use cases

# Prior Art

* [kube-janitor](https://codeberg.org/hjacobs/kube-janitor)

# Unresolved Questions

1. How should Pods be handled if either the annotation is set directly on them (and not its controller) or if the rule is written which targets Pods? For the rule, should it only clean-up Pods if they do not specify an owner? Or should we prevent this rule from being created in the first place? Should we leave this as docs guidance instead?

2. Kyverno resources (policies, reports, URs, etc.) themselves should either not permit being annotated with the ttl or expires annotation or, alternatively, Kyverno should ignore such annotations on its own resources to prevent them from being cleaned up.

3. Because cleanup rules may naturally require scanning the entire cluster, care should be taken to ensure, for large to huge clusters, this does not overwhelm Kyverno and result in excess processing which may cause failures in other areas. It may be prudent for such cleanup rules, which should run in background only mode, to be processed in chunks and at lower internal priority.

4. If a Kyverno generate rule is written which uses a `data` type and a user specifies it should also write either the ttl or expires annotation, the user must set synchronize: false or else Kyverno must block the rule from being created.

# CRD Changes (OPTIONAL)

This KDP will require introduction of a new rule type suggested as `cleanup`.

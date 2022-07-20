# Meta
[meta]: #meta
- Name: Split-up Policy Report Per Policy
- Start Date:  2022-06-17
- Update date:  2022-06-23
- Author(s): [Prateek Pandey](https://github.com/prateekpandey14), [Vyankatesh Kudtarkar](https://github.com/vyankyGH)
- Supersedes: "N/A"

# Table of Contents
[table-of-contents]: #table-of-contents
- [Meta](#meta)
- [Table of Contents](#table-of-contents)
- [Overview](#overview)
- [Motivation](#motivation)
- [Proposal](#proposal)
- [Implementation](#implementation)
- [Migration (OPTIONAL)](#migration-optional)
- [Drawbacks](#drawbacks)

# Overview
[overview]: #overview

### Current Design of PolicyReport Controller
With the current Implementation, Kyverno generates two types of report: namespace policy report and cluster policy report. Each namespace will have one policy report and there will be a single cluster policy report, which covers all cluster-wide resources.

The create/update of the report can be triggered by:

1) Admission Webhook: whenever there is a change on the resource, or policy, Kyverno enqueues policy application results, the policy report controller would later parse those results and update to policy reports

2) Background Scan: Based on a certain resync interval, kyverno scans all the existing resources in the cluster and updates the policy reports accordingly.

For the Delete operations:

- When a resource is to be deleted, Kyverno generates an empty ReportChangeRequest, with labels that have matched resource information. When the controller sees such request, it removes the matched resource from the results entry.

- When Kyverno starts, the init container will clean up all previous policyReport or clusterPolicyReport. This is required when the resource is deleted while Kyverno is not in running state at that moment.

- When a policy is deleted, the results entry for this policy will also be removed from the policy report.

# Motivation
[motivation]: #motivation

The current implementation works well for small scale environments but when the policy change report crosses the 2k+ entries hence the bigger size data resource in etcd. That may cause high CPU usage and `resourceExhausted` failures while adding/updating large reports and scan may not be completed in a given time frame.

This is not scalable for large clusters or the clusters have a relatively large number of policies configured. For example, one can hit this issue if the cluster has 20 policies (1 rule per policy) with 100 matching resources in a namespace (20 x 100 = 2k).

To overcome this problem we need to split  larger reports per namespace entries which have been explained in below mentioned proposals.

# Proposal

Proposed Design for PolicyReport Controller is to split the PolicyReport per policy per namespace (with feature control flag).
Kyverno will generate  policy reports for every policy per namespace instead of creating a single policy report for all policies deployed in a cluster.
As we have created a report per policy there is very less chance of more application report entries which causes the high memory consumption. 

On each reconciliation or kyverno pod restart with multiple goroutine of policy report controller will update the policy report with less amount of entries to aggregate.


# Implementation

- With new changes namespace scope resource `PolicyReport` name will be updated to include the policy name as a suffix, for example
if we apply the below policy `disallow-privileged-containers` the generated `PolicyReport` name is `polr-ns-default-disallow-privileged-containers`

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged-containers
  annotations:
    policies.kyverno.io/title: Disallow Privileged Containers
    policies.kyverno.io/category: Pod Security Standards (Baseline)
    policies.kyverno.io/severity: medium
    policies.kyverno.io/subject: Pod
    kyverno.io/kyverno-version: 1.6.0
    kyverno.io/kubernetes-version: "1.22-1.23"
    policies.kyverno.io/description: >-
      Privileged mode disables most security mechanisms and must not be allowed. This policy
      ensures Pods do not call for privileged mode.
spec:
  validationFailureAction: audit
  background: true
  rules:
    - name: privileged-containers
      match:
        any:
        - resources:
            kinds:
              - Pod
      validate:
        message: >-
          Privileged mode is disallowed. The fields spec.containers[*].securityContext.privileged
          and spec.initContainers[*].securityContext.privileged must be unset or set to `false`.
        pattern:
          spec:
            =(ephemeralContainers):
              - =(securityContext):
                  =(privileged): "false"
            =(initContainers):
              - =(securityContext):
                  =(privileged): "false"
            containers:
              - =(securityContext):
                  =(privileged): "false"
```

```sh
kubectl get polr -n default
NAMESPACE   NAME                                             PASS   FAIL   WARN   ERROR   SKIP   AGE
default     polr-ns-default-disallow-privileged-containers   18     15     0      0       0      114m
```

- With new changes cluster scope resource `ClusterPolicyReport` name will be updated to include the policy name as a suffix, for example
if we apply the below policy `require-ns-dev-labels` the generated `ClusterPolicyReport` name is `clusterpolicyreport-require-ns-dev-labels`.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-ns-dev-labels
spec:
  validationFailureAction: audit
  background: true
  rules:
  - name: check-for-labels-on-namespace
    match:
      any:
      - resources:
          kinds:
          - Namespace
    validate:
      message: "The label `thisshouldnotexist` is required."
      pattern:
        metadata:
          labels:
            thisshouldnotexist: "?*"
```

```sh
kubectl get cpolr
NAME                                        PASS   FAIL   WARN   ERROR   SKIP   AGE
clusterpolicyreport-require-ns-dev-labels   0      2      0      0       0      18s  // generated for missing dev labels
clusterpolicyreport-require-ns-prod-labels  0      2      0      0       0      18s  // generated for missing prod labels

```

## Link to the Implementation PR: 

- https://github.com/kyverno/kyverno/pull/4147

# Migration (OPTIONAL)

To enable this feature, user needs to enable the `splitPolicyReport: true` feature gate in kyverno container args, which is disable by default. Once enabled kyverno will delete the existing old reports and create the new reports on per policy bases in the background.

# Questions

1. User updates a ClusterPolicy/Policy to add a rule which previously didn't exist.
-  PolicyReport/ClusterPolicyReport gets updated with new rule validatation


2. User removes a rule from an existing ClusterPolicy/Policy.
-  PolicyReport/ClusterPolicyReport gets updated with existing rule validatations

3. User renames a ClusterPolicy/Policy.
-  New PolicyReport/ClusterPolicyReport gets created with new name, for ClusterPolicy generated name is `clusterreport-<policy-name>` and for namespace policy `polr-ns-<policy-name>`

4. How this will work with the new auto-gen capabilities (currently feature flagged) and if there are any modifications needed.
-  TODO

# Drawbacks

There will be multiple policy reports generated for each policy per namespace as compared to only one per namespace or cluster report. If single policy has large number of rule define kyverno may face similar memory issue.

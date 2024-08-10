# Kyverno Design Proposal - Add mutation support in the CLI

Created: Aug 10th, 2024
Author: [Ammar Yasser](https://github.com/aerosouund)

- [Overview](#overview)
  * [Online Operation](#online-operation)
  * [Offline Operation](#offline-operation)
- [Proposal](#proposal)
- [Example](#example)
- [Implementation](#implementation)
  * [Resources obtained from a cluster](#resources-obtained-from-a-cluster)
  * [Resources passed from the CLI directly](#resources-passed-from-the-cli-directly)
  * [Previous attempts](#previous-attempts)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

## Overview

Kyverno webhook controller runs as an admission controller in Kubernetes, and it mutates resources upon admission review. Such mutation is performed based on resources operations (CREATE/UPDATE/DELETE) via webhook.
This feature currently is included in the webhook controller only, the CLI has the current behavior:

### Online Operation
an operation performed on a live kubernetes cluster and has the `--cluster` flag passed

**Behavior**: No mutation occurs on the target resources fetched from the cluster.
The output is just how many resources passed or failed as typical in the output of the cli.

`pass: 5, fail: 0, warn: 0, error: 0, skip: 0`

**Technical Explanation**: The CLI succeeds in fetching the correct resources and applies the mutation properly. but what gets returned to the report generator is still the original resource

`pkg/engine/mutation.go`:

```go
resource, ruleResp := e.invokeRuleHandler(
	ctx,
	logger,
	handlerFactory,
	policyContext,
	matchedResource,
	rule,
	engineapi.Mutation,
)

matchedResource = resource
```

### Offline Operation

an operation performed on static resources passed through the `--resource` flag or from a directory

**Behavior**: The CLI outputs this message

```
Handler factory requires a client but a nil client was passed, likely due to a bug or unsupported operation.
```

**Technical Explanation**:

The introduction of this error message was required as a guardrail for when the mutation engine tries to fetch resources from an existing client (the object that interfaces with a kubernetes cluster for CRUD operations) that is uninitialized due to not being in cluster mode, the CLI would then panic, see [this issue](https://github.com/kyverno/kyverno/issues/6723) for additional context
	

## Proposal

The CLI isn't going to mutate actual resources, its only required to output the changes that will be made by a mutation rather than the original resource in case of a mutation

## Example

This example adds the label `foo=bar` to deployments in the `secure` namespace 

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: policy
spec:
  mutateExistingOnPolicyUpdate: true
  rules:
    - name: deployment-update
      match:
        any: 
        - resources:
            kinds:
            - Deployment
            namespaces:
              - "secure"
      mutate:
        targets:
        - apiVersion: apps/v1
          kind: Deployment
          namespace: "secure"
        patchStrategicMerge:
          metadata:
            annotations:
              foo: bar
```

Invocation:
`kyverno apply ~/policies/pol.yaml --cluster -n secure` 

Should output:

```
Applying 1 policy rule(s) to 1 resource(s)...

mutate policy policy applied to secure/Deployment/nginx-deployment:
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
	foo: bar <<< SHOULD REFLECT THE MODIFIED RESOURCE
    deployment.kubernetes.io/revision: "1"
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"name":"nginx-deployment","namespace":"secure"},"spec":{"replicas":3,"selector":{"matchLabels":{"app":"nginx"}},"template":{"metadata":{"labels":{"app":"nginx"}},"spec":{"containers":[{"image":"nginx:1.21.1","name":"nginx","ports":[{"containerPort":80}]}]}}}}
  creationTimestamp: "2024-07-29T13:18:13Z"
  generation: 2
  // omitted fields
spec:
  progressDeadlineSeconds: 600
  replicas: 0
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:
      maxSurge: 25%!
(MISSING)      maxUnavailable: 25%!
(MISSING)    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.21.1
        imagePullPolicy: IfNotPresent
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  conditions:
  - lastTransitionTime: "2024-07-29T13:18:13Z"
    lastUpdateTime: "2024-07-29T13:18:19Z"
    message: ReplicaSet "nginx-deployment-5b4b69594c" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  - lastTransitionTime: "2024-07-29T18:25:49Z"
    lastUpdateTime: "2024-07-29T18:25:49Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  observedGeneration: 2

---

pass: 1, fail: 0, warn: 0, error: 0, skip: 0 
```


## Implementation

There are two scenarios that need to be supported in the implementation:

### Resources obtained from a cluster

The `policyProcessor` of these resources will have a non nil client
`cmd/cli/kubectl-kyverno/commands/apply/command.go`:
```go
processor := processor.PolicyProcessor{
	Store: store,
	Policies: validPolicies,
	Resource: *resource,
	PolicyExceptions: exceptions,
	MutateLogPath: c.MutateLogPath,
	MutateLogPathIsDir: mutateLogPathIsDir,
	Variables: vars,
	UserInfo: userInfo,
	PolicyReport: c.PolicyReport,
	NamespaceSelectorMap: vars.NamespaceSelectors(),
	Stdin: c.Stdin,
	Rc: &rc,
	PrintPatchResource: true,
	Client: dClient, // THIS WONT BE NIL
	AuditWarn: c.AuditWarn,
	Subresources: vars.Subresources(),
	Out: out,
}
```
We need to return the actual mutated resource and modify the report generator to print it by changing the `matchedResource` variable to contain the mutated resource, which can be obtained from the rule response. `ruleResp.patchedTarget`	

### Resources passed from the CLI directly

The mutation engine should realize that when the client is nil, and create an appropriate handler that doesn't fetch resources from the cluster but applies the mutation

`pkg/engine/mutation.go`:

```go
handlerFactory := func() (handlers.Handler, error) {
if !rule.HasMutate() {
	return  nil, nil
}

if !policyContext.AdmissionOperation() && rule.HasMutateExisting() {
	if e.client == nil {
		return  nil, fmt.Errorf("Handler factory requires a client but a nil client was passed, likely due to a bug or unsupported operation.")
	}
	return mutation.NewMutateExistingHandler(e.client)
}
	return mutation.NewMutateResourceHandler()
}
```

A `MutateResourceHandler` should be returned in this case
In case also simulation of a client update is required for non cluster operation then a kubernetes Fake client can be used to test if the mutation will fail using `k8s.io/client-go/dynamic/fake`
In this case the client initialization must account for this scenario

```go
func (c *ApplyCommandConfig) initStoreAndClusterClient(store *store.Store, skipInvalidPolicies SkippedInvalidPolicies) (*processor.ResultCounts, []*unstructured.Unstructured, SkippedInvalidPolicies, []engineapi.EngineResponse, dclient.Interface, error) {

	// other logic

	var  err  error
	var  dClient dclient.Interface

	if !c.Cluster {
		dynamicClient := fake.NewSimpleDynamicClient(s) // s is a kubernetes scheme.Scheme
		dClient, err = dclient.NewClient(context.Background(), dynamicClient, kubeClient, 15*time.Minute)

		if err != nil {
			return  nil, nil, skipInvalidPolicies, nil, nil, err
		}
		return  nil, nil, skipInvalidPolicies, nil, dClient, err
	}
	// otherwise initialize a normal client
}
```
And this fake client will need to be passed the resources passed in the cli and created in its store to later be patched when the mutation engine returns the result


### Previous attempts

A previous [PR](https://github.com/kyverno/kyverno/pull/5220) tried to tackle this project before, the overall solution was to include a field in the `policyContext` that conveys if the command is a test command or something else. In case it is, the `loadTargets` method reads the resource from the context, where it exists as a field called `oldResource` and appends it to the resource list.

While this is still a viable approach it needs to be extended to be command agnostic, to be valid for `apply` and `test` as is the stated goal of the project. the overall idea that can be drawn from this is that the check for the nil client can be performed at the level of loading targets, this can be an alternative approach to what is proposed above
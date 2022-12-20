# Meta
[meta]: #meta
- Name: Service API Call
- Start Date: December 20, 2022
- Update data (optional):  December 20, 2022
- Author(s): @JimBugwadia


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

Support HTTP/S GET and POST calls to JSON web services.

# Definitions
[definitions]: #definitions


# Motivation
[motivation]: #motivation

Kyverno versions 1.9 and earlier support API calls to the Kubernetes API server and loading data from ConfigMap resources. 

However, policies often require data from other services e.g. a metrics service. Currently, this requires writing custom code to periodically fetch data from the services and store the results in a ConfigMap or other Kubernetes resource that can be retrieved by Kyverno. This feature allows the service to be directly queried, eliminating the need to write custom services and create Kubernetes custom resources for policy data.

In other cases, some policies may require delegation of processing to other controllers. For example, a separate controller may periodically fetch and cache external data which is used to process policies. With this feature, Kyverno can send the service a POST request with context data, and the service can respond back with results that are used to make the policy decision. 

Finally, in some scenarios 3rd party library code or binaries need to be executed to make a policy decision. Currently, the only way to support these "extensions" is to directly update the Kyverno core. With this feature, it will be possible to write Kyverno extensions as separate services hence decoupling the Kyverno code from 3rd party libraries or binaries.

# Proposal

A minimal (Phase 1) implementaion of this feature will enable:
1. HTTP GET and POST API calls to any service
2. Encryption and server authentication using a certificate bundle
3. Client authentication using a bearer token, configured via the Kubernetes service account token volume projection feature, in the HTTP Authorization header. This allows the server to use the [Token Review API](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-review-v1/) to authenticate requests from Kyverno.
4. Passing context data to the external service
5. Applying JMESPath transformation to the results
6. Storing service API call results in the rule context for further processing

Future enhancements can include:
1. Support for additional authentication methods (mTLS, Basic auth, etc.)
2. Configuring HTTP headers
3. Simplifying passing data as URL query parameters
4. Configuring connection retries, timeouts, etc.
5. Additional connection config e.g. insecureSkipSSLVerify 

# Implementation

Here are the changes involved:
1. Add a new `service` to the existing `APICall` declaration, to allow specifying information like the URL and CA bundle. The existing `urlPath` and `jmesPath` fields will stay.
2. Update the logic to handle APICall to check whether a `urlPath` is configured to invoke the API server. Otherwise, process the `service` declaration to execute an HTTP GET or POST to the JSON web service.
3. Apply the JMESPath (if specified) to the results and store in context, as before. 

## CRD Changes

Changes for the APICall object:

```sh
 ❯ kubectl explain cpol.spec.rules.context.apiCall

KIND:     ClusterPolicy
VERSION:  kyverno.io/v1

RESOURCE: apiCall <Object>

DESCRIPTION:
     APICall is an HTTP request to the Kubernetes API server, or other JSON web
     service. The data returned is stored in the context with the name for the
     context entry.

FIELDS:
   jmesPath     <string>
     JMESPath is an optional JSON Match Expression that can be used to transform
     the JSON response returned from the server. For example a JMESPath of
     "items | length(@)" applied to the API server response for the URLPath
     "/apis/apps/v1/deployments" will return the total count of deployments
     across all namespaces.

   service      <Object>
     Service is an API call to a JSON web service

   urlPath      <string>
     URLPath is the URL path to be used in the HTTP GET request to the
     Kubernetes API server (e.g. "/api/v1/namespaces" or
     "/apis/apps/v1/deployments"). The format required is the same format used
     by the `kubectl get --raw` command.

```

New ServiceCall object:

```sh
❯ kubectl explain cpol.spec.rules.context.apiCall.service

KIND:     ClusterPolicy
VERSION:  kyverno.io/v1

RESOURCE: service <Object>

DESCRIPTION:
     Service is an API call to a JSON web service

FIELDS:
   caBundle     <string>
     CABundle is a PEM encoded CA bundle which will be used to validate the
     server certificate.

   data <[]Object>
     Data specifies the POST data sent to the server.

   method <string> -required-
     method is the HTTP request method (GET or POST).

   urlPath      <string> -required-
     URL is the JSON web service URL. The typical format is
     `https://{service}.{namespace}:{port}/{path}`.

```


## Sample Policy Fragments

An HTTP GET API call to a service passing in the `serviceAccountName` and storing the result as `allowed`.

```yaml
    context:
    - name: allowed
        apiCall:
        service:
            url: https://service.namespace:443/check?sa={{serviceAccountName}}
```

An HTTP POST API call to a service that:
- passes in a map with one entry `images` and value containing a list of images e.g. {"images":["https://ghcr.io/tomcat/tomcat:9","https://ghcr.io/vault/vault:v3","https://ghcr.io/busybox/busybox:latest"]} 
- uses a X.509 certificate stored in the context variable `cert`
- applies a JMESPath transformation to check for failed items and stores results in in the context variable `failedCount`

```yaml
    context:
    - name: failedCount
      apiCall:
        service:
          url: https://service.namespace:443/validateImages
          caBundle: {{ cert }}
          requestType: POST
          data:
          - key: images
            value: "{{ images.[containers, initContainers, ephemeralContainers][].*.reference[] }}"
        jmesPath: "items[?status == 'failed'] | length(@)"
```

## Link to the Implementation PR

https://github.com/kyverno/kyverno/pull/5755

# Migration (OPTIONAL)

N/A

# Drawbacks

1. **Security**: Calling another service raises security concerns. These can be mitigated by ensuring network policies are configured and only in-cluster service calls are performed.

2. **Performance**: Service calls will add to processing time resulting in timeouts. Again, restricting usage to in-cluster services with low network latency is recommeded. Services can also be optimized for fast responses using local caches. 

# Alternatives

N/A

# Prior Art

OPA Gatekeeper supports [external data providers](https://open-policy-agent.github.io/gatekeeper/website/docs/externaldata/).

# Unresolved Questions

1. How should we handle CLI support?


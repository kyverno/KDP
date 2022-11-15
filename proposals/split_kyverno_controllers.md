# Meta
[meta]: #meta
- Name: Split / Decompose Kyverno Controllers
- Start Date: November 9th, 2022
- Update data (optional): November 9th, 2022
- Author(s): @JimBugwadia, @eddycharly

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

Kyverno 1.8 runs as a single Deployment. This includes the webhook, reporting, and background controllers for generate and update requests. This design causes heavy backhround processing to impact the webhook performance. This proposal separates the webhook into its own Deployment.

# Definitions
[definitions]: #definitions

Make a list of the definitions that may be useful for those reviewing. Include phrases and words that Kyverno users or other interested parties may not be familiar with.

# Motivation
[motivation]: #motivation

Here are some reasons why we are considering splitting a Kyverno installation into multiple components:
- Eliminate (or greatly reduce) client-side throttling during processing of admission requests in the webhook
- Allow independent sizing and scaling of components. For example, some components operate active-active while others operate using a leader-election. Hence, separating these makes sense.
- Enhance security by limiting permissions for each component. This may not be a main driver, but is inteesting to consider.

# Minimal requirements for a release

The main goal for an initial implementation is to separate all background processing from the webhook processing, to eliminate throttling and allow scaling the webhook to 

# Proposal

## Kyverno Components

Here is a list of the current top-level Kyverno components:

| Controller  | Function      |
| ------------| ------------- |
| Webhook     | Processes AdmissionReview requests from the API Server |
| Reports Controller | Processes AdmissionReport, ClusterAdmissionReport , BackgroundScanReport, and ClusterBackgroundScanReport |
| Update Controller | Processes UpdateRequests for mutate and generate policies |
| Policy Controller | Processes policy changes |
| Cleanup Controller | Processes cleanup policy related admission requests |

**Note**: other components like the webhook monitor, cetificate manager, policy cache, are not mentioned here as they are they are helpers to a top-level component.


### Deciding when to split

The reasons to split components match the items in the `Motivation` section.

Compoments that have different inputs / outputs, different performance profiles, and require independent sizing should be split.

Components that have a similar lifecycle and behaviors can remain together, to keep Kyverno easy to install and manage.

Its important to note, that each new binary or Deployment imposes some management overhead and complexity. Hence, the goal is to find the right balance of deployment artifacts.

## Proposed split

The proposal is to create three processes / deployments for Kyverno:
1. Webhook
2. Reports
3. Updates 

The policy controller can stay with the webhook, as it has a similar lifecycle. 

The cleanup controller seems fairly independent from an implementation perspective, but its not clear if splitting it provides any substantial user benefits. This requires more thought and discussion.

The mutate and generate modules can also stay together.

## Code Repository Structure

All Kyverno code can stay in the same repository, and the top-level packages can reflect the logical design:

* **webhook**
* **controllers/reports**
* **controllers/updates**
* **controllers/cleanup**
* **controllers/exceptions**

## User Experience

This section describes the user experience across installation and management of Kyverno:

### Installation

Kyverno can still be deployed as a single Helm chart or via a single YAML file. 

### Runtime 

When Kyverno is installed there would be two Deployment resources created. The Helm chart can allow each Deployment to be scaled independently. 

### Configuration

The Kyverno ConfigMap can allow enabling or disabling all major features or modules. The same configuration map can be used by all processes.


# Implementation

TBD.

## Link to the Implementation PR

# Migration (OPTIONAL)

When a new version of Kyverno, with this feature, is installed it will deploy two controllers. The `kyverno` deployment can continue to be the webhook, and a new `kyverno-background-controller` can execute to process the reports and update requests. 

# Drawbacks

This scheme adds some more complexity to the overall Kyverno installation.

# Alternatives

### Granular controllers

As mentioned above, it is possible to further decompose the background controller.

### Multiple Helm Charts

A more granular approach would also allow separate Helm charts to be installed. 

# Prior Art

## ArgoCD

When ArgoCD is installed it runs three different controllers.

## kube-prometheus

When installing kube-prometheus-stack you bring prometheus operator, grafana, node-exporter, kube-state-metrics, and a bunch of resources by installing a single chart.


# Unresolved Questions

TBD

# CRD Changes (OPTIONAL)

There are no CRD changes envisioned.
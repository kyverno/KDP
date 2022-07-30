# Meta
[meta]: #meta
- Name: Clean-up
- Start Date: 2022-07-30
- Author(s): chipzoller
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
- [Migration (OPTIONAL)](#migration-optional)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Prior Art](#prior-art)
- [Unresolved Questions](#unresolved-questions)
- [CRD Changes (OPTIONAL)](#crd-changes-optional)

# Overview
[overview]: #overview

A clean-up ability of Kyverno allowing it to clean-up (delete) resources based upon certain criteria.

# Definitions
[definitions]: #definitions

- TTL: Time-to-live. The amount of time a resource may exist until it is cleaned up.

# Motivation
[motivation]: #motivation

The initial idea for this feature came from issue [3483](https://github.com/kyverno/kyverno/issues/3483) authored by user luisdavim.

As long as there has been Kubernetes, there has been the inevitability that some resources--most especially those created directly by human users--are left derelict at some point without a good way to identify what and where they are. As Kubernetes continues to expand with its myriad of possible custom resources, this problem grows over time. The ultimate problems this poses are few: 1) Kubernetes' key-value store, which has fairly limited storage capacity, may become overwhelmed by resources which should be deleted; and 2) these resources, in the case of Pods, may result in consumption of excess resources driving up cost and reducing workload consolidation abilities. The solution to these problems is to identify the derelict resources and remove (i.e., delete) them.

# Proposal

In this proposal, Kyverno may function as a clean-up controller.


This provides a high level overview of the feature.

- Define any new terminology.
- Explaining the feature largely in terms of examples.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing users and new users.

# Implementation

This is the technical portion of the KDP, where you explain the design in sufficient detail.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

## Link to the Implementation PR

# Migration (OPTIONAL)

This section should document breaks to public API and breaks in compatibility due to this KDP's proposed changes. In addition, it should document the proposed steps that one would need to take to work through these changes.

# Drawbacks

Why should we **not** do this?

# Alternatives

- What other designs have been considered?
- Why is this proposal the best?
- What is the impact of not doing this?

# Prior Art

Discuss prior art, both the good and bad.

# Unresolved Questions

- What parts of the design do you expect to be resolved before this gets merged?
- What parts of the design do you expect to be resolved through implementation of the feature?
- What related issues do you consider out of scope for this KDP that could be addressed in the future independently of the solution that comes out of this KDP?

# CRD Changes (OPTIONAL)

Does this KDP entail any proposed changes to the core CRD or any extensions? If so, please document changes here.
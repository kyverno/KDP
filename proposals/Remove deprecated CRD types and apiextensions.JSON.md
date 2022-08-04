# Kyverno Design Proposal - Remove deprecated CRD types and apiextensions.JSON

- Name: Remove deprecated CRD types and apiextensions.JSON
- Start Date:  2022-08-04
- Author(s): [Vyankatesh Kudtarkar](https://github.com/vyankyGH)

# Table of Contents
- [Table of Contents](#table-of-contents)
- [Overview](#overview)
- [Definitions](#definitions)
- [Motivation](#motivation)
- [Proposal](#proposal)
- [Implementation](#implementation)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Prior Art](#prior-art)
- [Unresolved Questions](#unresolved-questions)
- [CRD Changes (OPTIONAL)](#crd-changes-optional)


# Overview
[overview]: #overview

This proposal is to remove deprecated CRD types and apiextensions.JSON from schema. Several CRD types have been marked as deprecated and need to be removed. We need to publish new schema version (v2) with these changes including backward compatibility.


## Definitions

[definitions]: #definitions

None

# Motivation
[motivation]: #motivation

apiextensions.JSON breaks structural schema validation and also does not allow kubectl explain and other tools to show the API syntax and help.

# Proposal

Proposed Design for remove all apiextensions.JSON types and all deprecated types. Publish new schema (v2) with support backward compatibility.


# Implementation
1. Kyverno needs to replace  apiextensions.JSON to maintain compatibility with the older syntax.
2. Kyverno needs to upgrade to v2 Schema. Also need to clean beta objects from v1 schema .
3. For backwards compatibility, I have to make sure that the old policies at least work for at least three versions.



## Link to the Implementation PR: 

TBD


## Drawbacks

Image verify rule where we're using apiextensions.JSON but we cannot remove that because kyverno using a nested structure and open API v3 does not allow method structures.


## Alternatives

N/A

## Prior Art

N/A

## Unresolved Questions

N/A

## CRD Changes (OPTIONAL)

This proposal should support backward compatibility for atleast 3 release.

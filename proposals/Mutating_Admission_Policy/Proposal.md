# Support MutatingAdmissionPolicies in Kyverno CLI

*Author:* Kushal Agrawal
*Date:* 08/03/2025
*Status:* Proposed


## Summary

This proposal aims to extend the Kyverno CLI to support Kubernetes MutatingAdmissionPolicies in the `apply` and `test` commands. With Kubernetes 1.32 introducing MutatingAdmissionPolicies, it is important for Kyverno to support these policies in the CLI—mirroring the support already available for ValidatingAdmissionPolicies.



## Motivation

- **New Kubernetes Feature:** Kubernetes 1.32 introduces MutatingAdmissionPolicies, which allow automatic mutation of resources during creation or update.
- **Consistency:** Kyverno already supports ValidatingAdmissionPolicies. Adding support for mutating policies ensures consistency in policy handling.
- **Enhanced Testing & Debugging:** Users will be able to test and validate their mutating policies locally via the CLI before applying them in a cluster, reducing errors and improving developer experience.
- **Future-proofing:** As Kubernetes evolves, having comprehensive support for both validating and mutating policies in the CLI will keep Kyverno competitive and up to date.


## Design Details

### Current State

- **ValidatingAdmissionPolicies:** Kyverno CLI currently applies and tests ValidatingAdmissionPolicies. The implementation is well-integrated into the CLI commands.
- **Missing Mutating Support:** There is no support yet for MutatingAdmissionPolicies, so any resource mutations defined by these policies are not processed by the CLI.

### Proposed Changes

1. **Policy Detection:**
   - Extend the policy loader to differentiate between validating and mutating policies.
   - Identify policies based on their type or annotations.

2. **Mutation Logic Integration:**
   - Implement mutation logic in the CLI similar to the existing validation flow.
   - When running `apply` or `test`, the CLI should:
     - Load the provided MutatingAdmissionPolicies.
     - Detect if the target resource qualifies for mutation.
     - Apply the mutation and output the mutated resource.

3. **Error Handling:**
   - Provide clear error messages if the mutation fails (e.g., due to malformed policies or incompatible resources).
   - Ensure that failing to mutate does not cause unexpected CLI behavior.

4. **User Interface:**
   - Optionally, include a CLI flag (e.g., `--mutate`) to explicitly enable/disable mutation processing.
   - Update CLI help messages to reflect the new functionality.

### Workflow Diagram

A simplified flow of execution could look like:


1. **Phase 1: Research & Planning**
   - Study Kubernetes documentation for MutatingAdmissionPolicies.
   - Analyze the current implementation for ValidatingAdmissionPolicies in the Kyverno CLI.

2. **Phase 2: Development**
   - Extend the policy loader to recognize MutatingAdmissionPolicies.
   - Implement mutation logic in CLI commands (`apply` and `test`).
   - Add CLI flags and update user messages.

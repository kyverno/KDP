# Introduce eBPF

- **Authors**: Eileen Yu (@Eileen-Yu), Jim Bugwadia (jim@nirmata.com)
- **Created**: Feb 15th, 2023
- **Updated**: Feb 20th, 2023
- **Abstract**: Introduce eBPF to trigger Kyverno policy for extended use cases

## Contents

- [Introduction](#introduction)
- [The Problem](#the-problem)
- [Proposed Solution](#proposed-solution)
- [Open Question](#open-question)

## Introduction

Kyverno is now running as a dynamic admission controller for policy management. This allows users to apply policies to Kubernetes resources during admission controls and as background scans.

This proposal aims to extend the ability of Kyverno, so that it can apply policies based on runtime events detected via eBPF events.

## The Problem

Besides configuration changes and periodic scans, security conscious users may want to trigger the application of policies based on runtime events . This goes far beyond simply "approve / deny" the request. One instance is, users may want to detect when a volume is mounted in a running container, via an operation that bypasses the API server, and shutdown a pod . Another example is to report a violation for certain network calls .

It would be best if there is a way to enforce these security requirements as Kyverno policies.

## Proposed Solution

To instrument on the system call interface, some reliable eBPF observability tools (e.g Tetragon, Falco) can help to monitor which sys-call is invoked and generate logs / events / metrics.

Once Kyverno gets the data through informer / websocket, it would check if a certain existing policy should be triggered. If so, Kyverno would apply rules to modify or delete the manifest of corresponding resource, and generate rule violation reports.

## Open Question:

**Q1: What type of logs / events / metrics that Kyverno would accept? How to parse them?**

- Thought: There are various tools that can monitor kernel activities, and they may generate reports in various forms. For example, tetragon would provide logs, while Falco can have multiple choice to send alerts. It seems hard for Kyverno to implement logic to convert these reports to a unified format like the Kubernetes event format Kyverno can then process these events and trigger policies.

This would also grant users the flexibility to choose what kind of eBPF tool they would like to use.

**Q2: How to define the policy?**

- Thought: We may adding specs to current policy and bump the version to `alpha`.

**Q3: How would the spec look like?**

- Thought: We may start from a simple prototype focus on certain use cases.

  For example, to limit the volume mount to a certain container:

  ```yaml
  ---
  spec:
    rules:
      - name: restrict-volume-mount-to-container-A
        match:
          any:
            - resources:
              kinds:
                - Pod
        Restirct:
          message: Restrict volume mount to container A
          pattern:
            spec:
              containers:
                - name: A
              behavior:
                - volume-mount
  ```

  The `Restirct` spec means to block the target behavior.
  We may also have other key words for other use cases.

  `volume-mount` is a standard input set by Kyverno. In this way the incoming event can be matched with existing policy.

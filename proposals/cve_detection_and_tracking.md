
# KDP: Automated CVE Detection, Tracking, and Remediation

* **Status**: Proposed
* **Release**: N/A
* **Authors**: https://github.com/eddycharly

## Summary

This proposal defines an automated framework for detecting, tracking, and remediating vulnerabilities (CVEs) within the Kyverno codebase. By integrating **Trivy** with **GitHub Code Scanning** and **GitHub Issues**, we establish a continuous security loop that monitors the `main` branch and active release branches.

## Motivation

As a security-focused project, Kyverno must maintain a transparent and rigorous vulnerability management process. We need a system that:

1. **Shifts Security Left**: Detects vulnerabilities during the development phase (on push).
2. **Maintains Hygiene**: Monitors older release branches currently in support.
3. **Automates Triage**: Reduces manual overhead by syncing security alerts directly into the developer workflow (GitHub Issues).
4. **Visibility**: Provides a centralized, queryable dashboard for maintainers to assess project health.

## Requirements

* **Scanning Tool**: [Trivy](https://github.com/aquasecurity/trivy) for filesystem and dependency analysis.
* **Target Scope**: The `main` branch and the two most recent minor release branches.
* **Scan Frequency**:
* On every `push` to the targeted branches.
* Daily (cron) to catch new vulnerabilities in static code.


* **Reporting**: Integration with GitHub Code Scanning via **SARIF** uploads.
* **Lifecycle Management**: Bidirectional-like synchronization between Code Scanning alerts and GitHub Issues.

## Recommended Design

### 1. Detection Infrastructure

The system uses three specialized GitHub Action workflows to separate concerns:

* **Continuous Integration (`trivy.yaml`)**: Triggered on every push to specific branches. It performs a rapid scan of the repository filesystem and generates a SARIF report.
* **Periodic Surveillance (`trivy-periodic-scan.yaml`)**: A scheduled daily job that re-scans the `main` and release branches. This ensures that a dependency declared "safe" yesterday is re-evaluated against today's updated vulnerability databases.
* **Alert Synchronization (`trivy-sync-issues.yaml`)**: A dedicated workflow that polls GitHub Code Scanning alerts and manages the lifecycle of corresponding GitHub Issues.

### 2. Workflow Logic & SARIF Integration

The workflows are configured to output results in the SARIF format, which is then uploaded using the `github/codeql-action/upload-sarif` action. This populates the **Security > Code scanning alerts** tab in the repository.

### 3. Maintainer Visibility (Dashboarding)

Visibility is achieved through pre-filtered GitHub Security views. Maintainers can monitor the current state of vulnerabilities across all supported versions using a single URL:

> `https://github.com/kyverno/kyverno/security/code-scanning?query=is%3Aopen+branch%3Amain%2Crelease-1.17%2Crelease-1.16+++tool%3ATrivy+`

### 4. Issue Tracking Automation

To ensure CVEs are assigned and fixed:

* **Creation**: When Trivy detects a new vulnerability, the sync workflow opens a GitHub Issue labeled `security` and `cve`.
* **Resolution**: When a patch is merged and a subsequent Trivy scan confirms the CVE is gone, the sync workflow automatically closes the related Issue.

## Alternatives Considered

* **Dependabot Alerts Only**: Rejected because Dependabot primarily tracks manifest files and may miss vulnerabilities in the underlying OS packages or complex build-time dependencies that Trivy identifies.
* **Manual Tracking in Spreadsheets**: Rejected due to lack of scalability and poor integration with the developer PR workflow.

## Drawbacks

* **Noise**: High-frequency scanning can lead to "alert fatigue" if low-severity CVEs are not properly filtered.
* **Branch Maintenance**: The `trivy-periodic-scan.yaml` and the security dashboards require manual updates whenever a new release branch (e.g., `release-1.18`) is created to ensure the "two most recent releases" policy is maintained.

## Testing Plan

1. **Detection Test**: Introduce a known vulnerable dependency into a test branch and verify that `trivy.yaml` correctly identifies it and uploads the SARIF.
2. **Sync Test**: Verify that the `trivy-sync-issues.yaml` workflow successfully creates a labeled issue based on the test alert.
3. **Resolution Test**: Update the dependency to a fixed version and verify that both the Code Scanning alert and the GitHub Issue are closed automatically.

## Implementation Details

The following workflows already implement this design:

* [trivy.yaml](https://github.com/kyverno/kyverno/blob/main/.github/workflows/trivy.yaml)
* [trivy-periodic-scan.yaml](https://github.com/kyverno/kyverno/blob/main/.github/workflows/trivy-periodic-scan.yaml)
* [trivy-sync-issues.yaml](https://github.com/kyverno/kyverno/blob/main/.github/workflows/trivy-sync-issues.yaml)

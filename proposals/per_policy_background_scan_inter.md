# Meta
- Name: Per-Policy Background Scan Interval
- Start Date: 2026-05-06
- Author(s): Varsha U N (GitHub: @VarshaUN)
- Supersedes: N/A

# Table of Contents
- [Meta](#meta)
- [Table of Contents](#table-of-contents)
- [Overview](#overview)
- [Motivation](#motivation)
- [Proposal](#proposal)

# Overview
Allow users to set a custom `backgroundScanInterval` at the individual Policy / ClusterPolicy level, instead of relying only on the global `backgroundScanInterval` setting (default 1h).

# Motivation
Right now, all policies with background scanning enabled run on the **same global schedule**. This causes problems in real-world usage:

- Critical security or compliance policies need more frequent scanning (e.g. every 15m).
- Non-critical or heavy policies cause unnecessary load if scanned too often.
- Many policies starting at the exact same time create CPU/memory spikes in the Kyverno background controller (especially noticeable with PSA policies and cleanup controllers).

Users have repeatedly asked for this in Slack and in issue #5676.

**Expected outcome**: More flexibility, better resource usage, and the ability to spread background scan load across time.

# Proposal

Add a new optional field in the Policy/ClusterPolicy spec:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-pod-labels
spec:
  background: true
  backgroundScanInterval: 15m     # <-- new field
  rules:
    - name: check-labels
      ...
```

Format should support the same duration strings as the global setting (15m, 1h, 6h, 24h, etc.).
If not specified, it falls back to the global backgroundScanInterval.This setting only applies when background: true.

Example Use Cases

Security policy → 15m
Best-practice / hygiene policy → 4h
Very heavy audit policy → 12h or 24h

Implementation

Add the new field to the Policy and ClusterPolicy CRDs.
Update the background controller to respect per-policy intervals.
Add validation to ensure the duration format is valid.
Update documentation (especially Background Scans and Policy Reports pages).

Link to the Implementation PR
(To be added later)


Drawbacks

Slight increase in complexity for the background controller (needs to track multiple timers/schedules).
Slightly higher memory usage for tracking per-policy schedules (should be negligible).

Alternatives

Keep everything global — simple but doesn't solve the real pain users are facing.
Use multiple Kyverno installations — too heavy and operationally complex.
Rely only on background: false for some policies — loses reporting benefits.

Per-policy interval is the cleanest and most requested solution.


Unresolved Questions

Should we also support backgroundScanWorkers per policy? (probably out of scope for now)
Exact behavior when a policy is updated — should the new interval take effect immediately?
UI / Kyverno Dashboard implications (if any).
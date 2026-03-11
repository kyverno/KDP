# Meta
[meta]: #meta
- Name: Pull Request Milestone Tracking and Backport Automation
- Start Date: 2025-03-11
- Update data (optional): 2025-03-11
- Author(s): (Github usernames)
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

This KDP proposes a consistent approach to tracking pull requests on the Kyverno codebase across branches and milestones using GitHub’s milestone field. Each PR is assigned exactly one milestone corresponding to the release it targets. For changes that must be backported to earlier releases, we standardize on cherry-picking into new PRs against the target branch, with those backport PRs also receiving a milestone. The creation of backport PRs can be automated by labeling or commenting on the original (base) PR, reducing manual work and keeping release tracking accurate.

# Definitions
[definitions]: #definitions

- **Milestone**: A GitHub milestone representing a release (e.g. `v1.15`, `v1.14`). Used to group issues and pull requests for that release.
- **Base PR**: The original pull request that was merged (or is intended to be merged) into the main development branch (e.g. `main`).
- **Backport PR**: A pull request created by cherry-picking commits from a base PR, opened against an older release branch (e.g. `release-1.14`) to include a fix or feature in that release.
- **Cherry-pick**: The act of applying specific commits from one branch onto another, typically to port a fix or small change to a release branch without merging the full history.
- **Release branch**: A long-lived branch corresponding to a minor release (e.g. `release-1.14`), used for patch releases (e.g. 1.14.x).

# Motivation
[motivation]: #motivation

- **Why should we do this?**  
  Today, understanding which PRs belong to which release and how backports are tracked is ad hoc. Using milestones consistently and automating backport PR creation improves release visibility, reduces human error, and makes it easier to answer “what’s in release X?” or “was this change backported to 1.14?”.

- **What use cases does it support?**  
  - Release managers and maintainers can filter PRs by milestone to see what is slated for a given release.  
  - Contributors can request a backport by adding a label or comment instead of manually cherry-picking and opening a PR.  
  - Automation can create backport PRs with the correct base branch and milestone, and optionally link them to the base PR (e.g. in the description or via a comment).

- **What is the expected outcome?**  
  A clear, one-to-one relationship between PRs and milestones, consistent backport workflow, and less manual work when preparing patch releases.

# Proposal
[proposal]: #proposal

- **Milestone assignment**  
  Every pull request (including backport PRs) is assigned exactly one GitHub milestone. The milestone reflects the release the PR is intended for. For a PR targeting `main`, the milestone is the next planned release (e.g. `v1.16`). For a backport PR targeting `release-1.14`, the milestone is that release (e.g. `v1.14`). This gives a single source of truth for “this PR is part of release X.”

- **Backport workflow**  
  When a change needs to go into an older release:
  1. The base PR is merged (or is already merged) into the main development branch.
  2. A backport is requested by either:
     - Adding a label (e.g. `backport release-1.14`) to the base PR, or  
     - Commenting with a standard command (e.g. `/backport release-1.14`).
  3. Automation creates a new PR that:
     - Uses the target release branch (e.g. `release-1.14`) as the base.
     - Cherry-picks the relevant commit(s) from the base PR.
     - Assigns the corresponding milestone (e.g. `v1.14`) to the new PR.
     - Optionally references the base PR in the title or description (e.g. “Backport of #1234 to release-1.14”).

- **Terminology**  
  We use “base PR” for the original PR and “backport PR” for the PR created by cherry-pick. Both are “pull requests” and both have exactly one milestone.

- **Existing users vs new users**  
  Existing contributors continue to open PRs as today; the main change is that milestone assignment becomes mandatory and backporting can be requested via label or comment. New contributors see the same workflow, with documentation explaining milestones and the backport label/comment.

# Implementation
[implementation]: #implementation

- **Milestone enforcement (optional but recommended)**  
  - CI or a GitHub Action can check that every open PR has a milestone set, and optionally that the milestone matches the base branch (e.g. PRs targeting `release-1.14` have milestone `v1.14`).  
  - This keeps the one-to-one relationship and avoids PRs slipping through without release tracking.

- **Backport automation**  
  - Use an existing solution (e.g. [backport](https://github.com/backport-org/backport), [cherry-picker](https://github.com/kubernetes/test-infra/tree/master/prow/plugins/cherrypicker), or a lightweight custom Action) that:  
    - Triggers on label addition or on a comment (e.g. `/backport release-1.14` or `backport/release-1.14`).  
    - Creates a branch from the target release branch.  
    - Cherry-picks the commit(s) from the base PR (typically the merge commit or the squash commit).  
    - Opens a new PR against the release branch with a clear title/description and sets the appropriate milestone.  
  - Configuration (e.g. which labels or commands map to which branches) should be documented and, if possible, versioned in the repo (e.g. `.github/backport.yaml` or workflow inputs).

- **Documentation**  
  - Update contributor/release documentation to describe:  
    - That every PR must have a milestone.  
    - How to request a backport (label vs comment).  
    - How to map “release branch” to “milestone” (e.g. `release-1.14` → `v1.14`).

## Link to the Implementation PR

(To be added when implementation is started.)

# Migration (OPTIONAL)
[migration]: #migration-optional

No breaking changes to public API or compatibility. Existing PRs may lack milestones; migration could be:

- A one-time pass to assign milestones to recent open/merged PRs based on base branch or merge target, or  
- Rely on the new process going forward and leave historical PRs as-is.  
Documentation should state from which date/cutoff the new process applies.

# Drawbacks
[drawbacks]: #drawbacks

- **Process discipline**: Requiring a milestone on every PR adds a small step; without automation reminders, some PRs might be merged without a milestone until tooling enforces it.
- **Backport tooling dependency**: Introducing a bot or Action adds a dependency on that tool’s maintenance and security; choosing a well-maintained or simple solution mitigates this.
- **Cherry-pick conflicts**: Automated cherry-picks can fail due to conflicts; the backport PR would then need manual resolution, which is the same as today but the initial creation is automated.

# Alternatives
[alternatives]: #alternatives

- **Manual backports only**: Keep the current manual process. This avoids new tooling but does not improve consistency or reduce effort.
- **Branch-based tracking only**: Rely only on the base branch of the PR (e.g. `main` vs `release-1.14`) and not milestones. Milestones give an explicit release target and work well with GitHub’s filters and release notes.
- **Multiple milestones per PR**: Allowing a PR to belong to several milestones (e.g. “will be in 1.16 and 1.15”) was considered. A one-to-one relationship (one PR, one milestone) keeps the model simple; backport PRs are separate and get their own milestone, so the set of PRs for a release remains clear.
- **Impact of not doing this**: Without this proposal, release tracking stays inconsistent and backporting remains manual and error-prone, making it harder to answer “what’s in release X?” and to scale patch releases.

# Prior Art
[prior-art]: #prior-art

- **Kubernetes**: Uses Prow’s `cherrypicker` plugin and release branches; contributors use `/cherrypick release-x.y` to request backports. Milestones are used for release tracking.
- **Other CNCF projects**: Many use milestone-on-PR and either labels or bots (e.g. backport, cherry-pick) to automate backports. This aligns with common open-source practice.
- **GitHub’s own features**: Milestones and labels are standard; the proposal uses them in a way that fits GitHub’s UI and APIs.

# Unresolved Questions
[unresolved-questions]: #unresolved-questions

- **Tool choice**: Which backport automation (e.g. Backport app, custom GitHub Action, Prow-style plugin) should be adopted? Criteria may include: ease of setup, maintenance burden, support for label vs comment, and compatibility with Kyverno’s repo layout.
- **Label vs comment**: Should backport requests use only labels (e.g. `backport/release-1.14`), only comments (e.g. `/backport release-1.14`), or both? This can be decided during implementation based on maintainer preference.
- **Milestone naming**: Confirm convention (e.g. `v1.14` vs `1.14`) and alignment with existing GitHub milestones in the Kyverno org.
- **Out of scope**: Broader release automation (e.g. changelog generation, release note drafting) and CI for release branches are out of scope for this KDP but could build on top of consistent milestone and backport tracking.

# CRD Changes (OPTIONAL)
[crd-changes]: #crd-changes-optional

This KDP does not propose any changes to Kyverno CRDs or core API. It concerns repository and release process only.

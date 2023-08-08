# Kyverno Design Proposals

This repository is used to manage Kyverno Design Proposals (KDPs). A design proposal is recommend for any significant change, including new features and enhancements.

Older proposals were managed in documents. All new proposals should be submitted as a PR: https://github.com/kyverno/KDP/pulls.

## Active Proposals

| Name              | Status    | Release |
|------------------ | --------- | ------- |
| [Foreach](https://docs.google.com/document/d/1oZpbhjp6fJMO8KOdtNcaGWTt3DJMioHapQeVRxf8vms/edit) | Implemented | 1.5 |
| [Dynamic Webhooks](https://docs.google.com/document/d/1Y7_7ow4DgCLyCFQcFVz1vHclghazAKZyolIfprtNURc/edit) | Implemented | 1.5 |
| [Image Signature Verification](https://docs.google.com/document/d/1d2Qm47wjjoyGDT8v3_ijB1Q4mGYV5cncAQoQniiR414/edit#heading=h.s8qsd3dl8lqi) | Implemented | 1.6 |
| [Image Attestations ](https://docs.google.com/document/d/1d2Qm47wjjoyGDT8v3_ijB1Q4mGYV5cncAQoQniiR414/edit#heading=h.s8qsd3dl8lqi) | Implemented | 1.6 |
| [Internal Auto-Gen Rules](https://docs.google.com/document/d/1kEST03WoGC2mUIz-rGbZkhQxU2F6n_CbwAkzIlhRdns/edit#heading=h.k9m35w1krgjl) | Implemented | 1.7 |
| [Generate on existing](https://docs.google.com/document/d/1KHf19cV5o8fWWC78OBl3H1-OETRuzHciEEBTfRFfMUU/edit) | Implemented | 1.7 |
| [Mutate on existing](https://github.com/kyverno/KDP/pull/4) | Implemented| 1.7 |
| [CLI test generate policies ](https://github.com/kyverno/KDP/pull/6) | Implemented  | 1.7 |
| [Image Verification Refectoring](https://github.com/kyverno/KDP/blob/main/proposals/image_verify_enhancements.md) | Implemented | 1.7 |
| [Extending Pod Security Admission](/proposals/extend_pod_security_admission.md) | Implemented | 1.8 |
| [YAML Signing and Verification](/proposals/yaml_signing_and_verification.md) | Implemented | 1.8 |
| [Report Aggregation](https://github.com/kyverno/KDP/pull/32) | Implemented | 1.8 |
| [Store Kyverno policies in OCI registries](https://docs.google.com/document/d/15cqD4HPecI5Uv2u1Yfg0JCgWDVi2HLwGZPvTX_48W2E/edit?usp=sharing) | Implemented | 1.9 |
| [Policy Exceptions](https://github.com/kyverno/KDP/pull/33) | In Review | 1.9 |
| [ConfigMap cache enhancement with Informers](/proposals/cache_enhancements.md) | Implemented | 1.9 |
| [Resource Cache](/proposals/resource_cache.md) | Implemented | 1.9 |



## Inactive Proposals

| Name              | Status    |
|------------------ | --------- |
| [SBOM Policy](https://docs.google.com/document/d/1AoaSfJwo6XyAuFZCK4wc4bjiPajdCIEJ9lctS1a8A5Y/edit) | Rejected |

## KDP Process

### Proposal

To get a proposal into Kyverno, first, a KDP needs to be merged into the KDP repo. Once an KDP is merged, it's considered 'Accepted' and may be 'Implemented' to be included in the project. These steps will get an KDP to be considered:

1. Fork the KDP repo: <https://github.com/kyverno/KDP>
2. Copy `template.md` to `proposals/feature.md` (where 'my-feature' is descriptive.).
3. Fill in KDP. Any section can be marked as "N/A" if not applicable.
4. Submit a pull request. The pull request is the time to get review of the proposal from the larger community.
5. Build consensus and integrate feedback. KDPs that have broad support are much more likely to make progress than those that don't receive any comments.
6. Once the pull request is approved by two maintainers, the KDP will enter the 'Final Comment Period'.

### Final Comment Period

When a pull request enters FCP the following will happen:

1. A maintainer will apply the "Final Comment Period" label.
2. The FCP will last 7 days. If there's unanimous agreement amongst the maintainers the FCP can close early.
3. For voting, the binding votes are comprised of the maintainers. Acceptance requires majority of binding votes in favor. The absence of a vote from a party with a binding vote in the process is considered to be a vote in the affirmative. Non-binding votes are of course welcome.
4. If no substantial new arguments or ideas are raised, the FCP will follow the outcome decided. If there are substantial new arguments, then the KDP will go back into development.

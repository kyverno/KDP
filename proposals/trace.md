# Kyverno Design Proposal (KDP): `kyverno trace` Command

##  What is the `trace` Command?

The `trace` command helps users understand **why a Kyverno policy worked or didn’t work** on a given resource.

It gives you a step-by-step breakdown:
- Which rules matched or didn’t
- What conditions were evaluated
- What CEL expressions returned
- Whether a Secret or ConfigMap was found
- If an attestor verified the image or not

This command is useful for **beginners**, **advanced users**, and **maintainers** to troubleshoot and understand policies better.

---

##  Why Do We Need It?

Right now, if something goes wrong:
- You check logs
- You guess which rule failed
- You don’t know which CEL condition caused the problem

With `trace`, you just run a command — and Kyverno **tells you what happened and why**.

---

##  How to Use It

```bash
kyverno trace -p policy.yaml -r resource.yaml
```

This shows the evaluation of every rule in your policy against your resource. You can also control the output using flags.

---

##  All Available Flags (Explained Simply)

| Flag | What it does |
|------|---------------|
| `-p`, `--policy` | Point to your policy file(s) or folder |
| `-r`, `--resource` | Point to the resource YAML(s) you want to test |
| `--focus` | Only trace one rule type: `mutate`, `validate`, `generate`, or `image` |
| `--output` | Show the result in `table` (default), `json`, or `yaml` |
| `--verbose` | Show full CEL expressions, evaluated values, and debug info |
| `--remove-color` | Print plain text without any color formatting |
| `--filter-result` | Only show results that are `pass`, `fail`, `skip`, or `error` |
| `--set` | Pass variables like `--set env=prod` if your policy needs them |
| `--values-file` | Provide values from a YAML/JSON file instead of `--set` |
| `--cluster` | Instead of giving a file, pull the resource directly from the Kubernetes cluster |
| `--watch` | (Future) Watch the resource live and keep tracing in real time like `kubectl get --watch` |

---

##  Examples

```bash
kyverno trace -p policy.yaml -r resource.yaml
```

Simple trace on a local file.

```bash
kyverno trace -p policy.yaml -r resource.yaml --focus=image --verbose
```

Only show image verification logic and full CEL debug info.

```bash
kyverno trace -p policy.yaml -r resource.yaml --filter-result=fail --output=json
```

Only show failed rules, in JSON format.

---

##  Sample Output

```plaintext
 Policy: prod-policy
 Rule: require-team-label
 Matched: resource kind = Deployment
 CEL: object.metadata.labels.team != "" → true
 Passed

 Rule: disallow-latest
 CEL: image endsWith ":latest" → true
 Failed - image uses ':latest'
```

For image verification:

```plaintext
 Image: nginx:1.23
 Attestor 1 (user@example.com)
 CEL: request.userInfo.groups AnyIn ["prod"] → true
 Signature verified → Attestor passed
```

---

##  Why It Helps With Secrets

If your CEL or attestor logic uses `resource.Get(...)` to pull a Secret or ConfigMap, `trace` will show:
- Was the Secret found?
- Did the key exist?
- Did the expression return true or false?

This is extremely useful because many users struggle with dynamic Secret lookups.

---

##  Who This Helps

| Role | Why it helps |
|------|---------------|
| Beginner | Understand why your policy didn’t apply |
| Advanced | Debug tricky CEL logic or image signatures |
| Maintainers | Save time answering “why did my policy fail?” questions |

---

##  Final Note

This command will:
- Make Kyverno easier to use
- Reduce Slack questions
- Improve the CLI experience

It’s a game-changer for anyone writing or debugging policies.

 Author: **Kushal Agrawal**,
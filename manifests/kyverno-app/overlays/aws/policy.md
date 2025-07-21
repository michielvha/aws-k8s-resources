We can check for multiple disallowed values using the `AnyIn` operator in Kyverno

**Option 1: Block specific dangerous ACLs (recommended)**

```yaml
- name: deny-dangerous-acls
  match:
    any:
    - resources:
        kinds: ["Bucket"]
  preconditions:
    all:
    - key: "{{request.object.spec.acl || ''}}"
      operator: NotEquals
      value: ""
  validate:
    message: "ACL values 'public-read', 'public-read-write', and 'authenticated-read' are disallowed for security reasons."
    deny:
      conditions:
        any:
        - key: "{{request.object.spec.acl}}"
          operator: AnyIn
          value: ["public-read", "public-read-write", "authenticated-read"]
```

**Option 2: Allow only specific safe ACLs (more restrictive)**

```yaml
- name: allow-only-safe-acls
  match:
    any:
    - resources:
        kinds: ["Bucket"]
  preconditions:
    all:
    - key: "{{request.object.spec.acl || ''}}"
      operator: NotEquals
      value: ""
  validate:
    message: "Only 'private', 'log-delivery-write', 'bucket-owner-read', and 'bucket-owner-full-control' ACLs are allowed."
    deny:
      conditions:
        any:
        - key: "{{request.object.spec.acl}}"
          operator: AnyNotIn
          value: ["private", "log-delivery-write", "bucket-owner-read", "bucket-owner-full-control"]
```

The key differences:
- `AnyIn` blocks if the value matches any in the list
- `AnyNotIn` blocks if the value is NOT in the allowed list
- Using `deny` section instead of `pattern` for clearer logic with multiple values

For production, I'd recommend Option 1 since it's more explicit about what we're blocking and why.

---

### Kyverno’s **generate**

Think of the three Kyverno rule types like this:

| Rule         | Operates **on**                | Typical action                                        |
| ------------ | ------------------------------ |-------------------------------------------------------|
| **mutate**   | the very object being admitted | change/patch it (e.g., add a label)                   |
| **validate** | the very object being admitted | accept✔️ / reject ✖️ based on tests                   |
| **generate** | **another** Kubernetes object  | *create, clone, or keep in‑sync* a companion resource |

So **generate** is Kyverno’s built‑in little “operator factory”.
When the trigger resource (the one matched by your `match`/`exclude` blocks) appears, changes, or—depending on settings—already exists, Kyverno can automatically **materialise new resources** for you instead of just mutating or gate‑keeping the trigger. ([Kyverno][1])

---

#### What can it build?

* **Anything you can kubectl‑apply:** ConfigMaps, Secrets, NetworkPolicies, RoleBindings, CRDs, even Events.
* The source of the generated object can be

  * a **template you inline** in the rule (`generate.data …`) – allows JMESPath variables, templating, loops (`foreach`). ([Kyverno][1])
  * an **existing object you clone** (`generate.clone …`) – handy for copying an image‑pull Secret, or
  * a **bundle of objects** from one namespace you clone in bulk (`generate.cloneList …`). ([Kyverno][1])

---

#### Key knobs you’ll see in a `generate:` block

| Field                                             | Why it matters                                                                                                                                                                                                                              |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `apiVersion`, `kind`, `name`, `namespace`         | Identity of the resource to create. Templating is allowed (e.g., `namespace: "{{request.object.metadata.name}}"`).                                                                                                                          |
| `data` **vs** `clone`/`cloneList`                 | Choose between “define fresh” or “copy existing”. Only one is allowed.                                                                                                                                                                      |
| `synchronize: true/false`                         | If **true**, Kyverno keeps the downstream object **healed and up‑to‑date**. Edits are reverted; source changes propagate; deleting the trigger deletes the downstream. If **false**, Kyverno creates it once and walks away. ([Kyverno][1]) |
| `generateExisting: true` (or per‑rule equivalent) | “Generate for existing” – do the same for resources that were already in the cluster before the policy appeared. ([Kyverno][1])                                                                                                             |
| `orphanDownstreamOnPolicyDelete`                  | With sync on, delete vs. keep children when the policy goes away. ([Kyverno][1])                                                                                                                                                            |
| `useServerSideApply`                              | Let SSA merge with other controllers instead of fighting them. ([Kyverno][1])                                                                                                                                                               |

---

#### How it works under the hood (one level deeper)

1. The admission controller (or the background controller for “generate existing”) notices a matching trigger and writes an **`UpdateRequest`** CR.
2. A worker controller reconciles that `UpdateRequest`, creates/patches the downstream resource, and updates the UR status (`Completed`, `Failed`, `Pending`, `Skip`).
3. If you enabled `synchronize`, Kyverno sets up watches so future edits or deletions cause a re‑queue and re‑sync. ([Kyverno][1])

You can inspect them with

```bash
kubectl get updaterequests -A
```

to debug why something wasn’t created.

---

#### Typical real‑world uses

* **Namespace bootstrap:** every new namespace instantly gets a default‑deny NetworkPolicy, ResourceQuota, RoleBinding, etc.
* **Secret propagation:** copy an image‑pull Secret from `default` into every app namespace and keep it in sync.
* **Retrofit security controls:** in a brown‑field cluster, create PodDisruptionBudgets or NetworkPolicies for *all* existing Deployments/Namespaces with `generateExisting: true`.
* **Service‑linked config:** when a Service is created, generate a ConfigMap with its cluster‑IP and mark the ConfigMap as an `ownerReference` so it vanishes when the Service is deleted. ([Kyverno][1])

---

#### Why use generate instead of Helm or scripts?

* **Same YAML, same GitOps repo**—no separate operator to run.
* **Declarative drift correction** with `synchronize`.
* **Policy‑scoped permissions:** Kyverno only needs RBAC for the kinds you generate, and it can warn you if they’re missing. ([Kyverno][1])

---

### TL;DR

A **generate rule** turns Kyverno into an on‑cluster “if‑this‑then‑create‑that” engine: match a resource, and Kyverno births (and optionally shepherds) whatever supporting objects you describe—no extra tooling, fully declarative, and governed by the same policy YAML you’re already writing.

[1]: https://kyverno.io/docs/policy-types/cluster-policy/generate/ "Generate Rules | Kyverno"


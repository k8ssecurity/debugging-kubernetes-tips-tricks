# Quick tip: static pods bypass admission control

Most cluster guardrails — Pod Security Admission, validating/mutating webhooks
(Kyverno, OPA/Gatekeeper), and native `ValidatingAdmissionPolicy` — work by
intercepting objects **at the API server**. A **static pod** never goes through
the API server: the `kubelet` creates it directly from a manifest it reads on the
node (the `staticPodPath`, usually `/etc/kubernetes/manifests`, or a configured
URL). The API server only receives a read-only **mirror pod** reflecting what the
kubelet already ran.

Two consequences matter for security:

- **Admission control does not apply.** None of the policy engines above can
  block or mutate a static pod, because they never see the create request. RBAC
  on `pods` is bypassed as well.
- **`hostNetwork` escapes NetworkPolicy.** Static control-plane pods — and many
  attacker-crafted ones — run with `hostNetwork: true`, sharing the node's
  network namespace and IP. The CNI treats that as host traffic, not a policed
  pod endpoint, so `NetworkPolicy` does not filter it. Combined with
  `privileged` and a `hostPath` mount, a static pod is a policy-free,
  host-networked foothold.

**Prerequisite:** creating a static pod requires **file write on the node** (or
control of the kubelet config), so this is a post-node-compromise
escalation/persistence technique, not a remote one.

## Spot static pods

A static pod's mirror pod is owned by a `Node` (not a ReplicaSet/Deployment) and
its name ends with the node name:

```
kubectl get pods -A -o json \
  | jq -r '.items[] | select(.metadata.ownerReferences[]?.kind=="Node")
           | .metadata.namespace + "/" + .metadata.name'
```

On the node, look in the kubelet's static-pod directory (the path comes from the
kubelet config):

```
grep -i staticPodPath /var/lib/kubelet/config.yaml   # or --pod-manifest-path
sudo ls -l /etc/kubernetes/manifests
```

## Defend

- Protect the node's static-pod manifest directory and kubelet config with
  restrictive permissions and file-integrity monitoring.
- Don't rely on admission control alone for node-level threats — pair it with
  runtime detection (Falco / Tetragon) that alerts on new static pods,
  `hostNetwork`/privileged containers, or writes to `/etc/kubernetes/manifests`.
- Minimize what can reach the node filesystem in the first place (SSH,
  `hostPath` mounts, and the container escapes covered in the companion repos).

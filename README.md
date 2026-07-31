# Debugging Kubernetes — tips & tricks

A small, growing collection of quick, practical tips for debugging and securing
Kubernetes at the node and container level.

## Tips

- [Find the host PID of a pod's container](./finding_pid_from_podname/readme.md)
  — go from a pod name to the host PID of its container process (containerd and
  Docker), ready for `nsenter` / `/proc` / namespace inspection.
- [Static pods bypass admission control](./static_pods_bypass_admission/readme.md)
  — why a kubelet-created static pod sidesteps admission policies (and, via
  `hostNetwork`, `NetworkPolicy`), how to spot them, and how to defend.
- [The Downward API (pod metadata as env vars and files)](./downward_api/readme.md)
  — expose a pod's own name/namespace/node/IP, labels, annotations, and resource
  limits to the container without API access; labels/annotations show up as files
  that update at runtime.

> Looking for the deeper walkthrough (`kubectl debug`, ephemeral containers,
> eBPF/Tetragon)? See
> [`debugging_kubernetes`](https://github.com/k8ssecurity/debugging_kubernetes).

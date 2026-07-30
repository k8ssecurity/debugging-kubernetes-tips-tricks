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

> Looking for the deeper walkthrough (`kubectl debug`, ephemeral containers,
> eBPF/Tetragon)? See
> [`debugging_kubernetes`](https://github.com/k8ssecurity/debugging_kubernetes).

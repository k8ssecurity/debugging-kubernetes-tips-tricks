# Debugging Kubernetes — tips & tricks

A small, growing collection of quick, practical tips for debugging Kubernetes at
the node and container level.

## Tips

- [Find the host PID of a pod's container](./finding_pid_from_podname/readme.md)
  — go from a pod name to the host PID of its container process (containerd and
  Docker), ready for `nsenter` / `/proc` / namespace inspection.

> Looking for the deeper walkthrough (`kubectl debug`, ephemeral containers,
> eBPF/Tetragon)? See
> [`debugging_kubernetes`](https://github.com/k8ssecurity/debugging_kubernetes).

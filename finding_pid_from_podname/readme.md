# Quick tip: find the host PID of a pod's container

When you want to debug a container **from the node** — inspecting its
namespaces, reading `/proc/<pid>`, entering it with `nsenter`, or checking its
mounts — you need the **host PID** of the container's main process. Kubernetes
doesn't surface that PID directly; you get there in three hops:
**pod → containerID → PID**.

This works on both runtimes: **containerd** (the default since Kubernetes v1.24,
when dockershim was removed) and legacy **Docker**.

## 1. (Optional) Find the node the pod runs on

PID lookups happen on the node where the pod is scheduled, so note the node and
run the runtime commands there (over SSH, or from a `kubectl debug node/...`
session).

```
kubectl get pod www-demo -o wide
```

## 2. Get the containerID from the pod

The pod status reports the container ID with a runtime prefix
(`containerd://...` or `docker://...`):

```
containerid=$(kubectl get pod www-demo -o json | jq -r '.status.containerStatuses[0].containerID')
echo "$containerid"        # e.g. containerd://3b9c...  or  docker://7f21...
```

Strip the `runtime://` prefix to get the bare ID (this pattern works for both):

```
cid=${containerid#*://}
echo "$cid"
```

## 3. Resolve the container ID to a host PID (on the node)

**containerd** (the default) — via `crictl`:

```
pid=$(sudo crictl inspect "$cid" | jq -r .info.pid)
echo "$pid"
```

**Docker** (legacy) — via `docker inspect`:

```
pid=$(docker inspect -f '{{.State.Pid}}' "$cid")
echo "$pid"
```

> If `crictl` can't locate the runtime, point it at the socket first:
> `export CONTAINER_RUNTIME_ENDPOINT=unix:///run/containerd/containerd.sock`

## 4. Use the PID

With the PID in hand you can inspect the process, its namespaces, or its
filesystem straight from the node:

```
ps aux | grep "$pid"
sudo ls -l /proc/$pid/ns          # the container's namespaces
sudo nsenter -t $pid -a           # enter the container's namespaces
sudo ls /proc/$pid/root           # the container's filesystem, seen from the host
```

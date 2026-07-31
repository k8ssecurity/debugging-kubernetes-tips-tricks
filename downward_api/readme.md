# Quick tip: the Downward API (pod metadata as env vars and files)

## What it is

The **Downward API** lets a container read information about **itself and its
pod** — its name, namespace, node, IP, labels, annotations, and its resource
requests/limits — *without talking to the Kubernetes API server*. Kubernetes
injects the values directly, either as **environment variables** or as **files**
in a small in-memory volume.

## Why use it

- **No API access required.** The pod doesn't need a ServiceAccount token, RBAC,
  or a Kubernetes client just to learn its own name or node. That keeps the pod
  decoupled and least-privilege.
- **Common needs, cheaply met.** Apps frequently want their own
  `POD_NAME`/`POD_NAMESPACE` for logging, `POD_IP`/`NODE_NAME` for peer
  discovery, or their `memory limit` to size a cache or JVM heap.
- **Files can update at runtime.** The whole `labels`/`annotations` maps are only
  available as **files** (not as a single env var), and — unlike env vars, which
  are fixed at container start — those files are **refreshed** when you change the
  pod's labels/annotations. Handy for feature flags or config passed via
  annotations.

## Example

This pod exposes some fields as **env vars** and the full **labels** and
**annotations** maps as **files** under `/etc/podinfo`:

```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: downwardapi-demo
  labels:
    app: downwardapi
    env: prod
  annotations:
    owner: xxradar
    build: "42"
spec:
  containers:
  - name: shell
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      limits:
        cpu: "250m"
        memory: "64Mi"
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: MEM_LIMIT
      valueFrom:
        resourceFieldRef:
          containerName: shell
          resource: limits.memory
    volumeMounts:
    - name: podinfo
      mountPath: /etc/podinfo
      readOnly: true
  volumes:
  - name: podinfo
    downwardAPI:
      items:
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations
EOF
```

### Read the env vars

```
kubectl exec downwardapi-demo -- env | grep -E 'POD_|NODE_|MEM_'
# POD_NAME=downwardapi-demo
# POD_NAMESPACE=default
# NODE_NAME=ip-10-1-2-180
# POD_IP=192.168.1.23
# MEM_LIMIT=67108864
```

### Read the labels/annotations files

```
kubectl exec downwardapi-demo -- ls -l /etc/podinfo
kubectl exec downwardapi-demo -- cat /etc/podinfo/labels
# app="downwardapi"
# env="prod"
kubectl exec downwardapi-demo -- cat /etc/podinfo/annotations
# build="42"
# owner="xxradar"
# ...plus a few kubernetes.io/config.* annotations the system adds
```

### See the files update live

Change a label and an annotation on the **running** pod, wait a moment for the
kubelet to refresh the projected volume, and re-read the files:

```
kubectl label   pod downwardapi-demo env=staging  --overwrite
kubectl annotate pod downwardapi-demo owner=radarsec --overwrite

sleep 60   # projected downward API volumes refresh on the kubelet sync period

kubectl exec downwardapi-demo -- cat /etc/podinfo/labels       # env="staging"
kubectl exec downwardapi-demo -- cat /etc/podinfo/annotations  # owner="radarsec"
```

The **files** reflect the new values; the **env vars** (`POD_NAME`, etc.) do
**not** change until the container restarts — env is captured once at start,
files are projected and refreshed.

## Good to know

- Fields available as **env vars** (`fieldRef`): `metadata.name`,
  `metadata.namespace`, `metadata.uid`, `spec.nodeName`,
  `spec.serviceAccountName`, `status.hostIP`, `status.podIP`, `status.podIPs`,
  and a single label/annotation key via `metadata.labels['<KEY>']` /
  `metadata.annotations['<KEY>']`.
- The **whole** `metadata.labels` / `metadata.annotations` maps are available
  **only** through a `downwardAPI` volume (as files), not as one env var.
- Container resources come via `resourceFieldRef`
  (`requests.cpu`, `limits.memory`, `requests.ephemeral-storage`, ...).

## Cleanup

```
kubectl delete pod downwardapi-demo
```

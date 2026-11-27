# Kubernetes quick reference

Runs on **Docker Desktop's built-in Kubernetes** (kind mode, 3 nodes). kubectl context: `docker-desktop`.

Always check which cluster kubectl is pointing at before applying anything:

```sh
kubectl config current-context        # should say docker-desktop
kubectl config use-context docker-desktop
```

There's also an old `minikube` context left over from boot.dev synergychat (`~/bootdev/k8/`). Its container no longer exists; `minikube delete -p minikube` cleans it up.

## Docker Desktop: the cluster

There's one cluster, managed from **Docker Desktop → Settings → Kubernetes**:

- **Enable/disable Kubernetes**: start or stop the cluster. Disabling it deletes the cluster.
- **Reset Kubernetes cluster**: wipe everything and start fresh. Re-apply afterwards (see below).
- **Node count**: set in the same screen (kind mode only).

Quitting Docker Desktop or restarting the Mac keeps the cluster and everything applied to it.

Keep projects apart with **namespaces**, not separate clusters (this one uses `observability`).

### Setup (fresh or after a reset)

```sh
kubectl apply -k k8s/addons/metrics-server      # first, so the HPA gets metrics straight away
kubectl apply -k k8s/overlays/local
```

**metrics-server** isn't included with Docker Desktop. It powers `kubectl top` and the HPA; without it the HPA shows `cpu: <unknown>` and never scales. The kustomization in `addons/metrics-server/` pins the version and adds `--kubelet-insecure-tls`, because the local nodes use self-signed kubelet certs. Local clusters only.

### Editing the collector pipeline
The OTel pipeline config lives in `k8s/base/config/otel-collector.yaml`. Edit it there (pure OTel YAML, no Kubernetes syntax), then:
```sh
kubectl apply -k k8s/overlays/local && kubectl -n observability rollout restart deploy/otel-collector
```
Kustomize generates the ConfigMap from that file automatically.

### Sending telemetry in

- **Pods in the cluster** send to `otel-collector.observability.svc:4317` (gRPC) or `:4318` (HTTP), via `03-service.yaml`.
- **The local otel collector** (outside the cluster) sends to `localhost:14317` / `:14318`, via the LoadBalancer in `06-services-external.yaml`. If it runs in a Docker container, use `host.docker.internal:14317`, because `localhost` there means the container itself.

## Nodes: how they work

A **node** is a machine that runs pods. In the cloud it's a VM or server; in Docker Desktop each node is a Docker container.

```
docker-desktop cluster
┌───────────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ desktop-control-plane │  │ desktop-worker   │  │ desktop-worker2  │
│ control plane         │  │ worker           │  │ worker           │
│ ─ API server          │  │ ─ kubelet        │  │ ─ kubelet        │
│ ─ scheduler           │  │                  │  │                  │
│ ─ etcd (storage)      │  │ [collector pod]  │  │ [collector pod]  │
│ ─ kubelet             │  │                  │  │                  │
└───────────────────────┘  └──────────────────┘  └──────────────────┘
```

- **Control plane**: the brain. It runs the API server (what `kubectl` talks to), etcd (where applied YAML is stored), and the **scheduler** (which decides which node each pod runs on).
- **Workers**: these run your pods. Each has a **kubelet**, an agent that starts and stops containers on its node when the control plane tells it to.

**What happens on `kubectl apply`:**
1. The YAML goes to the API server and is stored in etcd.
2. A Deployment with `replicas: 2` creates 2 pods.
3. The scheduler picks a node for each pod, checking free CPU and memory and rules like `topologySpreadConstraints`.
4. The kubelet on that node pulls the image and starts the container.

**When a node dies:** Kubernetes notices it stopped reporting in and recreates its pods on the nodes still running. Spreading replicas across nodes means one keeps working throughout.

**Nodes aren't in your YAML.** They belong to the cluster layer (Docker Desktop settings here, EKS/GKE settings in the cloud). YAML says *what* to run; the scheduler decides *where*. You can steer it with spread constraints, affinity or `nodeSelector`.

```sh
kubectl get nodes -o wide          # list nodes, their status and container runtime
kubectl get pods -A -o wide        # every pod, with its NODE column
kubectl describe node desktop-worker   # capacity and which pods are on it
kubectl top nodes                  # live CPU/memory (needs metrics-server)
kubectl top pods -n observability
```

### Why pods don't show in `docker ps`

Pods aren't Docker containers. Each node runs its own **containerd**, and the kubelet starts pod containers there, a layer below Docker. Docker Desktop also hides the node containers themselves by default.

- See the nodes: Settings → Kubernetes → **Show system containers (advanced)**, then `docker ps`.
- See the pod containers inside a node: `docker exec desktop-worker crictl ps`.
- Normally just use `kubectl get pods` / `kubectl logs`.

## Two layers

```
Cluster (infrastructure)  ← Docker Desktop / EKS / GKE / Terraform
  ├─ Add-ons              ← kubectl apply -k k8s/addons/metrics-server
  └─ Workloads (YAML)     ← kubectl apply -k k8s/overlays/local
       Deployments, Pods, Services, ConfigMaps, HPA, PDB, namespaces
```

The cluster is disposable; the YAML is what matters. Keep it in git.

## Workloads

```sh
kubectl apply -f <file-or-folder>     # create/update (a folder applies every .yaml in it)
kubectl apply -k <folder>             # same, for a folder with a kustomization.yaml
kubectl delete -f <file-or-folder>    # remove (the YAML is kept, so you can re-apply)
kubectl scale deploy/<name> --replicas=0   # pause an app without deleting it
kubectl get deploy,pods,svc,hpa -n <namespace>
kubectl logs deploy/<name> -f
kubectl describe pod <pod>            # events: why a pod won't start
```

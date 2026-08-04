# AsciiArtify - local K8s tooling for the PoC

We picked a tool to run Kubernetes locally for the AsciiArtify PoC. Tried three: minikube, kind and k3d. Tested all of them on a Debian 13 box (2 vCPU, 3.7 GB RAM). All run the same YAML, so switching later is cheap.

## What these tools are

- **minikube** - a real single-node cluster, runs in a VM or in a container. Has lots of addons (dashboard, ingress, metrics). Good for learning.
- **kind** - Kubernetes IN Docker. Nodes are just Docker containers, built on kubeadm. Made for CI, very fast.
- **k3d** - wraps k3s (Rancher's slim Kubernetes) into containers. Lightest, fastest to start, ships with ingress, a load balancer, storage and even a built-in local registry (`k3d registry create`).

## Main differences

|                        | minikube          | kind              | k3d                       |
|------------------------|-------------------|-------------------|---------------------------|
| OS                     | Linux, macOS, Win | Linux, macOS, Win | Linux, macOS, Win         |
| Arch                   | amd64, arm64      | amd64, arm64      | amd64, arm64              |
| Container runtime      | Docker, Podman, VM, bare | Docker (Podman via socket) | Docker or Podman  |
| Multi-node             | experimental      | yes (config file) | yes (one flag)            |
| Built-in registry      | addon             | via config only   | yes, `k3d registry create`|
| Ingress / LoadBalancer | addons            | nothing           | Traefik + ServiceLB       |
| Storage                | addon             | nothing           | local-path built in       |
| Start time (our box)   | ~1m22s            | ~46s              | ~32s                      |
| K8s version tested     | v1.35.1           | v1.32.2           | v1.31.1+k3s1              |
| Footprint              | heavy (VM)        | medium            | light                     |
| CI-friendly            | good              | excellent         | excellent                 |

Measured on a Debian 13 box (2 vCPU, 3.7 GB RAM) from command start until nodes are Ready.

## Pros and cons

**minikube**
Pros: easiest for a beginner, best docs, tons of addons, several drivers.
Cons: slow, eats RAM, multi-node is half-broken, needs --force to run as root.

**kind**
Pros: fast, good for CI, true multi-node, no VM layer.
Cons: no load balancer or ingress out of the box, you wire everything yourself, needs Docker.

**k3d**
Pros: fastest start, light, multi-node, ingress + LB + storage + local registry included, works with Docker and Podman, ports easy to expose.
Cons: k3s is not 100% upstream Kubernetes (doesn't matter for a PoC), docs are thinner than minikube's.

## Docker licensing and Podman

Docker Desktop is only free for companies under 250 people and $10M revenue. After that you pay. For a startup that plans to grow this is a real risk, even if Linux Docker Engine stays free.

Podman is a free, open-source, rootless drop-in for Docker. It speaks the same API, so k3d and kind work against it without changes. We tested k3d on Podman - cluster came up in ~9s and ran the same hello app fine.

One quirk: Podman stores images as localhost/name:tag, so `k3d image import` needs fully-qualified names or a tarball. Easy to work around.

Recommendation: standardize on Podman now and avoid the licensing question forever.

## Demo - k3d with hello world

k3d is our pick for the PoC: fastest, lightest, has everything needed out of the box.

Create a cluster (1 server + 1 agent):

```bash
$ k3d cluster create k3d-demo --agents 1 --port "8081:80@loadbalancer" \
    --image rancher/k3s:v1.31.1-k3s1
INFO[0020] Cluster 'k3d-demo' created successfully!
```

Nodes are ready almost immediately:

```bash
$ kubectl get nodes
NAME                    STATUS   ROLES                  AGE    VERSION
k3d-k3d-demo-agent-0    Ready    <none>                 32s    v1.31.1+k3s1
k3d-k3d-demo-server-0   Ready    control-plane,master   36s    v1.31.1+k3s1
```

Build a hello image and load it into the cluster:

```bash
$ docker build -t asciiartify/hello:v1 .
$ k3d image import asciiartify/hello:v1 -c k3d-demo
INFO[0009] Successfully imported 1 image(s) into 1 cluster(s)
```

Deploy it (2 replicas + a LoadBalancer service):

```bash
$ kubectl apply -f deployment.yaml
deployment.apps/hello created
service/hello created

$ kubectl rollout status deployment/hello
deployment "hello" successfully rolled out

$ kubectl get svc hello
NAME    TYPE           CLUSTER-IP     EXTERNAL-IP             PORT(S)
hello   LoadBalancer   10.43.226.16   172.19.0.2,172.19.0.3   80:32576/TCP
```

Check it responds through the load balancer:

```bash
$ curl http://localhost:8081
<h1>Hello from AsciiArtify</h1>
```

Same demo on Podman:

```bash
$ export DOCKER_HOST=unix:///run/podman/podman.sock
$ k3d cluster create k3d-podman --no-lb
INFO[0009] Cluster 'k3d-podman' created successfully!
$ docker save asciiartify/hello:v1 rancher/mirrored-pause:3.6 | k3d image import -c k3d-podman -
$ kubectl apply -f deployment.yaml
$ curl http://localhost:31026
<h1>Hello from AsciiArtify</h1>
```

## Verdict

Use **k3d** for the PoC and daily dev loop.

Use **Podman** as the container runtime so there is no Docker licensing risk.

Keep **kind** in GitHub Actions for CI - it is the standard there.

Keep **minikube** around only for learning and for cases where you need a real upstream single-node cluster.

# AsciiArtify — Local Kubernetes Tooling Concept

**Project:** AsciiArtify (image → ASCII-art via ML)
**Goal:** pick a Kubernetes-based local dev tool for the PoC.
**Candidates:** minikube, kind, k3d.
**Context:** small startup, 2 devs, no DevOps experience, GitHub for SCM, Docker-licensing risk taken into account (Podman as alternative runtime).

---

## 1. Introduction

All three tools run a real Kubernetes cluster on a developer machine, but do it differently:

- **minikube** — runs a single-node cluster in a VM or container (Docker/Podman/QEMU/VMware/etc). Ships addons (dashboard, metrics-server, ingress, registry) and a helper CLI. Closest to "a local production-like cluster".
- **kind (Kubernetes IN Docker)** — runs control-plane and worker nodes as Docker containers. Built on `kubeadm`. The de-facto standard for CI testing of Kubernetes itself.
- **k3d** — wraps **k3s** (Rancher's lightweight K8s, CNCF sandbox) in containers on Docker or **Podman**. Very fast cluster startup, includes ServiceLB + Traefik ingress, multi-node support out of the box.

All were validated practically on a Debian 13 host (`kamdev`, 2 vCPU / 3.7 GB RAM, x86_64).

---

## 2. Characteristics

| Characteristic            | minikube                          | kind                               | k3d                                   |
|---------------------------|-----------------------------------|------------------------------------|---------------------------------------|
| Kubernetes flavor         | Upstream K8s (kubeadm)            | Upstream K8s (kubeadm)             | k3s (lightweight K8s, embedded)       |
| Default topology          | Single node (VM/container)        | Single node (container)            | 1 server + optional agents            |
| Multi-node support        | Experimental (`--nodes`)          | Yes (config file)                  | Yes (`--agents N`)                    |
| Supported OS              | Linux, macOS, Windows             | Linux, macOS, Windows (WSL2)       | Linux, macOS, Windows                 |
| Supported arch            | amd64, arm64                      | amd64, arm64                       | amd64, arm64                          |
| Container runtime        | Docker, Podman, containerd, bare   | Docker (or Podman via socket)      | Docker or Podman                      |
| Cluster creation time*    | ~1m22s                            | ~46s                               | ~32s (Docker) / ~9s (Podman)          |
| K8s version tested        | v1.35.1                           | v1.32.2                            | v1.31.1+k3s1                          |
| LoadBalancer / ingress    | Addons (optional)                 | None built-in (metallb possible)   | Built-in ServiceLB + Traefik          |
| Storage provisioner       | Addon `default-storageclass`      | None (manual)                      | `local-path` built-in                 |
| Addons / extra features   | Rich addon set, dashboard         | Minimal (CI-oriented)              | Minimal, but LB+ingress+storage in    |
| In-cluster monitoring     | metrics-server addon              | optional                           | metrics-server built-in               |
| Automation / CI-friendly  | Good (headless, `--wait`)         | Excellent (fast, images pre-load)  | Excellent (fast, `k3d image import`)  |
| Resource footprint        | Heavy (VM + full control plane)   | Medium                             | Light (k3s is small)                  |
| Run as root (no password) | Needs `--force`                   | Works                              | Works                                 |

*Measured on `kamdev` (2 vCPU / 3.7 GB RAM) from command start until `kubectl get nodes` shows Ready.

---

## 3. Pros & Cons

### minikube
**Pros**
- Easiest onboarding for beginners; single command, sensible defaults.
- Best documentation and biggest community.
- Addon system (dashboard, ingress, registry, metrics) — useful for learning.
- Multiple drivers: Docker, Podman, QEMU, VirtualBox, VMware, even `none` (bare metal).

**Cons**
- Slowest start; runs full control-plane in a VM → heavy on RAM/CPU.
- Multi-node is experimental; scaling story is weak (the team's concern).
- `docker` driver refuses root without `--force`.
- Overkill for quick throwaway PoC clusters.

### kind
**Pros**
- Very fast, lightweight, all-in-Docker.
- True multi-node support via config file — good for testing topology.
- Industry standard for CI (GitHub Actions use it for K8s tests).
- No VM layer; images shared with local Docker cache.

**Cons**
- No built-in LoadBalancer/ingress/storage — you configure them yourself.
- No addon ecosystem, no dashboard out of the box.
- Requires Docker (or a working Podman socket) on the machine.
- More for CI/testing than for a dev-loop PoC with ingress.

### k3d
**Pros**
- Fastest cluster creation (~32s with agent, ~9s with Podman).
- k3s brings ServiceLB, Traefik ingress, and `local-path` storage by default — a "mini production".
- Multi-node, ports easily exposed (`--port 8081:80@loadbalancer`).
- Runs on Docker **or Podman** — direct answer to the Docker-licensing risk.
- Very light on resources — perfect for a 2-person startup laptop.

**Cons**
- k3s is a trimmed distribution (not 100% identical to upstream K8s) — minor for PoC.
- Some knobs differ from stock Kubernetes (e.g., svclb port conflicts when multiple LoadBalancer services share a host port, worked around by disabling Traefik).
- Smaller docs/community than minikube.

---

## 4. Docker licensing risk → Podman

Docker Desktop (macOS/Windows) is **free only for small companies** (<250 employees and <$10M revenue) under the Docker Subscription Agreement; larger orgs must pay. Docker Engine on Linux (Moby, Apache-2.0) stays free, but for macOS/Windows the policy is a real business risk even for a startup that grows.

**Podman** (Apache-2.0, by Red Hat) is a daemonless, rootless, OCI-compliant drop-in replacement for the Docker CLI/API. Key points:

- Free and open-source everywhere, no subscription terms.
- No daemon → more secure, rootless by default.
- Exposes a Docker-compatible API socket, so tools like **k3d, kind, minikube** work against it unchanged.
- **Validated in this PoC:** a k3d cluster was created and ran the hello app purely on Podman (images pre-loaded into the node, since node-side registry DNS was restricted in the test env).

Caveat encountered: Podman normalizes image names (`localhost/asciiartify/hello:v1`), so `k3d image import` needs fully-qualified tags or a tarball (`docker save ... | k3d image import ...`).

**Recommendation:** standardize on Podman as the dev container runtime now; keep k3d (and optionally kind) on top of it. This removes the Docker licensing risk for the product and stays free forever.

---

## 5. Demonstration — k3d (recommended)

Recommended tool for the AsciiArtify PoC: **k3d**. Rationale: fastest startup, built-in ingress/LB/storage, works on both Docker and Podman, lightest footprint. Demo ran on `kamdev` (Debian 13, 2 vCPU / 3.7 GB).

### 5.1 Install
```bash
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
k3d version        # v5.9.0, k3s v1.35.5-k3s1 (default)
```

### 5.2 Create cluster (1 server + 1 agent, Traefik disabled to keep svclb ports clean)
```bash
$ k3d cluster create k3d-demo --agents 1 --port "8081:80@loadbalancer" \
    --k3s-arg "--disable=traefik@server:0" --image rancher/k3s:v1.31.1-k3s1
INFO[0020] Cluster 'k3d-demo' created successfully!
INFO[0020] You can now use it like this:
kubectl cluster-info

real    0m32.123s
```

### 5.3 Nodes ready
```bash
$ kubectl get nodes
NAME                    STATUS   ROLES                  AGE    VERSION
k3d-k3d-demo-agent-0    Ready    <none>                 32s    v1.31.1+k3s1
k3d-k3d-demo-server-0   Ready    control-plane,master   36s    v1.31.1+k3s1
```

### 5.4 Build a hello image and load it into the cluster
```bash
$ docker build -t asciiartify/hello:v1 .            # nginx serving "Hello from AsciiArtify"
$ k3d image import asciiartify/hello:v1 -c k3d-demo
INFO[0009] Successfully imported 1 image(s) into 1 cluster(s)
```

### 5.5 Deploy "Hello World" app (Deployment + LoadBalancer Service)
```bash
$ kubectl apply -f deployment.yaml     # replicas: 2, image asciiartify/hello:v1
deployment.apps/hello created
service/hello created
$ kubectl rollout status deployment/hello
deployment "hello" successfully rolled out
$ kubectl get svc hello
NAME    TYPE           CLUSTER-IP     EXTERNAL-IP             PORT(S)        AGE
hello   LoadBalancer   10.43.226.16   172.19.0.2,172.19.0.3   80:32576/TCP   11s
```

### 5.6 Verify via the load balancer (host port 8081 → LB :80)
```bash
$ curl http://localhost:8081 | grep -oE "<h1>[^<]+</h1>"
<h1>Hello from AsciiArtify</h1>
```

### 5.7 Same demo on Podman (Docker-licensing mitigation)
```bash
$ export DOCKER_HOST=unix:///run/podman/podman.sock
$ k3d cluster create k3d-podman --no-lb --k3s-arg "--disable=traefik@server:0"
INFO[0009] Cluster 'k3d-podman' created successfully!     # ~9s
$ docker save asciiartify/hello:v1 rancher/mirrored-pause:3.6 | k3d image import -c k3d-podman -
$ kubectl apply -f deployment.yaml
$ curl http://$(podman inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' k3d-k3d-podman-server-0):31026
<h1>Hello from AsciiArtify</h1>
```

---

## 6. Conclusions & Recommendations

| Tool       | Verdict for PoC                                        |
|------------|--------------------------------------------------------|
| minikube   | Great for learning, addons, one-node experiments; too heavy/slow and weak scaling for a fast PoC. |
| kind       | Excellent for CI and multi-node topology tests; needs extra setup for LB/ingress/storage. |
| **k3d**    | **Chosen for the PoC.** Fastest, lightest, multi-node, LB+ingress+storage included, runs on Docker or Podman. |

**Recommendations for AsciiArtify:**
1. Use **k3d** as the primary local cluster tool for the PoC and day-to-day dev loop.
2. **Adopt Podman as the container runtime** to remove the Docker Desktop licensing risk now; k3d works with it out of the box (validated above).
3. Keep **kind** as the CI tool (GitHub Actions) — it is the de-facto standard for automated K8s tests and shares the same YAML as k3d.
4. Use **minikube** only for learning and when a full upstream-K8s single node is needed.

**Next steps:** commit a `k3d` + Podman dev setup script, add GitHub Actions with a kind-based pipeline, and start the PoC deployments on k3d.

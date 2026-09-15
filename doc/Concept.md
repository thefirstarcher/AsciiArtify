# Comparison of different Kubernetes-based systems

## Introduction

To deploy Kubernetes infrastructure for development we need a local Kubernetes system. The most
popular options are **minikube**, **kind** and **k3d**. I compared them against the criteria
that matter for our startup:

- Fast cluster creation and deletion, since startups iterate quickly
- Low resource usage, since the Machine Learning workload will be expensive on its own
- Wide OS and architecture support, so any developer machine can run it

All three solve the same problem in different ways:

- minikube emulates a full machine running vanilla Kubernetes
- kind runs vanilla Kubernetes inside containers
- k3d runs a lightweight Kubernetes inside containers

### minikube

Runs a full single-node cluster where the node is a VM or a container. An official Kubernetes
SIG project shipping unmodified upstream Kubernetes.

Supports the widest range of backends: docker, podman, virtualbox and others. Its addon system
enables ingress, dashboard or metrics-server with a single `minikube addons enable` command.

### kind

The name stands for Kubernetes IN Docker: cluster nodes are Docker containers. Also an official
Kubernetes SIG project.

Ships plain upstream Kubernetes, usually the freshest of the three. Backends are docker, podman
(experimental) and nerdctl. Provides nothing beyond the control plane: no ingress, no load
balancer, no metrics.

### k3d

A wrapper around k3s, a lightweight Kubernetes distribution by Rancher/SUSE. k3s is
CNCF-certified but modified: etcd is replaced by SQLite, legacy and alpha APIs are dropped, and
the control plane is consolidated into a single process.

Traefik, metrics-server, local-path-provisioner and a working LoadBalancer are available right
after cluster creation. The only supported backend is Docker, so Podman is not an option.

## Numbers

All numbers are our own measurements, taken with the same script in two environments, one tool
at a time. Cold start includes downloading the images, warm start does not.

### Environment A: local workstation

- Arch Linux 6.19.11
- 12 vCPU
- 15 GiB RAM
- Docker 29.4.0
- minikube v1.38.1
- kind v0.32.0
- k3d v5.8.3

| Metric                | minikube |        kind |          k3d |
| --------------------- | -------: | ----------: | -----------: |
| Cold start, s         |    190.4 |        85.4 |     **54.0** |
| Warm start, s         |     20.9 |        15.4 |     **13.2** |
| Cluster deletion, s   |     16.1 |     **0.9** |      **0.9** |
| Idle RAM, MiB         |      758 |         521 |      **429** |
| Image footprint       |   1.3 GB |    ~1.34 GB |   **292 MB** |
| Kubernetes version    |  v1.35.1 | **v1.36.1** | v1.31.5+k3s1 |
| Pods in `kube-system` |        7 |           8 |            5 |
| Containers on host    |        1 |           1 |            2 |

### Environment B: Google Cloud Shell

- Ubuntu 24.04
- 4 vCPU
- 15 GiB RAM
- Docker 29.7.2
- minikube v1.39.0
- kind v0.32.0
- k3d v5.9.0
- nested containerisation

| Metric                |           minikube |        kind |          k3d |
| --------------------- | -----------------: | ----------: | -----------: |
| Cold start, s         | **fails to start** |        25.9 |         26.9 |
| Warm start, s         |                n/a |        20.5 |     **16.8** |
| Cluster deletion, s   |                n/a |    **0.71** |         0.81 |
| Idle RAM, MiB         |                n/a |       366.8 |    **349.7** |
| Image footprint       |                n/a |     1.34 GB |   **478 MB** |
| Kubernetes version    |                n/a | **v1.37.0** | v1.35.5+k3s1 |
| Pods in `kube-system` |                n/a |           8 |            5 |

Cold start depends heavily on network bandwidth: k3d took 54 s locally and 27 s in Cloud Shell.
Environment A is the realistic reference for a developer machine.

### Supported OS and architectures

All three run on Linux, macOS and Windows, on amd64 and arm64. The real difference is in the
supported backends:

|          | Backends                                         |
| -------- | ------------------------------------------------ |
| minikube | docker, podman, kvm2, virtualbox, ssh and others |
| kind     | docker, podman (experimental), nerdctl           |
| k3d      | **docker only**                                  |

### Automation

kind and k3d both accept a declarative cluster definition:

```bash
kind create cluster --config cluster.yaml
k3d cluster create --config cluster.yaml
```

minikube is configured through flags and `minikube config set`, without a single manifest.

### What comes out of the box

Contents of `kube-system` right after a default cluster creation.

**minikube**, upstream control plane plus a storage provisioner:

```
coredns, etcd, kube-apiserver, kube-controller-manager, kube-proxy, kube-scheduler, storage-provisioner
```

**kind**, upstream control plane and nothing else:

```
coredns ×2, etcd, kindnet, kube-apiserver, kube-controller-manager, kube-proxy, kube-scheduler
```

**k3d**, where the control plane is hidden inside the k3s process, so what is visible is the
added value:

```
coredns, helm-install-traefik, helm-install-traefik-crd, local-path-provisioner, metrics-server
```

Consequences:

- A `LoadBalancer` service gets an external address on k3d immediately. On kind it stays
  `<pending>` until MetalLB is installed. On minikube it needs `minikube tunnel`.
- Ingress works on k3d through the bundled Traefik. minikube has an addon for it. On kind it
  must be installed manually.
- A local registry is one flag on k3d, an addon on minikube, and a shell script on kind.

## Pros and cons

### minikube

**Pros**

- Widest choice of backends, including Podman and real virtual machines
- Addon system enables ingress, dashboard or a registry with one command, which helps a team
  without DevOps experience
- Official Kubernetes SIG project, plain upstream Kubernetes

**Cons**

- Heaviest of the three: 758 MiB idle RAM, 190.4 s cold start, 1.3 GB image
- Cluster deletion takes 16.1 s against 0.9 s for the other two, which is felt in a
  create, verify, destroy loop
- **Does not start in nested container environments.** In Google Cloud Shell the cluster never
  came up:

  ```
  error execution phase wait-control-plane: unable to create ClusterRoleBinding:
  context deadline exceeded
  ```

  The root cause is a cgroup driver mismatch. In a restricted nested cgroup environment
  minikube disables per-QoS cgroups (`kubelet.cgroups-per-qos=false`), after which the kubelet
  passes containerd a cgroupfs-style path while runc runs with the systemd driver and rejects
  it:

  ```
  runc create failed: expected cgroupsPath to be of format "slice:prefix:name"
  for systemd cgroups, got "/k8s.io/8dc5224c..." instead
  ```

  No pod sandbox is created, so the API server never starts. Neither `--force-systemd=false`
  nor `MINIKUBE_FORCE_SYSTEMD=false` helps. kind is not affected because its node image
  prepares the nested cgroup hierarchy itself. This rules minikube out for cloud IDEs and
  containerised CI runners.

### kind

**Pros**

- Lowest friction: one container, one image, no extra moving parts
- Freshest vanilla Kubernetes of the three
- Fastest cluster deletion
- Reference tool for Kubernetes CI, since the Kubernetes project runs its own end-to-end tests
  on kind
- Works in nested container environments

**Cons**

- Provides nothing beyond the control plane. A `LoadBalancer` service stays `<pending>` until
  MetalLB is installed, ingress and a local registry have to be set up by hand
- Largest image footprint, ~1.34 GB

### k3d

**Pros**

- Lightest by every resource metric: 54.0 s cold start, 429 MiB idle RAM, and 292 MB of images,
  roughly a quarter of what the others need on disk
- Traefik, metrics-server, local-path-provisioner and a working LoadBalancer are there from the
  first command, which removes the largest part of the manual setup
- Local registry with a single flag
- Works in nested container environments

**Cons**

- **Not vanilla Kubernetes.** k3s replaces etcd with SQLite and drops legacy and alpha APIs. It
  is certified, but behaviour differs from a managed cloud cluster
- **Docker only.** Unlike minikube and kind it cannot run on Podman
- Two containers and three images instead of one, so more moving parts

### Docker Desktop licensing and Podman

Docker Desktop is free only for small businesses with fewer than 250 employees **and** less
than $10 million in annual revenue, as well as for personal use, education and non-commercial
open source. Paid tiers are Pro $9 to $11, Team $15 to $16 and Business $24 per user per month
(docs.docker.com/subscription/desktop-license, checked September 2026).

The licence covers the **Docker Desktop product**, not Docker as a technology. **Docker Engine**
is Apache 2.0 and free for commercial use at any company size, so the risk can be avoided
without dropping Docker: the engine runs natively on Linux, inside WSL2 on Windows, and through
Colima or Rancher Desktop on macOS. All three tools work with those setups, k3d included,
because they expose the Docker API.

Moving to Podman is a different matter. It narrows the choice to minikube (`--driver=podman`)
and kind (`KIND_EXPERIMENTAL_PROVIDER=podman`), because k3d does not support Podman at all.

### Why k3d

At the PoC stage the priority is the speed of validating a hypothesis. k3d wins on every
resource metric and removes the largest part of the manual setup: load balancing, ingress,
metrics and storage are available from the first command, which saves hours on every iteration
for a team without DevOps experience. It is also the only one of the three that started in both
tested environments.

Its two risks, deviation from upstream and the Docker dependency, are acceptable at this stage
and are addressed in the conclusions.

## Demo

[![asciicast](https://asciinema.org/a/kn4rpZlIjYQuLvdd.svg)](https://asciinema.org/a/kn4rpZlIjYQuLvdd)

The key moment is the output of `kubectl get svc`:

```
NAME    TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
hello   LoadBalancer   10.43.233.120   172.18.0.2    8080:30389/TCP   9s
```

`EXTERNAL-IP` is assigned, so `curl` reaches the application without `kubectl port-forward`.
The same steps on kind would leave that column at `<pending>`.

## Conclusions

**Recommended for the PoC: k3d.** It is the fastest and lightest of the three, it brings load
balancing, ingress, metrics and storage out of the box, and it is the only one that started in
both tested environments. For a team without DevOps experience that removes the largest source
of setup friction.

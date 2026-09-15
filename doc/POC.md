# PoC: GitOps with ArgoCD on k3d

The cluster tool is **k3d**, as selected in [Concept.md](Concept.md).

## Prerequisites

- Docker Engine
- `k3d` v5.8 or newer
- `kubectl`

```bash
k3d version
kubectl version --client
```

## 1. Create the cluster

The ArgoCD interface has to be reachable from the host. We will use built-in loadbalancer instead of port mapping:

```bash
k3d cluster create argo -p "8080:8080@loadbalancer"
```

Port 8080 is used instead of 80 or 443 because k3s already occupies those with the bundled
Traefik.

Verify:

```bash
kubectl cluster-info
kubectl get nodes
```

If the cluster already exists without the mapping, the port can be added without recreating it:

```bash
k3d cluster edit argo --port-add "8080:8080@loadbalancer"
```

## 2. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

And wait untill pods will be Running:

```bash
kubectl get pods -n argocd -w
```

## 3. Expose the interface

By default `argocd-server` is a `ClusterIP` service and is not reachable from outside the
cluster. Switch it to `LoadBalancer` on the port mapped in step 1:

```bash
kubectl patch svc argocd-server -n argocd --type=merge \
  -p '{"spec":{"type":"LoadBalancer","ports":[{"name":"https","port":8080,"targetPort":8080,"protocol":"TCP"}]}}'
```

`--type=merge` is required: the default strategic patch merges the port list instead of
replacing it, which produces a duplicate port name.

Check that load balancer assigned an address:

```bash
kubectl get svc argocd-server -n argocd
```

## 4. Get the admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

## 5. Log in

Open **https://localhost:8080**

- username: `admin`
- password: from the previous step

The certificate is self-signed, so the browser shows a warning on first visit.

## Why not `kubectl port-forward`

Port forwarding is the usual way to reach ArgoCD, but it runs as a foreground or background
process tied to the terminal session: every developer would have to start it again after each
reboot, and a forgotten background job keeps the terminal busy.

The built-in load balancer of k3d gives a permanent address with no extra process. This is one
of the reasons k3d was chosen in Concept.md, so the PoC uses it rather than working around it.

## Verification

The PoC is successful when:

- [x] the cluster is created in under a minute
- [x] all ArgoCD pods are `Running`
- [x] `argocd-server` has an `EXTERNAL-IP`
- [x] the interface opens at https://localhost:8080 and accepts the `admin` credentials

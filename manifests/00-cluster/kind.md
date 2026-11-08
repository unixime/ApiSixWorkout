# Creating the kind cluster

This project runs on a local [kind](https://kind.sigs.k8s.io/) cluster
(Kubernetes in Docker). Each node is a Docker container on the `kind` bridge
network (`172.18.0.0/16`), which is why MetalLB can hand out IPs in the
`172.18.255.x` range that are reachable directly from the host.

## Cluster topology

1 control-plane node + 3 workers. Multiple workers are not required for
APISIX, but they make scheduling behavior (and MetalLB speaker placement)
more realistic than a single-node cluster.

## kind-config.yaml

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
```

No `extraPortMappings` or `kubeadmConfigPatches` are needed: traffic enters
through MetalLB-assigned LoadBalancer IPs, not through host port mappings.

## Create the cluster

```bash
kind create cluster --name kind --config kind-config.yaml
```

`kind` automatically points your current kubectl context at the new cluster
(`kind-kind`). Verify:

```bash
kubectl get nodes
# → kind-control-plane + kind-worker, kind-worker2, kind-worker3, all Ready
```

## Install MetalLB

A `LoadBalancer` service in kind stays `<pending>` forever without a load
balancer implementation. Install MetalLB, then give it an IP range that
belongs to the kind Docker network but is outside Docker's DHCP pool —
the `.255.x` range is the conventional choice:

```bash
helm repo add metallb https://metallb.github.io/metallb
helm install metallb metallb/metallb -n metallb-system --create-namespace
```

```yaml
# metallb-pool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: kind-pool
  namespace: metallb-system
spec:
  addresses:
    - 172.18.255.200-172.18.255.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: kind-l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - kind-pool
```

```bash
kubectl apply -f metallb-pool.yaml
```

The `IPAddressPool` reserves the addresses; the `L2Advertisement` makes
MetalLB speaker pods answer ARP requests for them (layer-2 mode — fine for a
single L2 network like the kind bridge). If Docker gave your kind network a
different subnet, check it with:

```bash
docker network inspect kind -f '{{range .IPAM.Config}}{{.Subnet}} {{end}}'
```

and pick a range inside that subnet.

## Verify

```bash
# Any LoadBalancer service should immediately get an IP from the pool
kubectl get svc -A | grep LoadBalancer
# → EXTERNAL-IP in 172.18.255.200-250
```

## Teardown

```bash
kind delete cluster --name kind
```

This removes all node containers; the `kind` Docker network stays and will be
reused by the next cluster.

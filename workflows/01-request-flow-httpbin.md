# Workflow 01 — How a request reaches the httpbin backend

This document traces a single request from your laptop to the httpbin pod,
and explains the role of every Kubernetes/APISIX resource it passes through.
It covers both plain HTTP and HTTPS (TLS-terminated) requests — the two flows
share everything except the entry point.

## The test requests

```bash
# Plain HTTP
curl -H "Host: httpbin.local" http://172.18.255.200/get

# HTTPS (requires the TLS setup in manifests/06-tls/README.md)
curl --cacert httpbin-local.crt --resolve httpbin.local:443:172.18.255.200 \
  https://httpbin.local/get
```

Expected result: a JSON document from httpbin echoing the request headers,
including `X-Forwarded-For` and `X-Forwarded-Host` added by APISIX.

## The big picture

```mermaid
flowchart TD
    curl["curl (host)<br/>Host: httpbin.local"]

    subgraph host_net["kind bridge network 172.18.0.0/16"]
        metallb["MetalLB<br/>172.18.255.200 (ARP)"]
    end

    subgraph ns_apisix["namespace: ingress-apisix"]
        gw_svc["Service apisix-gateway<br/>LoadBalancer :80 / :443"]
        apisix["APISIX data plane pod<br/>(proxy, TLS termination)"]
        controller["APISIX Ingress Controller<br/>(control plane)"]
        admin["Service apisix-admin :9180"]
        tls_secret["Secret httpbin-tls<br/>(cert + key)"]
    end

    subgraph ns_httpbin["namespace: httpbin"]
        httpbin_svc["Service httpbin<br/>ClusterIP :80"]
        httpbin_pod["httpbin pod<br/>10.244.2.6:8080"]
    end

    gwc["GatewayClass apisix"]
    gw["Gateway apisix<br/>listeners HTTP:80, HTTPS:443"]
    route["HTTPRoute httpbin<br/>host httpbin.local, path /"]

    %% data path
    curl -->|"GET /get (HTTP or HTTPS)"| metallb --> gw_svc --> apisix
    apisix -->|"proxies to endpoint (plain HTTP)"| httpbin_pod
    httpbin_svc -.->|"groups endpoints"| httpbin_pod

    %% control plane
    gwc -.->|"watches"| controller
    gw -.->|"watches"| controller
    route -.->|"watches"| controller
    tls_secret -.->|"referenced by listener"| gw
    controller -->|"pushes routes + SSL via admin API"| admin --> apisix
    route -.->|"backendRef"| httpbin_svc

    classDef datapath fill:#d4edda,stroke:#28a745
    classDef control fill:#fff3cd,stroke:#ffc107
    class curl,metallb,gw_svc,apisix,httpbin_svc,httpbin_pod datapath
    class gwc,gw,route,controller,admin control
```

Green = data path (the request actually flows through these).
Yellow = control plane / declarative config (no traffic, only intent).

## Hop by hop

### 1. MetalLB — gives the gateway a real IP

kind nodes are Docker containers on the `172.18.0.0/16` bridge network, so a
`LoadBalancer` service would stay `<pending>` forever without MetalLB.
MetalLB's `IPAddressPool` (`172.18.255.200–250`) assigned `172.18.255.200` to
the gateway service, and the `L2Advertisement` makes a speaker pod answer ARP
requests for that IP. Because your host sits on the same bridge network, the
request never leaves the machine.

### 2. Service `apisix-gateway` (namespace `ingress-apisix`)

The front door of APISIX. Type `LoadBalancer`, backed by the APISIX
data-plane pod. Everything external clients send to the gateway arrives here.
With the base Helm values it exposes only port 80; the
[ssl.yaml](../manifests/01-apisix/ssl.yaml) overlay adds `443 → 9443` for
HTTPS.

### 3. GatewayClass `apisix`

Declares *which controller* manages Gateways of this class:
`apisix.apache.org/apisix-ingress-controller`. It is the link between the
standard Gateway API and the APISIX-specific controller. No traffic flows
through it — it is pure control-plane metadata.

### 4. Gateway `apisix` (namespace `ingress-apisix`)

Describes the *listeners*: HTTP on port 80, and (once the TLS setup is
applied) HTTPS on port 443 with `tls.mode: Terminate` and a
`certificateRefs` pointer to the `httpbin-tls` Secret. Its
`allowedRoutes.namespaces.from: All` field is what permits HTTPRoutes living
in other namespaces (like `httpbin`) to attach to it. The
`infrastructure.parametersRef` pointing at the `GatewayProxy` resource tells
the controller which APISIX instance (and admin API credentials) to program.

### 4b. Secret `httpbin-tls` (namespace `ingress-apisix`) — HTTPS only

A `kubernetes.io/tls` Secret holding the self-signed certificate and private
key for `httpbin.local`. It must live in the Gateway's namespace because
`certificateRefs` are resolved there. The controller reads it and pushes an
SSL object to APISIX; the data plane then uses the key to terminate TLS.

### 5. HTTPRoute `httpbin` (namespace `httpbin`)

The routing rule: *requests with hostname `httpbin.local` and path prefix `/`
go to service `httpbin` on port 80*. Its `parentRefs` field attaches it to
the Gateway above — with no `sectionName`, so it binds to **all** listeners
whose hostname intersects. That is why the same route serves both HTTP:80
and HTTPS:443 without any change. This is the resource the ingress
controller translates into an APISIX route object.

### 6. APISIX Ingress Controller (namespace `ingress-apisix`)

The translator. It watches Gateway API resources (Gateway, HTTPRoute), and
pushes the resulting route/upstream configuration to the APISIX data plane
through its admin API (service `apisix-admin:9180`, authenticated with the
admin key stored in the `GatewayProxy`). It is never in the data path — if it
crashes, existing routes keep working.

### 7. APISIX data plane (namespace `ingress-apisix`)

The actual proxy. For an **HTTPS** request it first performs the TLS
handshake: the client sends the desired hostname in cleartext as SNI, APISIX
selects the matching SSL object (pushed from the `httpbin-tls` Secret) and
presents its certificate, then decrypts the request. From that point on, the
flow is identical to plain HTTP:

1. matches the `Host` header against the configured routes (`httpbin.local`),
2. matches the path (`/` prefix — everything),
3. rewrites/proxies the request to the upstream (the httpbin service
   endpoints), adding `X-Forwarded-*` headers.

Note the asymmetry: TLS exists **only between client and gateway**. The hop
from APISIX to httpbin is plain HTTP in both flows — termination at the
gateway is exactly what lets APISIX inspect and route the request.

APISIX resolves the backend to pod IPs directly (it watches endpoints), so
inside the cluster the request goes straight to the httpbin pod.

### 8. Service `httpbin` (namespace `httpbin`)

A plain `ClusterIP` service on port 80. It exists so the HTTPRoute has a
stable `backendRef` target; APISIX reads its endpoints (`10.244.2.6:8080`)
and load-balances across them itself.

### 9. httpbin pod

The workload. The container listens on port 8080 and answers with a JSON echo
of the request. It knows nothing about gateways, routes, or hostnames — that
decoupling is the point: routing policy lives entirely in the gateway layer.

## Who owns what

| Resource | Kind of role | In the data path? |
|---|---|---|
| MetalLB IPAddressPool/L2Advertisement | Network exposure | Yes (ARP/IP) |
| Service `apisix-gateway` | Entry point | Yes |
| GatewayClass / Gateway / HTTPRoute | Declarative intent | No (config only) |
| Secret `httpbin-tls` | TLS credentials (HTTPS only) | No (read by controller) |
| Ingress Controller | Control plane (configures APISIX) | No |
| APISIX data plane | Proxy / TLS termination point | Yes |
| Service `httpbin` + pod | Backend | Yes |

## Verify it yourself

```bash
# The route was accepted by the gateway and its backend resolved
kubectl describe httproute httpbin -n httpbin
# → Conditions: Accepted=True, ResolvedRefs=True

# The gateway has an external IP from MetalLB
kubectl get svc apisix-gateway -n ingress-apisix
# → EXTERNAL-IP 172.18.255.200

# End-to-end (HTTP)
curl -i -H "Host: httpbin.local" http://172.18.255.200/get
# → HTTP/1.1 200 OK, JSON body with X-Forwarded-Host: httpbin.local

# End-to-end (HTTPS, after the manifests/06-tls setup)
curl -v --cacert httpbin-local.crt --resolve httpbin.local:443:172.18.255.200 \
  https://httpbin.local/get
# → handshake shows "subject: CN=httpbin.local" and "SSL certificate verify ok",
#   then HTTP 200 with the same JSON body
```

## Things to try next

- Remove the `Host` header (`curl http://172.18.255.200/get`) → APISIX
  returns 404: no route matches, proving host-based routing works.
- Change the HTTPRoute `path` match from `/` to `/get` and re-apply →
  `/headers` starts returning 404 while `/get` still works.
- Over HTTPS, request the IP directly (`curl -vk https://172.18.255.200/get`)
  → no SNI hostname is sent, so cert selection and routing behave
  differently: a good way to see why SNI matters.

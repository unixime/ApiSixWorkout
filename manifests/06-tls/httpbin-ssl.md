# TLS termination: HTTPS to httpbin

Goal: reach the httpbin backend over **HTTPS** through the gateway:

```bash
curl --cacert httpbin-local.crt --resolve httpbin.local:443:172.18.255.200 \
  https://httpbin.local/get
```

## The concept

With **TLS termination**, the gateway holds the certificate/private key and
decrypts incoming HTTPS. Traffic between the gateway and the backend stays
plain HTTP:

```
client --(TLS handshake, SNI httpbin.local)--> APISIX --(plain HTTP)--> httpbin
```

During the handshake the client sends the desired hostname in cleartext
(**SNI**, Server Name Indication). APISIX uses it to pick which certificate to
present and, after decryption, which route matches the `Host` header. This is
what lets one IP address serve many hostnames.

The alternative, **TLS passthrough**, forwards the encrypted stream to the
backend untouched. The APISIX ingress controller effectively does not support
passthrough on Gateway listeners, and we don't want it anyway: terminating TLS
at the gateway is what allows APISIX to inspect requests and apply L7 routing
and plugins.

## Two ways to configure it

| | Gateway API HTTPS listener (used here) | `ApisixTls` CRD |
|---|---|---|
| Resource | `Gateway` listener with `tls.certificateRefs` | `apisix.apache.org/v2` `ApisixTls` |
| Fits our setup | Yes — routing is already Gateway API (`Gateway` + `HTTPRoute`) | Better when routing with `ApisixRoute` CRDs only |
| What the controller creates | APISIX SSL object + route binding | APISIX SSL object directly |

Both end up as the same SSL object inside APISIX; only the Kubernetes-side
API differs. This example uses the Gateway API listener because the rest of
the project is Gateway API based. An `ApisixTls` example can live in this
folder later.

## Why three changes are needed

1. **A certificate** — clients need something to verify. We use a throwaway
   self-signed cert for `httpbin.local`.
2. **An HTTPS listener on the Gateway** — tells the controller which cert to
   use for which hostname.
3. **A TLS port on the data plane** — the controller *cannot open ports
   dynamically* ([docs](https://apisix.apache.org/docs/ingress-controller/concepts/gateway-api/));
   APISIX must already listen on 9443 and the `apisix-gateway` Service must
   expose 443. The chart does this via `apisix.ssl.enabled: true`, kept in a
   separate overlay file [../01-apisix/ssl.yaml](../01-apisix/ssl.yaml) so the
   base install ([../01-apisix/values.yaml](../01-apisix/values.yaml)) stays
   HTTP-only and both configurations are supported.

## Steps

### 1. Generate a throwaway self-signed certificate

```bash
openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout httpbin-local.key -out httpbin-local.crt \
  -days 365 -subj "/CN=httpbin.local" \
  -addext "subjectAltName=DNS:httpbin.local"
```

- `-addext subjectAltName=...` matters: modern clients (curl included)
  validate the hostname against the **SAN**, not the CN.
- This key is a **throwaway demo value** — it is generated locally and never
  committed (AGENTS.md rule: no private TLS keys in the repo). That is why the
  Secret below is created imperatively, as an explicit exception to the
  "declarative YAML" convention.

### 2. Create the TLS Secret in the Gateway's namespace

```bash
kubectl create secret tls httpbin-tls -n ingress-apisix \
  --cert=httpbin-local.crt --key=httpbin-local.key
```

It must live in `ingress-apisix` because `certificateRefs` are resolved in the
Gateway's own namespace. Referencing a Secret elsewhere would require a
`ReferenceGrant`.

### 3. Enable the data-plane TLS port (Helm values change)

```bash
helm upgrade apisix apisix/apisix -n ingress-apisix \
  -f manifests/01-apisix/values.yaml -f manifests/01-apisix/ssl.yaml
```

Helm merges multiple `-f` files left to right, so `ssl.yaml` overlays the base
`values.yaml` without modifying it. To go back to HTTP-only, re-run the same
command with only the base file.

**What this affects:** APISIX reconfigures to listen on 9443 and the
`apisix-gateway` Service gains `443 -> 9443` alongside `80 -> 9080`. The pod
restarts briefly; the Service is updated in place, so MetalLB keeps the
existing IP (172.18.255.200). Plain HTTP on port 80 keeps working.

### 4. Add the HTTPS listener to the Gateway

Already prepared in [../01-apisix/gateway.yaml](../01-apisix/gateway.yaml):

```bash
kubectl apply -f manifests/01-apisix/gateway.yaml
```

No `HTTPRoute` change is needed: the route's `parentRef` has no `sectionName`,
so it binds to **all** listeners whose hostname intersects — `httpbin.local`
is now served on both 80 (HTTP) and 443 (HTTPS).

## Verify

```bash
# Listener accepted and cert resolved
kubectl describe gateway apisix -n ingress-apisix | sed -n '/Listeners/,$p'

# Service now exposes 443
kubectl get svc apisix-gateway -n ingress-apisix

# HTTPS request (proper cert validation against the generated cert)
curl --cacert httpbin-local.crt --resolve httpbin.local:443:172.18.255.200 \
  https://httpbin.local/get
```

Expected: HTTP 200 with the usual httpbin JSON.

- `--resolve httpbin.local:443:172.18.255.200` is a per-command static DNS
  entry: curl connects to the MetalLB IP but uses `httpbin.local` for SNI and
  the `Host` header. (Alternative: add a line to `/etc/hosts`.)
- `--cacert` makes curl trust our self-signed cert. Using `-k` instead also
  works but skips validation entirely — try `curl -v` once without either flag
  to see the verification failure, it's instructive.
- Watch the handshake with `curl -v`: look for `subject: CN=httpbin.local`
  and `SSL certificate verify ok`.

## Clean up

```bash
# Remove the HTTPS listener (edit gateway.yaml or re-apply a copy without it),
# then:
kubectl delete secret httpbin-tls -n ingress-apisix
rm httpbin-local.crt httpbin-local.key

# Optionally disable the data-plane TLS port again (base values only)
helm upgrade apisix apisix/apisix -n ingress-apisix -f manifests/01-apisix/values.yaml
```

## Follow-up experiments

- Request `https://172.18.255.200/get` directly (IP in URL, no SNI hostname)
  and observe which cert is served and why routing fails or succeeds.
- Redirect HTTP → HTTPS with a `RequestRedirect` filter on the HTTPRoute.
- Serve a second hostname with its own cert to see SNI-based cert selection.

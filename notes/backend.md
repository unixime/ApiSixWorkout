
# Backend applications

## HTTPBin


Add the Helm repository that hosts the `httpbin` chart (a fork of
`mccutchen/go-httpbin`). This only registers the repo locally under the name
`matheusfm`; it does not install anything in the cluster.

```
helm repo add matheusfm https://matheusfm.dev/charts
```

Install the chart as release `httpbin` in the `demo` namespace (created if
missing). Backends live in `demo` so they stay separate from the gateway in
`ingress-apisix`. The chart creates a `ClusterIP` service named `httpbin` on
port 80 — reachable only inside the cluster, which is fine because all
external traffic will arrive through the APISIX gateway.

```
helm install httpbin matheusfm/httpbin --namespace demo --create-namespace
```
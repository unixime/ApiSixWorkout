# AGENTS.md

Guidance for AI coding agents working in this repository.

## Purpose

This is a **learning project** for Apache APISIX on Kubernetes. The goal is to
understand how APISIX works, how to install it, and how to configure it through
its Ingress Controller CRDs (`ApisixRoute`, `ApisixUpstream`, `ApisixTls`,
`ApisixConsumer`, `ApisixPluginConfig`, `ApisixGlobalRule`, etc.).

Backends are deliberately simple (mainly [httpbin](https://httpbin.org)) so the
focus stays on the gateway, not the apps behind it.

Because the goal is learning, **explaining matters as much as the result**.

## How to work with me

- Explain *why* a resource or field is configured the way it is, not just what.
- Prefer small, incremental examples: one concept per manifest or directory.
- Add short YAML comments on non-obvious fields (route match rules, plugin config, upstream settings).
- When introducing a new concept (e.g. a plugin or CRD), give a one-paragraph summary and a command to verify it works.
- If there are several ways to do something (CRD vs. Gateway API, `ApisixRoute` vs. plugin config), briefly compare them and say which one the example uses.
- Don't silently change my setup. Say what you are about to apply and what it affects.

## Tech stack

- Gateway: Apache APISIX + APISIX Ingress Controller
- Platform: Kubernetes (local cluster, e.g. kind / minikube / k3d)
- Packaging: Helm for installing APISIX; plain YAML manifests for everything else
- Test backend: httpbin (`kennethreitz/httpbin` or `mccutchen/go-httpbin`)
- Tools: `kubectl`, `helm`, `curl`, optionally `jq`

> APISIX and its CRDs change between versions. **Check the installed
> controller/chart version and the official docs before writing manifests.**
> Use the current `apisix.apache.org/v2` API versions, not deprecated ones like
> `v2beta3`. If unsure, say so rather than guessing field names.

## Project structure

```
manifests/
  00-cluster/       Cluster setup notes/config (kind config, namespaces)
  01-apisix/        Helm values and install notes for APISIX + ingress controller
  02-backends/      httpbin Deployments and Services
  03-routes/        ApisixRoute examples (path, host, header matching)
  04-upstreams/     ApisixUpstream examples (load balancing, retries, health checks)
  05-plugins/       Plugin examples (rate limiting, auth, CORS, rewrite, etc.)
  06-tls/           ApisixTls examples
notes/              My learning notes, one file per topic
scripts/            Helper scripts (setup, teardown, smoke tests)
workflows/          End-to-end experiment walkthroughs: trace a request through
                    all the resources involved, with diagrams and verification
                    steps (one file per experiment)
```

Each numbered folder should have a short `README.md` describing the concept,
how to apply the manifests, and how to verify the result.

## Common commands

```bash
# Install / upgrade APISIX
helm repo add apisix https://charts.apiseven.com
helm repo update
helm upgrade --install apisix apisix/apisix -n ingress-apisix --create-namespace -f manifests/01-apisix/values.yaml

# Apply an example
kubectl apply -f manifests/03-routes/<file>.yaml

# Inspect resources
kubectl get apisixroutes,apisixupstreams,apisixtls -A
kubectl describe apisixroute <name> -n <namespace>

# Controller logs (useful when a CRD isn't being picked up)
kubectl logs -n ingress-apisix deploy/<ingress-controller-deployment> -f

# Reach the gateway locally
kubectl port-forward -n ingress-apisix svc/<apisix-gateway-service> 9080:80
curl -i http://127.0.0.1:9080/get

# Clean up an example
kubectl delete -f manifests/03-routes/<file>.yaml
```

Replace the placeholders with the actual names once the install is in place and
keep this section updated.

## Conventions

- One concept per manifest; name files descriptively (`route-path-prefix.yaml`, `plugin-rate-limit.yaml`).
- Use dedicated namespaces (e.g. `ingress-apisix` for the gateway, `demo` for backends).
- Keep backends tiny: httpbin endpoints like `/get`, `/headers`, `/status/<code>`, `/delay/<n>`, `/anything` cover most experiments.
- Pin image and chart versions; avoid `latest`.
- Every example should include the `curl` command that proves it works, plus the expected result.
- Prefer declarative YAML over imperative `kubectl` commands, so everything is reproducible.

## Verifying changes

After applying any manifest:

1. Check the resource status: `kubectl describe <kind> <name>`.
2. Check controller logs for sync errors.
3. Send a test request with `curl -i` through the gateway and confirm status code and headers.
4. For negative cases (blocked, rate-limited, unauthorized), show the failing request too.

## Boundaries

**Always**
- Read existing manifests and notes before adding new ones.
- Use the namespaces and naming conventions above.
- Include a way to verify and a way to clean up each example.

**Ask first**
- Upgrading APISIX, the controller, or the Kubernetes version.
- Changing Helm values that affect the whole gateway.
- Adding new tools or dependencies (service mesh, observability stack, etc.).
- Deleting namespaces or anything outside this project's resources.

**Never**
- Commit secrets, real API keys, or private TLS keys (use throwaway demo values and mark them clearly).
- Expose the cluster or gateway publicly.
- Use deprecated API versions without calling it out.
- Invent CRD fields or plugin options; check the docs.

## Useful references

- APISIX docs: https://apisix.apache.org/docs/
- APISIX Ingress Controller docs: https://apisix.apache.org/docs/ingress-controller/
- APISIX plugin hub: https://apisix.apache.org/plugins/
- httpbin: https://httpbin.org

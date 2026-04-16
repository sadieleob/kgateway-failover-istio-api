# kgateway Failover using Istio APIs (ServiceEntry + DestinationRule)

Native upstream failover is not yet implemented in kgateway ([gloo-gateway#72](https://github.com/solo-io/gloo-gateway/issues/72)).
This workaround uses Istio `ServiceEntry` + `DestinationRule` APIs, which kgateway translates into Envoy config when Istio integration is enabled.

## Why ServiceEntry?

The basic DestinationRule-only approach (locality failover on a single Kubernetes Service) requires all endpoints behind **one hostname**. This doesn't work when backends have **different DNS names per region** (e.g., `east1.api.example.com`, `west2.api.example.com`).

**ServiceEntry solves this** by acting as an endpoint aggregator: it groups multiple external endpoints (each with different addresses/DNS names) under one abstract hostname. Combined with DestinationRule's outlier detection and locality failover, Envoy routes to the nearest healthy endpoint and fails over to remote ones when the primary is unavailable.

## Architecture

```
                          ServiceEntry
                     "my-backend.internal"
                    ┌──────────────────────┐
                    │                      │
                    │  endpoint 1 (P0)     │──→ backend-west2.svc (us-west-2) ── PRIMARY
                    │  locality: us-west-2 │
                    │                      │
                    │  endpoint 2 (P1)     │──→ backend-east1.svc (us-east-1) ── FAILOVER
                    │  locality: us-east-1 │
                    └──────────────────────┘
                              │
                    DestinationRule
                    - outlierDetection (eject after 3 failures)
                    - localityLbSetting (failoverPriority by region)
                              │
                    HTTPRoute (backendRef kind: Hostname)
                              │
                         kgateway proxy
                      (runs in us-west-2)
```

## How Priority is Assigned

Priority is **not manually configured** anywhere in the YAML. It is automatically derived at translation time by comparing the **proxy's node topology labels** against each **endpoint's declared locality**.

When `failoverPriority` is set (e.g., `["topology.kubernetes.io/region"]`), kgateway compares the value of each listed label on the proxy's node against the corresponding value on each endpoint. If all labels match, the endpoint gets **priority 0** (highest preference). Each label mismatch increases the priority number (lower preference).

In this demo, the proxy runs on a node labeled `topology.kubernetes.io/region: us-west-2`. With `failoverPriority: ["topology.kubernetes.io/region"]`:

| Endpoint | Endpoint locality | Region match with proxy? | Assigned priority |
|---|---|---|---|
| backend-west2 | `us-west-2/us-west-2a` | Yes | **0** (preferred) |
| backend-east1 | `us-east-1/us-east-1a` | No | **1** (failover) |

### Multiple failoverPriority labels

You can specify multiple labels for finer-grained priority tiers. For example, with `failoverPriority: ["topology.kubernetes.io/region", "topology.kubernetes.io/zone"]`:

| Match | Priority |
|---|---|
| region + zone match | 0 |
| region matches, zone differs | 1 |
| region differs | 2 |

### Without failoverPriority

If `failoverPriority` is not set but locality failover is configured via the `failover` field, kgateway falls back to direct locality comparison:

| Match | Priority |
|---|---|
| region + zone + subzone match | 0 |
| region + zone match | 1 |
| region match | 2 |
| no match | 3 |

### Key takeaway

No manual priority assignment is needed. Deploy the proxy in the desired "primary" region, declare each endpoint's locality in the ServiceEntry, and the priority hierarchy is computed automatically. When the P0 endpoint is ejected via outlier detection, Envoy overflows traffic to P1.

## Prerequisites

- **Enterprise kgateway 2.1.4+** (or OSS kgateway with Istio integration)
- **Istio CRDs installed** (no Istio control plane or sidecars needed):
  ```bash
  kubectl apply -f https://raw.githubusercontent.com/kgateway-dev/kgateway/refs/heads/main/pkg/kgateway/setup/testdata/istio_crds_setup/crds.yaml
  ```
- **Istio integration enabled** on the kgateway controller:
  ```bash
  helm upgrade enterprise-kgateway <chart> -n kgateway-system --reuse-values \
    --set controller.extraEnv.KGW_ENABLE_ISTIO_INTEGRATION=true
  ```

## Quick Start

### 1. Create a Kind cluster with topology labels

```bash
cat <<EOF | kind create cluster --name failover-demo --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
  labels:
    topology.kubernetes.io/region: us-east-1
    topology.kubernetes.io/zone: us-east-1a
- role: worker
  labels:
    topology.kubernetes.io/region: us-west-2
    topology.kubernetes.io/zone: us-west-2a
EOF
```

### 2. Deploy the demo

```bash
kubectl create ns failover-demo
kubectl apply -f failover-demo-full.yaml
```

### 3. Test normal traffic (should hit PRIMARY)

```bash
GW_IP=$(kubectl get gateway failover-demo-gw -n kgateway-system -o jsonpath='{.status.addresses[0].value}')
for i in $(seq 1 5); do curl -s -H "host: se-failover.example.com" http://$GW_IP:8080/; done
```

### 4. Simulate regional failure

```bash
kubectl -n failover-demo scale deployment nginx-primary --replicas=0
# Wait a few seconds, then send requests with 1s delay
for i in $(seq 1 8); do
  curl -s -H "host: se-failover.example.com" --max-time 3 http://$GW_IP:8080/
  sleep 1
done
# First 3 requests fail (outlier detection threshold), then traffic moves to FAILOVER
```

### 5. Restore primary

```bash
kubectl -n failover-demo scale deployment nginx-primary --replicas=2
kubectl -n failover-demo rollout status deployment nginx-primary
# Wait 30s+ for ejection timer to expire
sleep 35
for i in $(seq 1 5); do curl -s -H "host: se-failover.example.com" http://$GW_IP:8080/; done
# Traffic returns to PRIMARY
```

## Key Configuration Details

### ServiceEntry (the endpoint aggregator)

```yaml
apiVersion: networking.istio.io/v1
kind: ServiceEntry
spec:
  hosts:
  - my-backend.internal          # abstract hostname
  resolution: DNS                 # Envoy resolves each endpoint's address via DNS
  location: MESH_INTERNAL
  endpoints:
  - address: west2.api.example.com   # region-specific DNS
    locality: us-west-2/us-west-2a
  - address: east1.api.example.com   # different region DNS
    locality: us-east-1/us-east-1a
```

### DestinationRule (failover + ejection)

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
spec:
  host: my-backend.internal
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 100     # CRITICAL - see below
    loadBalancer:
      localityLbSetting:
        failoverPriority:
        - "topology.kubernetes.io/region"
```

### HTTPRoute (references ServiceEntry)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
spec:
  rules:
  - backendRefs:
    - name: my-backend.internal     # matches ServiceEntry hosts[]
      port: 80
      kind: Hostname                # tells kgateway to look up ServiceEntry
      group: networking.istio.io
```

## Critical: maxEjectionPercent

**You MUST set `maxEjectionPercent: 100`** in the outlier detection config.

Without it, the default is 10%. If you have only 1 endpoint per priority level, ejecting it = 100% ejection in that priority, which exceeds the 10% default. Envoy will detect the failures but **refuse to eject**, and failover never triggers.

## Envoy Translation (what kgateway produces)

```json
{
  "name": "istio-se_failover-demo_my-backend_my-backend.internal_80",
  "type": "STRICT_DNS",
  "outlier_detection": {
    "consecutive_5xx": 3,
    "interval": "10s",
    "base_ejection_time": "30s",
    "max_ejection_percent": 100
  },
  "common_lb_config": {
    "healthy_panic_threshold": {},
    "locality_weighted_lb_config": {}
  },
  "load_assignment": {
    "endpoints": [
      {
        "locality": { "region": "us-west-2", "zone": "us-west-2a" },
        "lb_endpoints": [{ "endpoint": { "address": "backend-west2.svc:80" } }],
        "priority": 0
      },
      {
        "locality": { "region": "us-east-1", "zone": "us-east-1a" },
        "lb_endpoints": [{ "endpoint": { "address": "backend-east1.svc:80" } }],
        "priority": 1
      }
    ]
  }
}
```

## Test Results

| Scenario | Expected | Result |
|----------|----------|--------|
| Normal traffic | PRIMARY (v1) | PRIMARY 10/10 |
| Primary scaled to 0 | FAILOVER after ejection | 3 failures, then FAILOVER 4/4 |
| Primary restored + ejection expired | PRIMARY | PRIMARY 5/5 |

## Limitations

This workaround provides basic locality-based failover. It does **not** support:
- Header-based failover triggers (e.g., `X-routeDecision: SWITCH`)
- Custom retry categories (ONE vs MULTIPLE sequential targets)
- Arbitrary N-hop failover chains (priority is derived from locality hierarchy)
- Per-tenant/per-merchant enrollment logic
- Error code conversion at the proxy level

These advanced requirements need native kgateway failover support (tracked in [gloo-gateway#72](https://github.com/solo-io/gloo-gateway/issues/72)).

## References

- [gloo-gateway#72 - Implement Failover](https://github.com/solo-io/gloo-gateway/issues/72)
- [kgateway-dev/kgateway#13643](https://github.com/kgateway-dev/kgateway/issues/13643)
- [kgateway-dev/kgateway#13507](https://github.com/kgateway-dev/kgateway/issues/13507)
- [Istio ServiceEntry docs](https://istio.io/latest/docs/reference/config/networking/service-entry/)
- [Envoy Outlier Detection](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/outlier)

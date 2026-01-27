# RUNBOOK: DNS Failover Operations

## Overview

Operational procedures for the DNS-based failover system using k8gb and Failover Controller with external DNS witnesses.

## Components

| Component | Purpose | Namespace |
|-----------|---------|-----------|
| k8gb | Health-based DNS routing (stateless) | k8gb |
| Failover Controller | Witness-verified data failover (stateful) | failover-controller |
| External Witnesses | 8.8.8.8, 1.1.1.1, 9.9.9.9 | N/A |

## Health Checks

### Check k8gb Status

```bash
# k8gb pods
kubectl get pods -n k8gb

# k8gb logs
kubectl logs -l app.kubernetes.io/name=k8gb -n k8gb --tail=100

# Gslb resources
kubectl get gslb -A

# Gslb status for specific app
kubectl describe gslb <tenant>-app -n <tenant>-prod
```

### Check Failover Controller Status

```bash
# Pod status
kubectl get pods -n failover-controller

# Logs
kubectl logs -l app=failover-controller -n failover-controller --tail=100

# Health endpoint
kubectl exec -it deploy/failover-controller -n failover-controller -- curl localhost:8080/healthz

# Current status
kubectl exec -it deploy/failover-controller -n failover-controller -- curl localhost:8080/status
```

### Check External Witnesses

```bash
# Test witness reachability from cluster
for resolver in 8.8.8.8 1.1.1.1 9.9.9.9; do
  echo "Testing $resolver..."
  kubectl run -it --rm dns-test-$RANDOM --image=busybox --restart=Never -- \
    nslookup region-1.gslb.<domain> $resolver
done

# Check Failover Controller witness metrics
kubectl exec -it deploy/failover-controller -n failover-controller -- \
  curl localhost:9090/metrics | grep splitbrain_witness
```

### Verify DNS Resolution

```bash
# Test GSLB zone resolution
kubectl run -it --rm dns-test --image=busybox --restart=Never -- \
  nslookup app.gslb.<domain>

# Test via specific k8gb CoreDNS
kubectl exec -it deploy/k8gb -n k8gb -- \
  dig @localhost app.gslb.<domain>
```

## Troubleshooting

### Endpoint Not Being Added to DNS

1. Check Gslb resource status:
   ```bash
   kubectl describe gslb <name> -n <namespace>
   ```

2. Check k8gb logs:
   ```bash
   kubectl logs -l app.kubernetes.io/name=k8gb -n k8gb | grep -i error
   ```

3. Verify backend service is healthy:
   ```bash
   kubectl get endpoints <service> -n <namespace>
   ```

### Split-Brain Detection Issues

1. Check witness reachability:
   ```bash
   kubectl exec -it deploy/failover-controller -n failover-controller -- \
     curl localhost:8080/witnesses
   ```

2. Check witness metrics:
   ```bash
   kubectl exec -it deploy/failover-controller -n failover-controller -- \
     curl localhost:9090/metrics | grep splitbrain
   ```

3. Manual witness test:
   ```bash
   # From outside the cluster
   for resolver in 8.8.8.8 1.1.1.1 9.9.9.9; do
     echo "=== $resolver ==="
     dig @$resolver region-1.gslb.<domain> +short
   done
   ```

### CNPG Promotion Not Triggering

1. Check Failover Controller logs:
   ```bash
   kubectl logs -l app=failover-controller -n failover-controller | grep -i cnpg
   ```

2. Check witness quorum:
   ```bash
   kubectl exec -it deploy/failover-controller -n failover-controller -- \
     curl localhost:8080/status | jq '.quorum'
   ```

3. Check CNPG cluster status:
   ```bash
   kubectl get cluster -n databases
   kubectl describe cluster <cluster-name> -n databases
   ```

### High Failover Latency

1. Check TTL settings:
   - DNS TTL: 30s (Gslb spec)
   - Health check interval: 5s (k8gb)
   - Witness check interval: 10s (Failover Controller)

2. Verify k8gb sync interval:
   ```bash
   kubectl get gslb <name> -n <namespace> -o jsonpath='{.spec.strategy.dnsTtlSeconds}'
   ```

## Manual Failover Procedures

### Force CNPG Promotion (Emergency)

**Warning:** Only use if automatic failover is blocked and you've verified the primary is truly down.

```bash
# 1. Verify primary is unreachable
kubectl exec -it <cluster>-1 -n databases -- pg_isready

# 2. Verify replica status
kubectl exec -it <cluster>-2 -n databases -- psql -c "SELECT pg_is_in_recovery();"

# 3. Promote replica (CNPG handles this)
kubectl patch cluster <cluster> -n databases --type merge \
  -p '{"spec":{"instances":1,"primaryUpdateStrategy":"unsupervised"}}'

# 4. Verify promotion
kubectl exec -it <cluster>-1 -n databases -- psql -c "SELECT pg_is_in_recovery();"
```

### Force DNS Update

```bash
# Trigger k8gb reconciliation
kubectl annotate gslb <name> -n <namespace> k8gb.io/reconcile=$(date +%s) --overwrite

# Verify DNS records updated
dig app.gslb.<domain> +short
```

## Monitoring

### Key Metrics

| Metric | Query | Threshold |
|--------|-------|-----------|
| GSLB healthy endpoints | `k8gb_gslb_healthy_records` | <2 = warning |
| Witness reachability | `splitbrain_witness_reachable` | 0 = warning |
| Quorum status | `splitbrain_quorum_reached` | 0 when quorum lost |
| Promotions triggered | `splitbrain_promotions_total` | Any increase = alert |

### Grafana Dashboard Queries

```promql
# GSLB health by region
sum by (gslb_name) (k8gb_gslb_healthy_records)

# Witness status
splitbrain_witness_reachable

# Recent promotions
increase(splitbrain_promotions_total[1h])

# Failover controller health
up{job="failover-controller"}
```

### Silence Alerts During Maintenance

```bash
# Create silence for planned maintenance
curl -X POST http://alertmanager.monitoring:9093/api/v2/silences \
  -H "Content-Type: application/json" \
  -d '{
    "matchers": [{"name": "alertname", "value": "GslbEndpointDown", "isRegex": false}],
    "startsAt": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'",
    "endsAt": "'$(date -u -d "+1 hour" +%Y-%m-%dT%H:%M:%SZ)'",
    "comment": "Planned maintenance"
  }'
```

## Recovery Procedures

### Failover Controller Down

1. Check pod status:
   ```bash
   kubectl describe pod -l app=failover-controller -n failover-controller
   ```

2. Force restart:
   ```bash
   kubectl delete pod -l app=failover-controller -n failover-controller
   ```

3. If persistent issue, k8gb continues handling DNS failover automatically. Data service failover requires manual intervention until controller is restored.

### k8gb Down

1. Check pod status:
   ```bash
   kubectl describe pod -l app.kubernetes.io/name=k8gb -n k8gb
   ```

2. Force restart:
   ```bash
   kubectl rollout restart deployment/k8gb -n k8gb
   ```

3. If persistent issue, manually update ExternalDNS or parent zone records.

### Region Recovery After Failover

1. Verify region is healthy:
   ```bash
   # Check all pods
   kubectl get pods -A | grep -v Running

   # Check data services
   kubectl get cluster -n databases
   ```

2. Re-enable in k8gb (automatic when healthy):
   ```bash
   # Verify Gslb status shows region
   kubectl describe gslb <name> -n <namespace>
   ```

3. For CNPG, the old primary becomes a replica:
   ```bash
   # Verify replication status
   kubectl exec -it <old-primary> -n databases -- psql -c "SELECT pg_is_in_recovery();"
   ```

## Related

- [ADR-K8GB-GSLB](./ADR-K8GB-GSLB.md)
- [BLUEPRINT-K8GB](./BLUEPRINT-K8GB.md)
- [ADR-FAILOVER-CONTROLLER](../../failover-controller/docs/ADR-FAILOVER-CONTROLLER.md)

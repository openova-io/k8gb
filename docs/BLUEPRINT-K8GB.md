# BLUEPRINT: DNS Failover Deployment

## Overview

Kubernetes manifests for the DNS-based failover system using k8gb and Failover Controller.

## k8gb Deployment

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: k8gb
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: k8gb
  namespace: k8gb
spec:
  interval: 10m
  chart:
    spec:
      chart: k8gb
      version: "0.12.x"
      sourceRef:
        kind: HelmRepository
        name: k8gb
        namespace: flux-system
  values:
    k8gb:
      dnsZone: "gslb.<domain>"
      edgeDNSZone: "<domain>"
      edgeDNSServers:
        - "8.8.8.8"
        - "1.1.1.1"
      clusterGeoTag: "<region>"
      extGslbClustersGeoTags: "<other-region>"
      reconcileRequeueSeconds: 30
```

## Gslb Custom Resource

```yaml
apiVersion: k8gb.absa.oss/v1beta1
kind: Gslb
metadata:
  name: <tenant>-app
  namespace: <tenant>-prod
spec:
  ingress:
    ingressClassName: cilium
    rules:
      - host: app.gslb.<domain>
        http:
          paths:
            - path: /
              pathType: Prefix
              backend:
                service:
                  name: app-service
                  port:
                    number: 80
  strategy:
    type: roundRobin
    splitBrainThresholdSeconds: 300
    dnsTtlSeconds: 30
```

## Failover Controller Deployment

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: failover-controller
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: failover-controller
  namespace: failover-controller
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: failover-controller
rules:
  - apiGroups: ["postgresql.cnpg.io"]
    resources: ["clusters"]
    verbs: ["get", "list", "watch", "patch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: failover-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: failover-controller
subjects:
  - kind: ServiceAccount
    name: failover-controller
    namespace: failover-controller
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: failover-controller-config
  namespace: failover-controller
data:
  config.yaml: |
    witnesses:
      externalDNS:
        resolvers:
          - 8.8.8.8
          - 1.1.1.1
          - 9.9.9.9
        targetHostname: region-1.gslb.<domain>
        queryTimeout: 10s
        quorum: 2

    healthCheck:
      interval: 10s
      failureThreshold: 3

    services:
      - name: cnpg
        namespace: databases
        type: cnpg
        promotionAction: pg_promote
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: failover-controller
  namespace: failover-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      app: failover-controller
  template:
    metadata:
      labels:
        app: failover-controller
    spec:
      serviceAccountName: failover-controller
      containers:
        - name: failover-controller
          image: ghcr.io/openova-io/failover-controller:latest
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 9090
              name: metrics
          resources:
            requests:
              memory: "32Mi"
              cpu: "10m"
            limits:
              memory: "64Mi"
              cpu: "100m"
          volumeMounts:
            - name: config
              mountPath: /etc/failover-controller
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
      volumes:
        - name: config
          configMap:
            name: failover-controller-config
```

## FailoverConfig Custom Resource

```yaml
apiVersion: failover.openova.io/v1alpha1
kind: FailoverConfig
metadata:
  name: split-brain-protection
  namespace: failover-controller
spec:
  witnesses:
    externalDNS:
      resolvers:
        - 8.8.8.8
        - 1.1.1.1
        - 9.9.9.9
      targetHostname: region-1.gslb.<domain>
      queryTimeout: 10s
      quorum: 2

  healthCheck:
    interval: 10s
    failureThreshold: 3
```

## Alert Rules

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: dns-failover-alerts
  namespace: monitoring
spec:
  groups:
    - name: k8gb
      rules:
        - alert: GslbEndpointDown
          expr: k8gb_gslb_healthy_records < 2
          for: 1m
          labels:
            severity: warning
          annotations:
            summary: "Only {{ $value }} healthy GSLB endpoints"

        - alert: GslbAllEndpointsDown
          expr: k8gb_gslb_healthy_records == 0
          for: 30s
          labels:
            severity: critical
          annotations:
            summary: "No healthy GSLB endpoints available"

    - name: failover-controller
      rules:
        - alert: SplitBrainWitnessUnreachable
          expr: splitbrain_witness_reachable == 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "External DNS witness {{ $labels.resolver }} unreachable"

        - alert: SplitBrainAllWitnessesUnreachable
          expr: sum(splitbrain_witness_reachable) == 0
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "All external DNS witnesses unreachable"

        - alert: FailoverTriggered
          expr: increase(splitbrain_promotions_total[5m]) > 0
          labels:
            severity: warning
          annotations:
            summary: "Failover was triggered for {{ $labels.service }}"
```

## Related

- [ADR-K8GB-GSLB](./ADR-K8GB-GSLB.md)
- [RUNBOOK-K8GB](./RUNBOOK-K8GB.md)
- [ADR-FAILOVER-CONTROLLER](../../failover-controller/docs/ADR-FAILOVER-CONTROLLER.md)

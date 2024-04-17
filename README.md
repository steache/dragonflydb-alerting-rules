# dragonflydb-alerting-rules
 
- an example of implementing some alerting rules to dragonflydb (not saying it's correct :) use at your own risk)
- some rules need label dragonfly_cluster in order to work, in PodMonitor, there is section relabelings that calculates this field

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: monitor
  namespace: caches
spec:
  namespaceSelector:
    matchNames:
      - caches
  selector:
    matchLabels:
      app.kubernetes.io/name: dragonfly
  podMetricsEndpoints:
    - port: admin
      metricRelabelings:
        - action: keep
          regex: >-
            (dragonfly_master|dragonfly_blocked_clients|dragonfly_connected_clients|dragonfly_memory_used_bytes|dragonfly_memory_max_bytes|dragonfly_connected_replica_lag_records)
          sourceLabels:
            - __name__
      relabelings:
        - sourceLabels: [pod]
          separator: "-"
          regex: "(.*)-(.*)"
          targetLabel: dragonfly_cluster
          replacement: "$1"
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: dragonfly
  namespace: caches
spec:
  groups:
    - name: dragonfly
      rules:

        - alert: DragonflyClusterHasNoMaster
          expr: sum(dragonfly_master) without (pod, instance, container, job, endpoint) < 1
          for: 0m
          labels:
            severity: critical
          annotations:
            summary: Dragonfly cluster {{ $labels.namespace }}/{{ $labels.dragonfly_cluster }} has no master!

        - alert: DragonflyClusterHasMultipleMasters
          expr: sum(dragonfly_master) without (pod, instance, container, job, endpoint) > 1
          for: 0m
          labels:
            severity: critical
          annotations:
            summary: Dragonfly cluster {{ $labels.namespace }}/{{ $labels.dragonfly_cluster }} has multiple masters!

        - alert: DragonflyBlockingConnections
          expr: increase(dragonfly_blocked_clients[1m]) > 0
          for: 0m
          labels:
            severity: warning
          annotations:
            summary: Dragonfly instance {{ $labels.namespace }}/{{ $labels.dragonfly_cluster }}/{{ $labels.pod }} blocked some connections.

        - alert: DragonflyNoConnections
          expr: dragonfly_connected_clients < 4 and dragonfly_master == 1
          for: 0m
          labels:
            severity: warning
          annotations:
            summary: Dragonfly master instance {{ $labels.namespace }}/{{ $labels.dragonfly_cluster }}/{{ $labels.pod }} has no connections.

        - alert: DragonflyOutOfMemory
          expr: dragonfly_memory_used_bytes / dragonfly_memory_max_bytes * 100 > 90
          for: 0m
          labels:
            severity: critical
          annotations:
            summary: Dragonfly instance {{ $labels.namespace }}/{{ $labels.dragonfly_cluster }}/{{ $labels.pod }} is running out of memory!

        - alert: DragonflyReplicaLagging
          expr: dragonfly_connected_replica_lag_records > 100
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: Dragonfly instance {{ $labels.namespace }}/{{ $labels.dragonfly_cluster }}/{{ $labels.pod }} is lagging behind master!

```

- rules, that cannot be used right now, as there is no dragonfly_maxclients metric
- discussed in https://github.com/dragonflydb/dragonfly/issues/2912

```yaml
- alert: DragonflyTooManyConnections
  expr: dragonfly_connected_clients / dragonfly_maxclients * 100 > 90
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: Dragonfly too many connections {{ $labels.namespace }}/{{ $labels.dragonfly_cluster }}/{{ $labels.pod }}
```

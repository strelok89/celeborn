<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# Helm Chart for Apache Celeborn

[Apache Celeborn](https://celeborn.apache.org) is an intermediate data service for Big Data compute engines (i.e. ETL, OLAP and Streaming engines) to boost performance, stability, and flexibility. Intermediate data typically include shuffle and spilled data.

## Introduction

This chart will bootstrap an [Celeborn](https://celeborn.apache.org) deployment on a [Kubernetes](http://kubernetes.io) cluster using the [Helm](https://helm.sh) package manager.

## Requirements

Configure kubectl to connect to the Kubernetes cluster.

- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl)
- [Helm 3.0+](https://helm.sh/docs/using_helm/#installing-helm)

## Template rendering

When you want to test the template rendering, but not actually install anything. [Debugging templates](https://helm.sh/docs/chart_template_guide/debugging/) provide a quick way of viewing the generated content without YAML parse errors blocking.

There are two ways to render templates. It will return the rendered template to you so you can see the output.

- Local rendering chart templates

```shell
helm template --debug ../celeborn
```

- Server side rendering chart templates

```shell
helm install --dry-run --debug --generate-name ../celeborn
```

More details in [Helm Install](https://helm.sh/docs/helm/helm_install/).
The chart can be customized using the following [celeborn configurations](https://celeborn.apache.org/docs/latest/configuration/#important-configurations).
Specify parameters using `--set key=value[,key=value]` argument to `helm install`.

## Zone-aware worker replication

By default the chart deploys all workers in a single statefulset, and which zone a worker
lands in is whatever the scheduler decides. Pod-level spreading (`worker.affinity`,
`worker.topologySpreadConstraints`) can balance that, but the balance is re-derived on every
scheduling decision: it holds while the constraint is satisfiable and silently degrades when
it is not, for example when one zone is out of capacity for the instance type the workers
need, or when several nodes are replaced at once.

Setting `worker.zoneAwareReplication.enabled` deploys one worker statefulset per zone
instead, which makes zone membership structural rather than emergent:

```yaml
worker:
  # Total across all zones; each zone gets ceil(replicas / zones) = 3.
  replicas: 9
  zoneAwareReplication:
    enabled: true
    zones:
      - name: us-east-1a
        nodeSelector:
          topology.kubernetes.io/zone: us-east-1a
      - name: us-east-1b
        nodeSelector:
          topology.kubernetes.io/zone: us-east-1b
      - name: us-east-1c
        nodeSelector:
          topology.kubernetes.io/zone: us-east-1c
```

This renders `celeborn-worker-us-east-1a`, `-1b` and `-1c`, each carrying a
`celeborn.apache.org/zone` label that is part of its statefulset selector. What it gives you:

- A zone's worker count is declared, not inferred, so it survives concurrent node churn and
  a full recreate.
- Each zone can be rolled, scaled or paused on its own, leaving the other zones serving.
- A worker's ordinal-to-zone mapping is stable, so its identity
  (`<pod>.<service>.<namespace>.svc.<cluster>.local`) implies its zone. That is what makes
  zone-pinned worker selection (see below) predictable.

Notes:

- `worker.replicas` is the total across zones; each zone deploys
  `ceil(worker.replicas / number of zones)`, so 5 replicas over 3 zones is 2 per zone (6
  total). Set `zones[].replicas` to size a zone explicitly, which is also the lever for
  running a zone short while its capacity is constrained.
- `zones[].nodeSelector` is merged over `worker.nodeSelector` and wins on conflicting keys;
  `zones[].affinity` replaces `worker.affinity` for that zone.
- All zones share one headless service, so the service and the pod monitor keep selecting
  workers in every zone. Only the statefulset selectors are zone-scoped.
- Per-zone `nodeSelector` already separates the zones physically, so a zone
  `topologySpreadConstraints` entry becomes redundant once this is enabled. A per-hostname
  spread or anti-affinity rule is still meaningful within a zone.

### Zone-aware replication is not zone-aware data placement

This only controls where worker pods run. It does not make a client prefer a worker in its
own zone: Celeborn allocates slots in the master, and the client-side view of which workers
are eligible comes from worker tags (`celeborn.tags.tagsExpr`, `celeborn.tags.enabled`), not
from pod topology. To keep shuffle traffic inside one zone, tag each worker with its zone and
have each application select its own zone's tag. Note that
`celeborn.client.reserveSlots.rackaware.enabled` does the opposite on purpose: it spreads a
partition's replicas across racks for durability.

### Migrating an existing release

The statefulset names change, so enabling this on a running release replaces
`<fullname>-worker` with the per-zone statefulsets. Every worker is recreated, and with it
its identity, so:

- Worker data on `hostPath` or `emptyDir` volumes does not survive.
- The master holds stale worker registrations until `celeborn.master.heartbeat.worker.timeout`
  elapses.
- In-flight shuffles fail.

Treat it as a maintenance-window change rather than a rolling update.

## Autoscaling workers with KEDA

`worker.autoscaling` creates a [KEDA](https://keda.sh) `ScaledObject` for each worker
statefulset. With zone-aware replication enabled that is one per zone, so each zone scales
on its own load - which is what you want when applications are pinned to a zone, because
their load is genuinely uneven across zones.

Triggers are yours to define; the chart does not assume a metrics backend. They are rendered
through `tpl` against the zone's context, so one definition covers every zone (with
zone-aware replication off there is no zone, and `{{ .zone.name }}` renders `<no value>`):

```yaml
worker:
  terminationGracePeriodSeconds: 21900   # see "Draining on scale-in" below
  autoscaling:
    enabled: true
    maxReplicaCount: 6
    behavior:
      scaleDown:
        stabilizationWindowSeconds: 1800
        policies:
          - type: Pods
            value: 1
            periodSeconds: 900
    triggers:
      - type: prometheus
        metricType: Value
        metadata:
          serverAddress: http://prometheus.monitoring.svc.cluster.local:9090
          query: >-
            max(1 -
              metrics_DeviceCelebornFreeBytes_Value{role="Worker",zone="{{ .zone.name }}"}
              / metrics_DeviceCelebornTotalBytes_Value{role="Worker",zone="{{ .zone.name }}"})
          threshold: "0.7"
```

Celeborn exports gauges as `metrics_<Name>_Value` and counters as `metrics_<Name>_Count`.
The ones worth scaling on are `DeviceCelebornFreeBytes` / `DeviceCelebornTotalBytes` (disk
headroom), `ActiveShuffleSize` and `ActiveShuffleFileCount` (data held), `DirectMemoryUsageRatio`
(the off-heap ceiling that pauses pushes), and `IsHighWorkload` / `PausePushDataStatus` (the
worker is already in trouble). `IsDecommissioningWorker` is how you keep a draining worker from
counting towards the load that triggered its own removal.

The `zone` label comes from `worker.zoneAwareReplication.metricsLabel`, which passes the zone
into `celeborn.metrics.extraLabels` so the worker stamps it on everything it emits. Celeborn
publishes no zone label of its own, and neither does the scrape, so without this a per-zone
query has to match on pod names. `role="Worker"` matters too: masters report the device gauges
for whichever volume holds the Ratis directory.

Mind `metricType`. KEDA defaults to `AverageValue`, where the metric is treated as total work
and the replica count becomes `ceil(metric / threshold)` - right for a sum like
`sum(metrics_ActiveShuffleSize_Value{...})` against a per-worker byte target, wrong for a ratio,
which would collapse the fleet to one or two pods. A saturation ratio needs `metricType: Value`,
where the count becomes `ceil(replicas * metric / threshold)`.

`minReplicaCount` defaults to the statefulset's own replica count, so a zone never scales
below the size it was deployed at unless you set it explicitly.

Three properties of this fleet are worth sizing for. Scaling out is slow: a new worker
usually needs a new node, and it starts with no shuffle data on it, so autoscaling answers
sustained load changes rather than bursts. Scaling in always removes the highest ordinal,
which is not necessarily the least busy worker. And while a worker drains, its statefulset
cannot grow - so a scale-in decided at low load leaves that zone one worker short until the
drain finishes. Pace scale-in conservatively with `behavior.scaleDown`, and consider
excluding draining workers from the trigger query.

### Draining on scale-in

Kubernetes scale-in only deletes a pod, which reaches Celeborn as a `SIGTERM`. That never
enters the decommission path, so a removed worker takes its shuffle data with it and any
application still reading from it loses that stage. Celeborn's safe drain is `DECOMMISSION`,
which waits for the worker's shuffle keys to expire (up to
`celeborn.worker.decommission.forceExitTimeout`, 6h by default) and is only reachable through
the worker's HTTP API.

`worker.autoscaling.drain.enabled` (on by default when autoscaling is enabled) adds a
`preStop` hook that calls it. The hook must not decommission on a rolling update or a node
drain, or every pod replacement would block for hours, so it distinguishes the two: the
statefulset controller lowers `spec.replicas` *before* deleting pods on a scale-in, so a pod
whose ordinal is at or above the desired count is being removed for good and decommissions,
while any other pod shuts down gracefully and comes back. If the desired count cannot be read
the hook falls back to a graceful shutdown, so a broken lookup cannot stall a rollout.

This needs three things:

- `rbac.create: true`. The chart adds `get` on `statefulsets/scale` to the role when the
  drain is enabled; the hook reads the count with the pod's own service account.
- `celeborn.worker.graceful.shutdown.enabled: true`, so the non-scale-in path actually
  persists state and recovers on restart.
- `worker.terminationGracePeriodSeconds` above `celeborn.worker.decommission.forceExitTimeout`.
  This is an upper bound, not a wait: a graceful shutdown still finishes in
  `celeborn.worker.graceful.shutdown.timeout`, so a long grace period does not slow rollouts
  down.

Set `worker.autoscaling.drain.enabled: false` to opt out, but then treat scale-in as
destructive and only let it happen when the fleet is idle.

### Helm and the autoscaler both own `replicas`

The chart keeps rendering `spec.replicas`, so a fresh install starts at the size you asked
for rather than at one. Once KEDA is scaling, a continuous-delivery tool that reconciles the
rendered manifest will fight it over that field. Tell it to ignore the field - in Argo CD,
`ignoreDifferences` on `/spec/replicas` for the worker statefulsets, with
`RespectIgnoreDifferences=true`.

## Documentation

For additional details on deploying the Celeborn Kubernetes Helm chart, please refer to the [Celeborn on Kubernetes](https://celeborn.apache.org/docs/latest/deploy_on_k8s/) documentation.

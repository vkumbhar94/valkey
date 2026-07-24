# Valkey Cluster on Kubernetes

Kubernetes manifests to deploy a 2-node [Valkey](https://valkey.io/) cluster (managed via Kustomize), along with [RedisInsight](https://github.com/RedisInsight/RedisInsight) for browsing/managing the data.

## What's included

- `k8s/statefulset.yaml` — Valkey StatefulSet (2 replicas, cluster mode enabled)
- `k8s/configmap.yaml` — Valkey cluster configuration
- `k8s/service.yaml` — headless service (cluster gossip), ClusterIP service, and LoadBalancer service for clients
- `k8s/cluster-init-job.yaml` — one-shot Job that waits for pods to be ready and initializes the Valkey cluster
- `k8s/pdb.yaml` — PodDisruptionBudget protecting the cluster from voluntary evictions taking down both pods at once
- `k8s/redisinsight.yaml` — RedisInsight deployment + LoadBalancer service
- `k8s/kustomization.yaml` — Kustomize entrypoint tying the above together

## Restart resilience

Pods survive restarts and rescheduling without losing their cluster identity or becoming unreachable:

- **Stable node IDs**: Valkey's cluster identity (`cluster-config-file`, i.e. `/data/nodes.conf`) lives on each pod's PVC. `statefulset.yaml` sets `persistentVolumeClaimRetentionPolicy: {whenDeleted: Retain, whenScaled: Retain}` so that PVC — and the node ID inside it — is never reclaimed as a side effect of normal scaling/deletion. As long as the PVC survives, a restarted pod keeps the same node ID and the cluster still recognizes it.
- **Hostname-based advertising**: each pod launches `valkey-server` with `--cluster-announce-hostname <pod>.valkey-headless.<namespace>.svc.cluster.local --cluster-preferred-endpoint-type hostname` (derived at container startup, since the value is per-pod). The cluster bus, `CLUSTER SLOTS`, and client redirects all key off this stable DNS name instead of the pod's IP, which can change on every restart. The `valkey-headless` service sets `publishNotReadyAddresses: true` so that DNS name resolves as soon as the pod exists, letting peers reconnect during rejoin even before the pod passes its readiness probe.
- **Idempotent bootstrap**: `cluster-init-job.yaml` checks `cluster_state:ok` before running `--cluster create`, so re-applying the manifests against an already-initialized cluster is a no-op rather than an error.
- **Replication-aware readiness**: the readiness probe checks `PING` and, for a replica, also requires `master_link_status:up` — so a replica whose replication link is broken is reported `NotReady` instead of silently looking healthy.

This covers pod restarts and rescheduling where the underlying PVC survives. It does not cover full volume loss (e.g. the PV itself is deleted or wiped) — that scenario still requires manual `CLUSTER FORGET`/`CLUSTER MEET` recovery.

### Known limitation: stuck FAIL flag after a restart race

With only one master (this is a 2-node, 1-master + 1-replica cluster), Valkey's cluster-bus failure detection has no quorum to "un-fail" a node automatically — clearing a FAIL flag normally requires agreement from other masters, and there isn't a second one here. In practice this means: if a pod goes down and comes back up within roughly one `cluster-node-timeout` window of its peer also being busy/restarting, the peer can mark it FAIL and then stop actively re-probing it, even though the pod is back up and otherwise healthy. Verified live: this can leave a replica with `master_link_status:down` and its own `cluster_state:fail`, while the master's `cluster_state` still reports `ok` — so the master alone isn't a reliable signal.

If you see this (an otherwise-healthy pod showing `NotReady`, or `cluster nodes` listing a live peer as `fail`/`fail?`), recover it manually:

```sh
# From the node that has the stale/failed record of the other:
kubectl exec -n development valkey-0 -- valkey-cli cluster forget <stuck-node-id>
kubectl exec -n development valkey-0 -- valkey-cli cluster meet <other-pod-current-ip> 6379

# If the affected pod is a replica, re-point it at its master:
kubectl exec -n development valkey-1 -- valkey-cli cluster replicate <master-node-id>
```

This is a Valkey/Redis Cluster characteristic of small, single-master topologies, not something the manifests can fix by themselves — a real fix would mean either automating this recovery (a watchdog that detects and re-runs it) or moving to a multi-master topology (e.g. 3 masters + 3 replicas) where quorum-based recovery works as designed.

## Deploy

```sh
kubectl apply -k ./k8s
```

By default, all resources are created in the `development` namespace.

If you want to deploy to a different namespace, edit the `namespace` field in `k8s/kustomization.yaml` before applying:

```yaml
namespace: development
```

Change it to your desired namespace and re-run `kubectl apply -k ./k8s`.

## Verifying the cluster

Once the pods are running, the `valkey-cluster-init` Job initializes the cluster automatically. You can check its status with:

```sh
kubectl get jobs -n development
kubectl logs job/valkey-cluster-init -n development
```

To confirm cluster health directly:

```sh
kubectl exec -n development valkey-0 -- valkey-cli cluster info
```

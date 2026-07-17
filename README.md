# Valkey Cluster on Kubernetes

Kubernetes manifests to deploy a 2-node [Valkey](https://valkey.io/) cluster (managed via Kustomize), along with [RedisInsight](https://github.com/RedisInsight/RedisInsight) for browsing/managing the data.

## What's included

- `k8s/statefulset.yaml` — Valkey StatefulSet (2 replicas, cluster mode enabled)
- `k8s/configmap.yaml` — Valkey cluster configuration
- `k8s/service.yaml` — headless service (cluster gossip), ClusterIP service, and LoadBalancer service for clients
- `k8s/cluster-init-job.yaml` — one-shot Job that waits for pods to be ready and initializes the Valkey cluster
- `k8s/redisinsight.yaml` — RedisInsight deployment + LoadBalancer service
- `k8s/kustomization.yaml` — Kustomize entrypoint tying the above together

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

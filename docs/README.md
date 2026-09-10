# k8s-nifi-cluster

[Apache NiFi](https://nifi.apache.org/) routes, transforms and mediates data between systems using
directed graphs. This project runs an Apache NiFi 2.x cluster on Kubernetes as a set of Kustomize
manifests, coordinated through the Kubernetes API and served over HTTPS.

There is no ZooKeeper. Leader election runs on Kubernetes Leases, cluster state lives in
ConfigMaps, and the nodes authenticate each other with a certificate generated on first start into
a shared volume.

## What gets deployed

| Resource | Detail |
| --- | --- |
| Namespace | `nifi`, holding every resource |
| StatefulSet | 2 NiFi nodes, `OrderedReady` pod management |
| Leader election | `KubernetesLeaderElectionManager`, one Lease per role |
| Cluster state | `KubernetesConfigMapStateProvider`, written to ConfigMaps |
| Certificates | self-signed keystore and truststore on a shared PersistentVolumeClaim |
| Access | HTTPS-only Service and an nginx Ingress |
| Autoscaling | HorizontalPodAutoscaler, from 2 replicas |

## Requirements

| Requirement | Value |
| --- | --- |
| Kubernetes | v1.23 or later |
| Client | `kubectl` with Kustomize support |
| Ingress controller | any, see [ingress and access](ingress.md) |
| Storage | ReadWriteMany for the shared certificate volume, see [deploying](deploying.md#the-shared-certificate-volume) |
| Capacity | 2 pods requesting 500m CPU and 2Gi each, with limits of 2 CPU and 4Gi |
| metrics-server | only for the HorizontalPodAutoscaler |

## Quick start

```shell
kubectl apply -k deployment/
kubectl rollout status statefulset/nifi -n nifi --timeout=900s
```

A first start takes several minutes. `nifi-1` waits for `nifi-0` to report ready, and flow election
waits for both votes. [Deploying](deploying.md) covers what the logs say while that happens.

The UI is at `https://nifi/nifi` through the Ingress, with the single-user credentials from
`deployment/base/config/configmap-env.yml`. `kubectl port-forward` does not reach it, see
[why](ingress.md#why-port-forward-does-not-work).

The single user password, the sensitive properties key and the keystore password ship as
placeholders. Change them before production, see
[configuration](configuration.md#before-production).

## Documentation

| Page | Contents |
| --- | --- |
| [Architecture](architecture.md) | How the components fit together, with a diagram |
| [Deploying](deploying.md) | Cluster requirements, and what the first start looks like |
| [Ingress and access](ingress.md) | Deploy targets, supplying your own controller, why port-forward fails |
| [Configuration](configuration.md) | Every setting in the ConfigMap, and what to change before production |
| [Operations](operations.md) | Health checks, and what the common failures mean |
| [Layout](layout.md) | How `deployment/` is organised, image pinning, relocating the release |
| [Local testing with kind](local-testing.md) | Reproduce the CI cluster locally |

For a first deployment, [local testing with kind](local-testing.md) produces a working cluster in a
single page. Read [deploying](deploying.md) before deploying to a production cluster, particularly
the section on the shared certificate volume.

## Repository

Source code and releases: [github.com/saidsef/k8s-nifi-cluster](https://github.com/saidsef/k8s-nifi-cluster)

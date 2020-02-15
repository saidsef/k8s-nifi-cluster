# Kubernetes NiFi Cluster

[Apache NiFi](https://nifi.apache.org/) supports powerful and scalable directed graphs of data routing, transformation, and system mediation logic.

My goal is to show how to run Apache NiFi Cluster in Kubernetes

## Prerequisites

- Kubernetes Cluster
- Ingress Controller

## Deployments

This will deploy Apache NiFi in a Cluster mode with extenal Apache Zookeeper managing ellections:

```shell
kubectl apply -k deployments/
```

This will create:

- 1x NiFi Namespace (all the items will be deployed here)
- 3x Apache NiFi (each with it's own Service endpoint)
- 1x Apache Zookeeper (accessible within the cluster only)
- 1x Secrets (basic auth username/passowrd: `admin:admin`)
- 1x Ingress (access endpoint)

> Important: If this is exposed to public remeber to update the default `username/password`.

## Services

```shell
kubectl get all,ing --namespace nifi
```

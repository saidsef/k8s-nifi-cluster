# Kubernetes NiFi Cluster

[Apache NiFi](https://nifi.apache.org/) supports powerful and scalable directed graphs of data routing, transformation, and system mediation logic.

My goal is to show how to run Apache NiFi Cluster in Kubernetes

## Prerequisites

- Kubernetes Cluster >= v1.22

## Deployments

This will deploy Apache NiFi in a Cluster mode with extenal Apache Zookeeper managing ellections:

```shell
kubectl apply -k deployments/
```

> *NOTE:* Remember to update Ingress hostname

This will create:

- 1x NiFi Namespace (all the items will be deployed here)
- 3x Apache NiFi (each with it's own Service endpoint)
- 1x Apache Zookeeper (accessible within the cluster only)
- 1x Secrets (basic auth username/passowrd: `admin:admin`)
- 1x Ingress (access endpoint)

> :information_source: Set `Ingress` hostname to valid hostname before enabling it in `kustomization.yml` 
> Important: Remeber to update the default `username/password`.

## Services

```shell
kubectl get all,ing
```

## Contributing

Feel free to contribute by making a [pull request](https://github.com/saidsef/k8s-nifi-cluster/pulls).

Please read the official [Contribution Guide](./CONTRIBUTING.md) for more information on how you can contribute.

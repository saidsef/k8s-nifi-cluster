# Kubernetes NiFi Cluster

[Apache NiFi](https://nifi.apache.org/) supports powerful and scalable directed graphs of data routing, transformation, and system mediation logic.

This project demonstrates how to run an Apache NiFi 2.x Cluster in Kubernetes with secure defaults and Kubernetes-native coordination.

## Prerequisites

- Kubernetes Cluster >= v1.23
- `kubectl` with Kustomize support

## Deployments

This will deploy Apache NiFi 2.7.2 in a Cluster mode using [Kubernetes Leases](https://kubernetes.io/docs/concepts/architecture/leases/) for leader election and an external Apache Zookeeper for state management:

```shell
kubectl apply -k deployment/
```

### What gets created?

- **Namespace:** `nifi` (all resources are isolated here)
- **NiFi Cluster:** 2x Apache NiFi instances (StatefulSet)
- **Coordination:** Kubernetes-native leader election (RBAC included)
- **Zookeeper:** 1x instance for NiFi state management
- **Security:** Automatic self-signed certificate generation for HTTPS
- **Access:** Ingress and Service for external/internal access

## Accessing the UI

By default, NiFi 2.x requires HTTPS. You can access the UI using port-forwarding:

```shell
kubectl port-forward svc/nifi 8443:8443 -n nifi
```

Then open: [https://localhost:8443/nifi](https://localhost:8443/nifi)

### Default Credentials
- **Username:** `admin`
- **Password:** `Password123456`

> :warning: **Important:** Update the `NIFI_SINGLE_USER_CREDENTIALS_PASSWORD` in `deployment/nifi/configmap.yml` and the `nifi-basic-auth` secret before production use.

## Verification

```shell
kubectl get all,ing,leases -n nifi
```

## Contributing

We would :heart: you to contribute by making a [pull request](https://github.com/saidsef/k8s-nifi-cluster/pulls).

Please read the official [Contribution Guide](./CONTRIBUTING.md) for more information on how you can contribute.

## Useful links:

- [NiFi 2.0 Migration Guide](https://cwiki.apache.org/confluence/display/NIFI/NiFi+2.0+Migration+Guide)
- [NiFi System Administrator’s Guide](https://nifi.apache.org/docs/nifi-docs/html/administration-guide.html)
# Kubernetes NiFi Cluster

[Apache NiFi](https://nifi.apache.org/) supports powerful and scalable directed graphs of data routing, transformation, and system mediation logic.

This project demonstrates how to run an Apache NiFi 2.x Cluster in Kubernetes with secure defaults and Kubernetes-native coordination.

## Prerequisites

- Kubernetes Cluster >= v1.23
- `kubectl` with Kustomize support

## Deployments

This will deploy Apache NiFi 2.9.0 in a Cluster mode using [Kubernetes Leases](https://kubernetes.io/docs/concepts/architecture/leases/) for leader election and Kubernetes ConfigMaps for state management:

```shell
kubectl apply -k deployment/
```

### What gets created?

- **Namespace:** `nifi` (all resources are isolated here)
- **NiFi Cluster:** 2x Apache NiFi instances (StatefulSet)
- **Coordination:** Kubernetes-native leader election and ConfigMap state (no ZooKeeper)
- **Security:** Automatic self-signed certificate generation for HTTPS
- **Access:** Ingress and Service for external/internal access

## Accessing the UI

NiFi runs HTTPS-only. With an nginx Ingress controller, access it via the Ingress host:

```shell
# Add to /etc/hosts: 127.0.0.1 nifi
kubectl port-forward svc/nifi 8443:8443 -n nifi  # fallback without ingress
```

Then open: [http://nifi/nifi](http://nifi/nifi)

NiFi handles its own authentication — log in with the single-user credentials configured in `configmap.yml`.

### Default Credentials

- **Username:** `admin`
- **Password:** `Password123456`

> :warning: **Important:** Change `SINGLE_USER_CREDENTIALS_PASSWORD` and `NIFI_SENSITIVE_PROPS_KEY` in `deployment/nifi/configmap.yml` before production use.

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
# Kubernetes NiFi Cluster

[Apache NiFi](https://nifi.apache.org/) supports powerful and scalable directed graphs of data routing, transformation, and system mediation logic.

This project demonstrates how to run an Apache NiFi 2.x Cluster in Kubernetes with secure defaults and Kubernetes-native coordination.

## Prerequisites

- Kubernetes Cluster >= v1.23
- `kubectl` with Kustomize support
- An Ingress controller, see [Ingress and access](./docs/ingress.md)
- ReadWriteMany storage for the shared certificate volume, see [Deploying](./docs/deploying.md#the-shared-certificate-volume)

New to this? [Local testing with kind](./docs/local-testing.md) gets you a working cluster in one
page. Read [Deploying](./docs/deploying.md) before putting it on a real one.

## Deployments

This will deploy Apache NiFi 2.11.0 in a Cluster mode using [Kubernetes Leases](https://kubernetes.io/docs/concepts/architecture/leases/) for leader election and Kubernetes ConfigMaps for state management:

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

NiFi runs HTTPS-only and is reached through the Ingress. Point the host at your ingress
controller, for example by adding `127.0.0.1 nifi` to `/etc/hosts`, then open
[http://nifi/nifi](http://nifi/nifi).

NiFi handles its own authentication - log in with the single-user credentials configured in
`deployment/base/config/configmap-env.yml`.

### Default Credentials

- **Username:** `admin`
- **Password:** `Password123456`

> :warning: **Important:** Change `SINGLE_USER_CREDENTIALS_PASSWORD`, `NIFI_SENSITIVE_PROPS_KEY` and the keystore password before production use. See [Configuration](./docs/configuration.md#before-production).
>
> :warning: `kubectl port-forward` does not reach the UI. See [why](./docs/ingress.md#why-port-forward-does-not-work).

## Verification

```shell
kubectl get all,ing,leases -n nifi
```

## Documentation

Full index in [docs/](./docs/README.md).

- [Deploying](./docs/deploying.md) - what your cluster needs, and what the first start looks like
- [Ingress and access](./docs/ingress.md) - deploy targets, bringing your own controller, why port-forward fails
- [Configuration](./docs/configuration.md) - every ConfigMap setting, and what to change before production
- [Operations](./docs/operations.md) - health checks, and what the errors mean
- [Layout](./docs/layout.md) - how `deployment/` is organised, image pinning, relocating the release
- [Local testing with kind](./docs/local-testing.md) - reproduce the CI cluster locally

## Contributing

We would :heart: you to contribute by making a [pull request](https://github.com/saidsef/k8s-nifi-cluster/pulls).

Please read the official [Contribution Guide](./CONTRIBUTING.md) for more information on how you can contribute.

## Useful links

- [NiFi 2.0 Migration Guide](https://cwiki.apache.org/confluence/display/NIFI/NiFi+2.0+Migration+Guide)
- [NiFi System Administrator’s Guide](https://nifi.apache.org/docs/nifi-docs/html/administration-guide.html)

# Ingress and access

The Ingress is kept out of the base so you can bring your own controller.

| Command | Deploys |
| --- | --- |
| `kubectl apply -k deployment/` | base + nginx Ingress (default) |
| `kubectl apply -k deployment/nginx` | the same, stated explicitly |
| `kubectl apply -k deployment/base` | NiFi only - no Ingress |

## Bring your own controller

Add a folder next to `nginx/` that layers your own resource on top of the base. For example
`deployment/traefik/kustomization.yml`:

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: nifi

resources:
- ../base
- ingress.yml
```

Then place your Ingress, Gateway or `LoadBalancer` Service in `deployment/traefik/ingress.yml` and
apply with `kubectl apply -k deployment/traefik`.

Route traffic to the `nifi` Service on port `8443`. NiFi is bound to HTTPS and the certificate is
self-signed. Your controller needs the equivalent of the settings in
`deployment/nginx/ingress.yml`:

| Setting | Why |
| --- | --- |
| backend protocol HTTPS | NiFi is bound to HTTPS and has no plaintext port |
| SNI server name `nifi`, verification off | the certificate is self-signed |
| upstream host `nifi:8443` | must match an entry in `NIFI_WEB_PROXY_HOST` |
| sticky sessions | each node signs its own JWTs and rejects the other's |

Sticky sessions are not an optimisation. A token issued by one node is rejected by the other with
`Signed JWT rejected: Another algorithm expected, or no matching key(s) found`.

Two failures are worth knowing before you start. Neither error says what it means. A `host:port`
NiFi has not been told about gives `421 Invalid Port Requested`, and a hostname outside the
certificate's SAN gives `400 Invalid SNI`. Both are covered in
[operations](./operations.md#when-something-is-wrong), and the allowlist that drives the first is
`NIFI_WEB_PROXY_HOST` in [configuration](./configuration.md).

## Why port-forward does not work

`kubectl port-forward svc/nifi 8443:8443 -n nifi` fails with `connection refused`.

NiFi binds its HTTPS connector to `nifi.web.https.host`, which is set to the pod's FQDN. Nothing
listens on the loopback address port-forward connects to inside the pod. The same property is the
address a node advertises to the rest of the cluster. Widening it to `0.0.0.0` makes every node
announce itself as `localhost` and the cluster never forms.

Deploy an Ingress controller, or expose the `nifi` Service yourself and add that `host:port` to
`NIFI_WEB_PROXY_HOST` in `deployment/base/config/configmap-env.yml`.

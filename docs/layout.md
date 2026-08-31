# Layout

```
deployment/
├── kustomization.yml   # default entry point -> nginx
├── base/               # NiFi itself, no Ingress
│   ├── namespace/
│   ├── config/         # ConfigMaps: env, authorizers, state management, cert script
│   └── nifi/           # RBAC, storage, Service, StatefulSet, HPA
└── nginx/              # base + an nginx Ingress
```

Every folder carries its own `kustomization.yml`, so each can be built on its own:

```shell
kubectl kustomize deployment/base/config
```

`base/` is the whole of NiFi with no Ingress attached. `nginx/` layers an nginx Ingress on top of
it. `deployment/kustomization.yml` is a thin default that points at `nginx/`.

## Image tags

The NiFi and busybox tags are pinned in `deployment/base/nifi/kustomization.yml`:

```yaml
images:
- name: docker.io/apache/nifi
  newTag: "2.11.0"
- name: docker.io/busybox
  newTag: "1.37"
```

This overrides whatever tag is written in the manifests, so change the tag here rather than
editing `statefulset.yml`.

## Relocating the release

An overlay may change `namespace:`. The certificate SAN is built from the running namespace at
startup, so the pod FQDNs stay covered without further edits. If you also rename the `nifi`
Service, set `SERVICE_NAME` on the `security-setup` init container to match and update
`NIFI_WEB_PROXY_HOST` in `deployment/base/config/configmap-env.yml`.

## What is not persisted

The only PersistentVolumeClaim is for the shared certificates. The flow definition
(`conf/flow.json.gz`) and the flowfile, content and provenance repositories all live on the
container's writable layer. A rolling update is safe, because the surviving node holds the flow
and replicates it back, but losing both pods at once loses the flow.

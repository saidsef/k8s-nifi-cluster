# Layout

```text
deployment/
├── kustomization.yml   # default entry point -> nginx
├── base/               # NiFi itself, no Ingress
│   ├── namespace/
│   ├── config/         # ConfigMaps: env, authorizers, state management, cert script
│   └── nifi/           # RBAC, storage, Service, StatefulSet, HPA
└── nginx/              # base + an nginx Ingress
```

Each folder carries its own `kustomization.yml` and can be built independently:

```shell
kubectl kustomize deployment/base/config
```

`base/` contains NiFi with no Ingress. `nginx/` layers an nginx Ingress on top of it.
`deployment/kustomization.yml` is a default that points at `nginx/`.

## Image tags

The NiFi and busybox tags are pinned in `deployment/base/nifi/kustomization.yml`:

```yaml
images:
- name: docker.io/apache/nifi
  newTag: "2.11.0"
- name: docker.io/busybox
  newTag: "1.37"
```

This overrides whatever tag is written in the manifests. Change the tag here rather than in
`statefulset.yml`.

## Relocating the release

An overlay may change `namespace:`. The certificate SAN is built from the running namespace at
startup. The pod FQDNs stay covered without further edits. If you also rename the `nifi` Service,
set `SERVICE_NAME` on the `security-setup` init container to match and update `NIFI_WEB_PROXY_HOST`
in `deployment/base/config/configmap-env.yml`.

## What is not persisted

The only PersistentVolumeClaim is for the shared certificates. The flow definition
(`conf/flow.json.gz`) and the flowfile, content and provenance repositories all live on the
container's writable layer. A rolling update is safe. The surviving node holds the flow and
replicates it back. Losing both pods at once loses the flow.

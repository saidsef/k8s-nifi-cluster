# Local testing with kind

`kind-config.yml` builds the same two node cluster CI uses. It maps ports 80 and 443 on the
control-plane node, labels that node `ingress-ready=true`, and mounts `/tmp/nifi-certs` into both
nodes so the shared certificate volume is visible cluster-wide.

```shell
mkdir -p /tmp/nifi-certs
kind create cluster --name nifi-test --config kind-config.yml
```

## Ingress controller

The upstream ingress-nginx manifest carries no node selector. Pin it to the node holding the port
mappings. Without this it can land on the worker and `http://nifi/` will not answer. It already
tolerates the control-plane taint.

```shell
kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/deploy-ingress-nginx.yaml
kubectl -n ingress-nginx patch deployment ingress-nginx-controller \
  -p '{"spec":{"template":{"spec":{"nodeSelector":{"ingress-ready":"true"}}}}}'
kubectl -n ingress-nginx rollout status deployment/ingress-nginx-controller
```

## Deploy and check

```shell
kubectl apply -k deployment/
kubectl rollout status statefulset/nifi -n nifi --timeout=900s
```

`podManagementPolicy: OrderedReady` starts `nifi-1` only once `nifi-0` is ready, and flow election
needs both votes. First start takes a few minutes. Confirm the cluster formed:

```shell
curl -s -c /tmp/cj -b /tmp/cj -X POST -H "Host: nifi" \
  -d "username=admin&password=Password123456" \
  http://localhost/nifi-api/access/token > /tmp/token

curl -s -c /tmp/cj -b /tmp/cj -H "Host: nifi" \
  -H "Authorization: Bearer $(cat /tmp/token)" \
  http://localhost/nifi-api/flow/cluster/summary
```

Expect `"connectedNodes":"2 / 2"`. The cookie jar matters, see
[sticky sessions](./ingress.md#bring-your-own-controller).

## Autoscaling

The HPA needs `metrics-server`, which kind does not ship. Install it with `--kubelet-insecure-tls`
if you want to exercise scaling, otherwise the HPA reports `<unknown>` targets.

If it does not come up, [operations](./operations.md#when-something-is-wrong) lists what the
errors mean.

## Teardown

```shell
kind delete cluster --name nifi-test
```

## What CI does

CI validates the spec rather than running the workload. It builds the cluster from this same
`kind-config.yml`, runs `kubectl kustomize` over every kustomization directory, then applies the
manifests so the API server validates them. It does not install an ingress controller or
metrics-server, and does not wait for NiFi to be ready.

# Operations

## Is it healthy

```shell
kubectl get pods,leases,hpa -n nifi
```

Healthy means two pods `1/1`, and both leases held:

```text
lease.coordination.k8s.io/cluster-coordinator   nifi-1.nifi.nifi.svc.cluster.local:11443
lease.coordination.k8s.io/primary-node          nifi-1.nifi.nifi.svc.cluster.local:11443
```

The holder moves when pods restart, and either node may hold either lease. What matters is that
both exist and name a running pod. No `cluster-coordinator` lease means leader election through
the Kubernetes API is not working, and the Role in `deployment/base/nifi/role.yml` is the first
thing to check.

Kubernetes readiness is not the whole story. The probe is a TCP check on the cluster protocol
port, so a pod reports `1/1` once that port is listening, which happens before it has joined the
cluster. For NiFi's own view, ask NiFi:

```shell
curl -s -c /tmp/cj -b /tmp/cj -X POST -H "Host: nifi" \
  -d "username=admin&password=Password123456" \
  http://localhost/nifi-api/access/token > /tmp/token

curl -s -c /tmp/cj -b /tmp/cj -H "Host: nifi" \
  -H "Authorization: Bearer $(cat /tmp/token)" \
  http://localhost/nifi-api/flow/cluster/summary
```

```json
{"clusterSummary":{"connectedNodes":"2 / 2","connectedNodeCount":2,"totalNodeCount":2,"clustered":true,"connectedToCluster":true}}
```

The cookie jar is not optional, see the JWT entry below.

## When something is wrong

### `400 Invalid SNI`

The hostname you are calling is not in the certificate's SAN. The certificate covers `localhost`,
`nifi`, and the service and pod names in the namespace it was generated in. Reaching NiFi under
any other name needs that name in the SAN, which means regenerating the certificate, so delete
the keystore from the shared volume and restart the pods.

### `421 Invalid Port Requested`

The `host:port` in the Host header is not in `NIFI_WEB_PROXY_HOST`. This is the usual first
failure when putting your own Ingress controller in front. Either add the host and port to
`configmap-env.yml`, or have the controller rewrite the Host, which is what the nginx Ingress
here does with `upstream-vhost: nifi:8443`.

### `Signed JWT rejected: Another algorithm expected, or no matching key(s) found`

Your request landed on a different node from the one that issued the token. Each node signs its
own JWTs and does not accept another node's. The Ingress needs sticky sessions, and a client
calling the API directly needs to keep the session cookie. This is why the nginx Ingress sets
`affinity: cookie`.

### `connection refused` from `kubectl port-forward`

Expected. NiFi binds to the pod's FQDN, not the loopback address port-forward connects to. See
[ingress and access](./ingress.md#why-port-forward-does-not-work).

### HPA shows `<unknown>` targets

No metrics-server in the cluster. Install it, or ignore the HPA. Nothing else depends on it.

### Stuck on `Cluster is still voting on which Flow is the correct flow`

Normal for the first minute or two of a cold start, because election waits for
`NIFI_ELECTION_MAX_CANDIDATES` votes. If it does not clear, the second node never became ready.
Check whether `nifi-1` exists at all, since `OrderedReady` will not create it until `nifi-0` is
ready, and check `NIFI_ELECTION_MAX_CANDIDATES` against the replica count.

### Both pods run but the cluster never forms

Most likely the certificate volume is not actually shared, so each node generated its own
keystore and neither trusts the other. Compare the fingerprints, see
[deploying](./deploying.md#the-shared-certificate-volume).

### Pods stay `Pending`

Each pod requests 500m CPU and 2Gi of memory. Check what the node can offer with
`kubectl describe pod -n nifi`.

## Logs

```shell
kubectl logs -n nifi nifi-0 -c nifi
kubectl logs -n nifi nifi-0 -c security-setup    # certificate generation
```

A healthy start logs no `ERROR` lines:

```shell
kubectl logs -n nifi nifi-0 -c nifi | grep -c ' ERROR '
```

## Restarting

A rolling restart is safe. The surviving node holds the flow and replicates it back to the one
that comes up.

```shell
kubectl rollout restart statefulset/nifi -n nifi
```

Losing both pods at once is not safe. The flow is not persisted, see
[layout](./layout.md#what-is-not-persisted).

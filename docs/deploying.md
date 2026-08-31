# Deploying

## What your cluster needs

| Requirement | Detail |
| --- | --- |
| Kubernetes | v1.23 or later |
| Ingress controller | the only route to the UI, see [ingress and access](./ingress.md) |
| ReadWriteMany storage | for the shared certificate volume, see below |
| Capacity | 2 pods requesting 500m CPU and 2Gi each, with limits of 2 CPU and 4Gi |
| metrics-server | only if you want the HorizontalPodAutoscaler to do anything |

Without metrics-server the HPA sits at `<unknown>` targets and never scales. Nothing else breaks.

## The shared certificate volume

Read this before deploying to a real cluster.

Both pods serve the same certificate. The `security-setup` init container generates a keystore
into the shared volume on first start, exports the certificate, imports it into a truststore, and
every later pod finds the keystore already there and reuses it. The truststore holds exactly one
entry, that one certificate, which is how the nodes authenticate each other on the cluster
protocol port.

`persistentvolume-certs.yml` backs that volume with a `hostPath` at `/data/nifi-certs`. hostPath
is node-local. Put two pods on two nodes and they see two different directories, each generates
its own keystore, and neither trusts the other's certificate, so the cluster protocol handshake
fails and the cluster never forms.

The kind config in this repo works around it by mounting one host directory into every node. That
is fine for testing and is not a pattern to copy onto a real cluster. There, either:

- point the claim at a real ReadWriteMany class - NFS, EFS, Azure Files, CephFS - and delete
  `persistentvolume-certs.yml`, letting the class provision the volume, or
- pin the pods to a single node with a node selector, accepting that you lose the second node.

You can confirm the volume is genuinely shared by comparing the certificate each pod loaded:

```shell
for p in nifi-0 nifi-1; do
  kubectl exec -n nifi "$p" -c nifi -- keytool -list -v \
    -keystore /opt/nifi/nifi-current/conf/certs/keystore.p12 \
    -storepass th1s1s3up34e5r37 -storetype pkcs12 | grep -m1 SHA256
done
```

Two different fingerprints mean the volume is not shared and the cluster will not form.

## Deploy

```shell
kubectl apply -k deployment/
kubectl rollout status statefulset/nifi -n nifi --timeout=900s
```

See [ingress and access](./ingress.md) for the other deploy targets, and
[configuration](./configuration.md) for what to change first.

## What the first start looks like

Allow a few minutes. `podManagementPolicy: OrderedReady` holds `nifi-1` back until `nifi-0` is
ready, and `nifi-0` has a 60 second readiness delay before its first probe, so the second node
does not even begin until well over a minute in. NiFi itself starts in around 30 seconds once the
image is pulled.

Flow election then needs both votes, because `NIFI_ELECTION_MAX_CANDIDATES` is 2. Until the second
node casts its vote the first one logs this on a loop, and it is not a fault:

```
Requested by cluster coordinator to retry connection in 5 seconds with explanation:
Cluster is still voting on which Flow is the correct flow for the cluster.
```

`There is currently no Cluster Coordinator` in the first few seconds is also normal. The node
that logs it is about to become the coordinator.

## Confirm it worked

```shell
kubectl get pods,leases,hpa -n nifi
```

Two pods `1/1`, and two leases, `cluster-coordinator` and `primary-node`, both held. See
[operations](./operations.md) for checking the cluster from NiFi's own API, and for what the
errors mean when it does not come up.

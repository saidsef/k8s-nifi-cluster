# Architecture

Two NiFi nodes run as a StatefulSet in the `nifi` namespace. They coordinate through the
Kubernetes API rather than ZooKeeper, share one certificate, and are reached through an Ingress.

![Architecture of the NiFi cluster on Kubernetes](./architecture.svg)

## Getting in

The Ingress is the only route to the UI. NiFi binds its HTTPS connector to the pod's own fully
qualified name rather than to all addresses, so nothing is listening on the loopback address
`kubectl port-forward` connects to. See [ingress and access](./ingress.md).

The nginx Ingress carries three settings, and another controller needs their equivalents. The
backend protocol is HTTPS, because NiFi serves no plaintext port, and certificate verification is
off because the certificate is self-signed. The Host header is rewritten to `nifi:8443`, because
NiFi rejects any host and port outside `NIFI_WEB_PROXY_HOST` with `421 Invalid Port Requested`,
and rewriting means one allowlist entry covers every name you reach the cluster under. Sessions
are pinned to a node by cookie, because each node signs its own JWTs and rejects the other's.

## Forming a cluster

`KubernetesLeaderElectionManager` elects two roles through Leases. The cluster coordinator admits
nodes and holds the authoritative view of membership. The primary node runs isolated processors.
Either pod can hold either lease, and the holder changes when pods restart.

Nodes then talk to each other directly rather than through the Service. Port 11443 carries the
cluster protocol under mutual TLS, and 6342 carries flowfiles between nodes.

Flow election decides whose flow becomes the cluster's. It waits for two votes, matching the
replica count, or gives up after `NIFI_ELECTION_MAX_WAIT`. Until the second node votes, the first
retries every five seconds and logs `Cluster is still voting on which Flow is the correct flow`.

State that has to be visible across the cluster is written to ConfigMaps by
`KubernetesConfigMapStateProvider`. Local state stays on the container's disk.

## Certificates

Every node presents the same certificate. The `security-setup` init container generates a keystore
in the shared volume on first start, exports the certificate, and imports it into a truststore
holding that one entry. Later pods find the keystore already there and reuse it. Because all nodes
hold the same key and trust only that certificate, the cluster protocol handshake succeeds and
nothing else is accepted.

The certificate covers `localhost`, `127.0.0.1`, `nifi`, and the service and pod names in its
namespace. Any other hostname gives `400 Invalid SNI`.

The volume therefore has to be genuinely ReadWriteMany. Two pods on two nodes backed by node-local
storage each generate their own keystore, neither trusts the other, and the cluster never forms.
See [deploying](./deploying.md#the-shared-certificate-volume).

Users are separate and much simpler. One username and password come from the ConfigMap, authorised
by `SingleUserAuthorizer`, and the main container deletes `users.xml` and `authorizations.xml` on
every start so those credentials always apply.

## What is kept

The certificates are on the shared volume and cluster state is in ConfigMaps, so both outlive the
pods. Everything else sits on the container's writable layer: the flow, the flowfile, content and
provenance repositories, and local state. A rolling restart is safe because the surviving node
replicates the flow back to the one that comes up. Losing both pods at once loses the flow. See
[layout](./layout.md#what-is-not-persisted).

## Starting up

`fix-permissions` takes ownership of the certificate directory as root. `security-setup` then
generates the keystore or finds it already there. The main container rewrites `nifi.properties`
for the shared certificate paths, disables the SNI host check, clears the user files, and starts
NiFi.

`nifi-0` reports ready once port 11443 is listening, about a minute in, and `OrderedReady` holds
`nifi-1` back until then, so a first start takes several minutes. Readiness only checks that the
port is open, so a pod reads `1/1` before it has joined the cluster. For membership, ask NiFi, see
[operations](./operations.md#is-it-healthy).

The single user password, the sensitive properties key and the keystore password all ship as
placeholders. Change them before this goes anywhere real, see
[configuration](./configuration.md#before-production).

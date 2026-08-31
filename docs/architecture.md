# Architecture

Two NiFi nodes run as a StatefulSet in the `nifi` namespace. They coordinate through the
Kubernetes API rather than ZooKeeper, share one certificate, and are reached through an Ingress.

![Architecture of the NiFi cluster on Kubernetes](./architecture.svg)

## Ports

| Port | Carries | Between |
| --- | --- | --- |
| 8443 | UI and API over HTTPS | client and node |
| 11443 | cluster protocol under mutual TLS | node and node |
| 6342 | load balanced flowfiles | node and node |

## Reaching the UI

The Ingress is the only route in. NiFi binds its HTTPS connector to the pod's fully qualified
name, not to all addresses. Nothing listens on the loopback address `kubectl port-forward`
connects to. See [ingress and access](./ingress.md).

The nginx Ingress carries four settings. Another controller needs their equivalents.

| Setting | What it reflects |
| --- | --- |
| backend protocol HTTPS | NiFi is bound to HTTPS and has no plaintext port |
| certificate verification off | the certificate is self-signed |
| Host rewritten to `nifi:8443` | `NIFI_WEB_PROXY_HOST` allows that one value, anything else gives `421 Invalid Port Requested` |
| sticky sessions by cookie | each node signs its own JWTs and rejects the other's |

Rewriting the Host keeps the allowlist to a single entry whatever name you reach the cluster
under.

## Forming a cluster

`KubernetesLeaderElectionManager` elects two roles through Leases. The cluster coordinator admits
nodes and holds the authoritative view of membership. The primary node runs isolated processors.
Either pod can hold either lease, and the holder changes when pods restart.

Nodes talk to each other directly, not through the Service.

Flow election decides whose flow becomes the cluster's. It waits for two votes, matching the
replica count, or gives up after `NIFI_ELECTION_MAX_WAIT`. The first node retries every five
seconds until the second votes, logging `Cluster is still voting on which Flow is the correct
flow`.

`KubernetesConfigMapStateProvider` writes cluster-wide state to ConfigMaps. Local state stays on
the container's disk.

## Certificates

Every node presents the same certificate. The `security-setup` init container generates a keystore
in the shared volume on first start, exports the certificate, and imports it into a truststore
holding that one entry. Later pods reuse the keystore they find. All nodes hold the same key and
trust that one certificate. The cluster protocol handshake succeeds between them and fails for
anything else.

The certificate's SAN covers:

- `localhost` and `127.0.0.1`
- `nifi`, the Service name
- `nifi.<namespace>.svc.cluster.local`, the Service FQDN
- `*.nifi.<namespace>.svc.cluster.local`, every pod

Any other hostname gives `400 Invalid SNI`.

The volume has to be ReadWriteMany. Two pods on two nodes backed by node-local storage each
generate their own keystore. Neither trusts the other and the cluster never forms. See
[deploying](./deploying.md#the-shared-certificate-volume).

User authentication is separate. One username and password come from the ConfigMap, authorised by
`SingleUserAuthorizer`. The main container deletes `users.xml` and `authorizations.xml` on every
start. Those credentials always apply.

## What is kept

| Data | Where it lives | Survives losing the pod |
| --- | --- | --- |
| Certificates | shared PVC | yes |
| Cluster state | ConfigMaps | yes |
| Flow | container writable layer | only through the surviving node |
| Flowfile, content and provenance repositories | container writable layer | no |
| Local state | container writable layer | no |

A rolling restart is safe. The surviving node replicates the flow back to the one that comes up.
Losing both pods at once loses the flow. See [layout](./layout.md#what-is-not-persisted).

## Starting up

1. `fix-permissions` takes ownership of the certificate directory as root.
2. `security-setup` generates the keystore, or finds one and leaves it.
3. The main container rewrites `nifi.properties` for the shared certificate paths, disables the
   SNI host check and clears the user files.
4. `start.sh` brings NiFi up.

`nifi-0` reports ready once port 11443 is listening, about a minute in. `OrderedReady` holds
`nifi-1` back until then, and a first start takes several minutes.

Readiness only checks that the port is open. A pod reads `1/1` before it has joined the cluster.
Ask NiFi for membership, see [operations](./operations.md#is-it-healthy).

The single user password, the sensitive properties key and the keystore password ship as
placeholders. Change them before production, see
[configuration](./configuration.md#before-production).

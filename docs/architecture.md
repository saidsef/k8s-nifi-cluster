# Architecture

Two NiFi nodes in a StatefulSet, coordinating through the Kubernetes API rather than ZooKeeper,
sharing one certificate and reached through an Ingress.

![Architecture of the NiFi cluster on Kubernetes](./architecture.svg)

## What runs

| Resource | Name | Purpose |
| --- | --- | --- |
| StatefulSet | `nifi` | 2 NiFi nodes, `OrderedReady`, stable pod names and DNS |
| Service | `nifi` | ClusterIP on 8443, 11443 and 6342, and the governing service that gives each pod a stable name |
| Ingress | `nifi` | the only route to the UI, terminates on the nginx controller |
| PersistentVolumeClaim | `nifi-shared-certs` | ReadWriteMany, holds the keystore every node loads |
| ConfigMap | `nifi-cm` | environment read by the image's `start.sh` |
| ConfigMap | `nifi-ssl-cm` | `security.sh`, the certificate generator |
| ConfigMap | `nifi-state-mgmt-cm` | local and cluster state providers |
| ConfigMap | `nifi-authorizers-cm` | single user authorizer |
| ServiceAccount, Role, RoleBinding | `nifi`, `nifi-role`, `nifi-rolebinding` | leases and ConfigMaps in the namespace |
| HorizontalPodAutoscaler | `nifi` | 2 to 8 replicas on CPU and memory |

## The request path

A request goes browser, ingress controller, Service, pod. There is no other way in, because NiFi
binds its HTTPS connector to the pod's own FQDN rather than to all addresses, which is also why
`kubectl port-forward` cannot reach it. See [ingress and access](./ingress.md).

Three properties of that path are load bearing:

- **The backend speaks HTTPS.** NiFi serves no plaintext port, so the controller has to connect
  over TLS, with verification off because the certificate is self-signed.
- **The Host header is rewritten to `nifi:8443`.** NiFi rejects any `host:port` that is not in
  `NIFI_WEB_PROXY_HOST` with `421 Invalid Port Requested`. The nginx `upstream-vhost` annotation
  makes one allowlist entry cover every hostname you reach the cluster under.
- **Sessions are sticky.** Each node signs its own JWTs and rejects tokens signed by the other,
  so a session that moves between nodes fails with `Signed JWT rejected`. The `affinity: cookie`
  annotation pins it.

The three settings are per-controller. Putting a different controller in front means reproducing
all three, covered in [ingress and access](./ingress.md#bring-your-own-controller).

## How the cluster forms

Coordination runs through the Kubernetes API, so there is no ZooKeeper to deploy or operate.

`KubernetesLeaderElectionManager` elects both roles through Leases. `cluster-coordinator` is the
node that admits others and holds the authoritative view of membership, `primary-node` is the one
that runs isolated processors. Either pod may hold either lease, and the holder moves on restart.
The Role granting access to those leases is what makes this work, so an absent
`cluster-coordinator` lease points at RBAC first.

Nodes then talk to each other directly. Port 11443 carries the cluster protocol under mutual TLS,
authenticated by the shared certificate, and 6342 carries load balanced flowfiles between nodes.
Neither goes through the Service.

Flow election decides which node's flow becomes the cluster's flow. It waits for
`NIFI_ELECTION_MAX_CANDIDATES` votes, set to 2 to match the replica count, or gives up after
`NIFI_ELECTION_MAX_WAIT`. Until both votes are in, the first node logs `Cluster is still voting on
which Flow is the correct flow` on a loop, which is expected rather than a fault.

Component state that has to be visible cluster-wide is written to ConfigMaps by
`KubernetesConfigMapStateProvider`, one per component. Local state stays on the container's disk.

## Identity and trust

Every node presents the same certificate, and that is the whole trust model.

The `security-setup` init container runs `security.sh` against the shared volume. The first node
to start generates a keystore, exports the certificate, and imports it into a truststore holding
exactly that one entry. Every later pod finds the keystore already there and reuses it. Because
all nodes hold the same key and the same single-entry truststore, the cluster protocol handshake
succeeds and nothing else is trusted.

The certificate's SAN covers `localhost`, `127.0.0.1`, `nifi`, the service FQDN and a wildcard for
pod FQDNs in the namespace it was generated in. A hostname outside that set gives
`400 Invalid SNI`.

This is why the volume has to be genuinely ReadWriteMany. Two pods on two nodes with a node-local
volume each generate their own keystore, neither trusts the other, and the cluster never forms.
The `hostPath` PersistentVolume shipped here works only because the kind config mounts one host
directory into every node, and it is the first thing to replace on a real cluster. See
[deploying](./deploying.md#the-shared-certificate-volume).

User authentication is separate and much simpler: one user, defined by
`SINGLE_USER_CREDENTIALS_USERNAME` and `SINGLE_USER_CREDENTIALS_PASSWORD`, authorised by
`SingleUserAuthorizer`. The main container deletes `users.xml` and `authorizations.xml` on every
start so those credentials are always the ones in force.

## What survives a restart

| Data | Where | Survives a pod restart |
| --- | --- | --- |
| Certificates | shared PVC | yes |
| Cluster component state | ConfigMaps | yes |
| Flow definition | container writable layer | only via the surviving node |
| Flowfile, content, provenance repositories | container writable layer | no |
| Local state | container writable layer | no |

A rolling restart is safe because the surviving node holds the flow and replicates it back to the
node that comes up. Losing both pods at once loses the flow. See
[layout](./layout.md#what-is-not-persisted).

## Start-up order

1. `fix-permissions` takes ownership of the shared certificate directory as root, then drops out.
2. `security-setup` generates the keystore and truststore, or finds them already present.
3. The main container rewrites `nifi.properties` to point at the shared certificates, disables the
   SNI host check, clears the user files, then hands over to the image's `start.sh`.
4. `nifi-0` becomes ready once port 11443 is listening, roughly 60 seconds in because of the
   readiness delay. `OrderedReady` holds `nifi-1` back until then.
5. `nifi-1` starts, joins, and casts the second vote. Flow election ends and both nodes connect.

Readiness is a TCP check on the cluster protocol port, so a pod reports `1/1` before it has
finished joining. For NiFi's own view of membership, ask the API, see
[operations](./operations.md#is-it-healthy).

## Design decisions

| Decision | Why | What it costs |
| --- | --- | --- |
| Kubernetes leases instead of ZooKeeper | nothing extra to deploy, and leader election is visible with `kubectl get leases` | needs RBAC on `coordination.k8s.io` |
| One shared certificate for every node | node to node mutual TLS works with a single-entry truststore | needs ReadWriteMany storage, and a new node cannot be added without the volume |
| HTTPS only, no plaintext port | credentials and tokens never cross the network in clear | port-forward does not work, an Ingress controller is mandatory |
| `OrderedReady` pod management | the second node joins a cluster that already has a coordinator | first start takes minutes rather than seconds |
| Single user authentication | usable out of the box with no identity provider | one shared login, not suitable beyond evaluation |
| Repositories on the writable layer | no per-pod storage to provision | losing both pods at once loses the flow |

The single user credentials, sensitive properties key and keystore password all ship as
placeholders in a public repository. Change them before this goes anywhere real, see
[configuration](./configuration.md#before-production).

# Configuration

Everything tunable lives in `deployment/base/config/`. The environment variables in
`configmap-env.yml` are read by the NiFi image's `start.sh`, which writes them into
`nifi.properties` at container start.

## Before production

Three values ship as placeholders and all of them need changing.

| Setting | Where | Why |
| --- | --- | --- |
| `SINGLE_USER_CREDENTIALS_PASSWORD` | `configmap-env.yml` | the login for the UI |
| `NIFI_SENSITIVE_PROPS_KEY` | `configmap-env.yml` | encrypts sensitive processor properties inside the flow |
| keystore password | `configmap-security.yml` and `statefulset.yml` | protects the private key |

The keystore password is written out in two files, twice in `configmap-security.yml` as
`KEYSTORE_PASS` and `TRUSTSTORE_PASS`, and three times in the `sed` block in `statefulset.yml`
that points NiFi at the certificates. Change all five or NiFi will not open its own keystore.

Change `NIFI_SENSITIVE_PROPS_KEY` before you build a flow, not after. It encrypts sensitive
properties stored in the flow. Changing it later leaves NiFi unable to decrypt them.

None of these are Secrets. They are ConfigMap values in a public repo. Treat this as a starting
point and move them into whatever secret management you run.

## Settings

| Variable | Effect |
| --- | --- |
| `NIFI_JVM_HEAP_INIT`, `NIFI_JVM_HEAP_MAX` | JVM heap. Must fit inside the container memory limit with room for non-heap usage |
| `NIFI_WEB_HTTPS_PORT` | the port NiFi serves the UI and API on |
| `NIFI_WEB_PROXY_HOST` | comma separated allowlist of `host:port` values NiFi will accept in a Host header |
| `NIFI_SENSITIVE_PROPS_KEY` | key for encrypting sensitive properties in the flow |
| `NIFI_CLUSTER_IS_NODE` | joins the cluster rather than running standalone |
| `NIFI_CLUSTER_NODE_PROTOCOL_PORT` | node to node protocol port, mutual TLS |
| `NIFI_CLUSTER_NODE_PROTOCOL_MAX_THREADS` | thread pool for that protocol |
| `NIFI_CLUSTER_LEADER_ELECTION_IMPLEMENTATION` | `KubernetesLeaderElectionManager` elects through Leases, no ZooKeeper |
| `NIFI_STATE_MANAGEMENT_PROVIDER_CLUSTER` | must match an `<id>` in `configmap-state-management.yml` |
| `NIFI_ELECTION_MAX_WAIT` | how long flow election waits before deciding without a full set of votes |
| `NIFI_ELECTION_MAX_CANDIDATES` | votes needed to end flow election early |
| `SINGLE_USER_CREDENTIALS_USERNAME`, `SINGLE_USER_CREDENTIALS_PASSWORD` | the single user NiFi authenticates |

## Settings that have to agree

`NIFI_WEB_PROXY_HOST` has to contain whatever Host header reaches NiFi. The nginx Ingress
rewrites it to `nifi:8443` with `upstream-vhost`, which is why that one value is enough here. Put
a different controller in front and you have to add its host and port, or you get
`421 Invalid Port Requested`. See [operations](./operations.md).

`NIFI_ELECTION_MAX_CANDIDATES` should match the StatefulSet replica count. Set it higher than the
number of nodes that will start and every deploy waits the full `NIFI_ELECTION_MAX_WAIT` before
forming.

`NIFI_JVM_HEAP_MAX` is 2g against a 4Gi container limit. Raise the heap and raise the limit with
it, or the JVM gets killed by the kernel rather than throwing `OutOfMemoryError`.

## Scaling

`replicas` in `statefulset.yml` and `minReplicas` in `hpa.yml` are both 2. Change one and change
the other, or the HPA will immediately scale the StatefulSet back to its own minimum.

Raising the replica count means raising `NIFI_ELECTION_MAX_CANDIDATES` to match, and it means the
certificate volume has to be shared across every node the pods can land on. See
[deploying](./deploying.md#the-shared-certificate-volume).

## Other files

- `configmap-authorizers.yml` - single user authorizer. Swap this for a real authorizer if you move off single user auth
- `configmap-state-management.yml` - local state on disk, cluster state in ConfigMaps
- `configmap-security.yml` - the certificate generation script, covered in [deploying](./deploying.md#the-shared-certificate-volume)

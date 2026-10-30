# How it works

This page follows one run of `omerfsen.stratum.cluster` from an empty set of
hosts to a highly available cluster, and says why each step is done the way
it is. Behaviour of Stratum itself is taken from its source
(github.com/ctrliq/stratum, `pkg/cli`, `internal/`) and documentation.

## Topology

```text
                         VIP 10.0.10.10:6443  (kube-vip, ARP)
                                   │
          ┌────────────────────────┼────────────────────────┐
          │                        │                        │
   stratum_control_plane[0]   control plane 2          control plane 3
   bootstraps the cluster     joins through VIP        joins through VIP
   etcd member 1              etcd member 2            etcd member 3
          │                        │                        │
          └──────────── Cilium VXLAN (8472/udp) ────────────┘
                       │                         │
                   worker 1                  worker N   (optional)
```

- Control planes: 1, 3 or 5. etcd needs a majority, so two members tolerate
  no failure; the role refuses an even number.
- More than one control plane needs `stratum_vip`, a free address on the node
  subnet. kube-vip announces it by ARP from the controller that holds it, so
  all controllers must share one L2 segment.
- Workers: 0 or more. With none, the control planes run workloads (Stratum's
  default). `stratum_control_plane_taint: true` keeps workloads off them.

## 1. Prepare (all nodes at once)

`tasks_from: prepare`:

1. Platform: RHEL family 9 or later, x86_64 or aarch64, cgroup v2 only.
2. Packages from the distribution's own repositories: `container-selinux`,
   `policycoreutils-python-utils`, `libselinux-utils` (Stratum refuses a new
   node under enforcing SELinux without them), and `iproute-tc`,
   `iptables-nft`, `bpftool` (used by `stratum uninstall`).
3. SELinux must be enforcing unless `stratum_require_selinux_enforcing: false`.
4. firewalld, as in CIQ's tested configuration: a zone bound to the node
   subnet with 6443, 10250, 2379-2380, 9448/tcp, 8472/udp, 4240/tcp, ICMP
   echo, SSH and the dashboard NodePort; the pod range in `trusted`. Every rule
   is permanent and immediate. The only reload loads a newly created zone,
   before Cilium has rules: on the iptables backend a reload flushes them.
5. The binary: the release's sha256 is always checked; with
   `stratum_verify_signature: true` the keyless cosign signature is checked
   against the upstream release workflow at that tag. By default it is
   downloaded once onto the controller and pushed, so nodes need no internet
   access. `stratum_binary_local_path` takes a file you copied in yourself.

Stratum then does the rest of the host setup on its own at `stratum up`:
`overlay` and `br_netfilter`, swap off, forwarding sysctls, inotify limits.

## 2. `/etc/stratum/config.yaml`

```yaml
kind: ClusterConfig
spec:
  network:
    nodeIP: 10.0.10.11
    podCIDR: 10.245.0.0/16       # control planes only
    serviceCIDR: 10.96.0.0/12    # control planes only
    clusterDomain: cluster.local # control planes only
  vip:
    address: 10.0.10.10          # control planes only
  selinux: true
```

The file is parsed strictly: an unknown key is an error. A worker reads only
`nodeIP` and `selinux` and refuses anything else, so the role writes those two
keys only on workers. The network block is fixed at the cluster's first boot:
a joining control plane must match it.

## 3. The first control plane

`stratum up` never returns: it stays in the foreground and runs the node.
After a successful bootstrap or join it installs and enables
`stratum.service`, but does not start it. The role therefore starts it
detached, as upstream's test harness does, and hands it to systemd as the
quick start describes:

1. `systemd-run --unit=stratum-first-up … stratum up`: a transient unit with
   the same `KillMode` and `Delegate` as `stratum.service`, output to
   `/var/log/stratum-first-up.log`.
2. Wait until the node can be handed over, or fail with the last 40 log
   lines (and, for a worker, the operator's reason for a pending certificate):
   - control plane: Stratum logged `node <name> registered` and the local
     etcd has a write-ahead log; a control plane restarted without it refuses
     to start;
   - worker: the kubelet has its client certificate
     (`/var/lib/stratum/pki/kubelet/kubelet-client-current.pem`).
     Stopping it earlier makes the next kubelet file a second bootstrap CSR,
     and Stratum's operator then approves neither ("another bootstrap CSR
     claims node").
3. SIGTERM the foreground process (it exits 0 and keeps the node's state),
   then `systemctl start stratum`. From here on systemd runs
   `stratum up --config /etc/stratum/config.yaml`, also after a reboot.
4. Wait for `/healthz` through the VIP and for this node to be Ready.

## 4. The other control planes, one at a time

For each, on the first control plane:

```shell
stratum token create --control-plane --ttl 45m
```

prints one join line, with the certificate key written to a 0600 file whose
path is in that line:

```text
  stratum up --join https://10.0.10.10:6443 --token abcdef.0123456789abcdef --discovery-token-ca-cert-hash sha256:… --certificate-key-file '/tmp/stratum-ha-certificate-key-…'
```

The role then:

- writes the token and the certificate key to `/root/.stratum-join/` (0700) on
  the joining node and runs the line with `--token-file`, so neither shows up
  in `ps` or in Ansible's output (`no_log` on every task that holds them);
- starts the join with the same detached sequence as above;
- waits for `/healthz` and for this node to be a **started** etcd member
  before the next control plane begins (overlapping joins lose quorum);
- deletes its local copies of the token and key once the start is done;
- once the node is registered and Ready, or when anything fails: deletes the
  certificate key on the first control plane and revokes the token by
  deleting its `bootstrap-token-<id>` Secret. Not earlier: a worker's kubelet
  still needs the token for its TLS bootstrap after the kubelet has started.

## 5. Workers, all together

Same as above with `stratum token create --ttl 45m` and `stratum up --worker
--server https://<vip>:6443 …`. Extra flags in the printed line
(`--pod-cidr`, `--service-cidr`, `--cluster-domain`, `--profile` for a
non-default cluster) are kept as printed. Workers join in parallel; each mints
its own token.

## 6. Idempotency

A node is a member once Stratum's saved state (`/var/lib/stratum/config/node.yaml`)
exists and the node can start from it: a worker also needs the client
certificate of its TLS bootstrap (`kubelet-client-current.pem`), because Stratum
saves a worker's state before that bootstrap; a control plane needs its local
etcd write-ahead log. A join that failed before the state was saved is
repeated with a fresh token on the next run; one that failed after it is
reported, with how to start that node clean. On a member
the role never runs `stratum up` again; it only makes sure `stratum.service`
is running. A foreground `stratum up` left behind by an interrupted run is
waited for (until it has joined or exited) and then handed over to systemd;
a new one is never started beside it.

Bumping `stratum_version` on an existing cluster fails the run instead of
swapping the binary under running nodes: upgrade with `stratum upgrade`, or
set `stratum_allow_binary_replace: true` if you mean it. A second run of the playbook reports `changed=0`.

Changing `/etc/stratum/config.yaml` on a running node is reported, not
applied: it takes effect on the next `systemctl restart stratum`, and the
network block cannot change at all after the first boot.

## 7. Kubeconfig

From the first control plane's `/var/lib/stratum/kubeconfig/admin.conf` the
role builds an admin kubeconfig pointed at the VIP, with its cluster, user and
context all named `stratum_cluster_name`, and writes it to
`~/.kube/<stratum_cluster_name>.yaml` (0600) on the controller.

With `stratum_kubeconfig_merge: true` (off by default) it is also merged into
`~/.kube/config`: entries with that name are replaced, every other cluster,
user and context is kept, and a backup (`config.<timestamp>~`) is taken
before each change. The merged context then becomes the current one
(`stratum_kubeconfig_set_current`, default true). Give
each cluster its own `stratum_cluster_name`. All of this is YAML processing
in Ansible: the controller needs no kubectl.

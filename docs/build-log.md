# Build log

Every step, command, failure and fix, written as it happens. Dead ends stay in.

## 2026-10-29: first cluster on KVM

### Goal

At first: a CIQ Stratum cluster (one control plane, two workers) on Rocky
Linux 10.2 VMs on one KVM host, with Ansible. It grew the same evening into an
Ansible collection for Galaxy, `omerfsen.stratum`, with three control planes
behind a VIP, optional workers, and KVM as the first provisioning provider.
There was no Stratum collection or role on Galaxy yet.

### Where Stratum comes from

The docs (docs.ciq.com/stratum/installation) say the artifact is "one file"
and do not say where to get it; the releases page still said "Stratum has not
cut its first tagged release". The GitHub repo `ctrliq/stratum` is public and
had three releases tagged two days earlier, one per Kubernetes minor:

```shell
gh release list -R ctrliq/stratum
# v1.37.1-stratum.0   Latest   2026-10-27
# v1.36.5-stratum.0            2026-10-27
# v1.35.9-stratum.0            2026-10-27
```

Picked `v1.37.1-stratum.0` (Kubernetes 1.37.1), asset `stratum-linux-amd64`
(704 MB), with its sha256 from the release's `sha256sums.txt`. The release also
ships cosign `.sig`/`.pem` files; checksum only for now.

### Prerequisites from the docs

- Linux with cgroup v2, root, amd64 or arm64. Tested OS: Rocky 9.8. This repo
  uses Rocky 10.2, which the docs do not mention.
- Packages: `iproute-tc`, `iptables` (for `iptables-save`), `bpftool`, and on
  SELinux hosts `container-selinux`, `policycoreutils-python-utils`,
  `libselinux-utils`. A new node is refused under enforcing SELinux without them.
- Stratum itself loads `overlay`/`br_netfilter`, turns swap off, writes its
  sysctls and inotify limits. Nothing to do for those.
- Stratum does not open firewall ports. The GenericCloud image has no
  firewalld; the collection installs it and configures it the way CIQ tests it.

### The host

One KVM host (Ubuntu 22.04, 32 cores, 125 GB RAM) already running other VMs.
No LAN bridge: guests use libvirt's `default` network, 192.168.122.0/24, in
`route` mode, with the host doing NAT. Its DHCP range is .10-.29, so the
cluster takes static addresses .80 (control plane) and .81+ (workers). Checked
first that nothing answered on .80-.89.

Ansible was not installed on the host: a venv in the repo with ansible-core
2.17.14 (the newest that runs on the host's Python 3.10), collections pinned in
`ansible/requirements.yml`.

### VMs

The VM role comes from the runai-oss repo (Rocky 10.2 + RKE2 on KVM), minus the
bridge and GPU passthrough: cloud-init seed with a static IP matched by MAC,
UEFI, `--cpu host-passthrough` (Rocky 10 needs x86-64-v3). `virt-install` here
has no `rocky10` osinfo entry; `rhel10.1` is the same platform.

The Rocky 10.2 GenericCloud qcow2 (545 MB) is downloaded once, checksum
verified, into `/var/lib/libvirt/images`. The download took about 8 minutes
from this host. Every VM disk is a copy of it.

Workers are optional: `stratum_worker_count` (0 = single node). The first
version listed the nodes by hand; the node list is now generated from the
control-plane entry and the worker count. A Jinja template in group_vars came
back as a string until the whitespace was trimmed (`{%- -%}`).

```shell
ansible-playbook vms.yml
```

### From one control plane to three

Three control planes need a VIP (kube-vip, ARP) in `/etc/stratum/config.yaml`
before the first controller's first boot; adding one later needs
`stratum ha-repoint` on every node. The IP plan changed to VIP `.80`, control
planes `.81-.83`, workers `.91+`, so the first VMs were destroyed and rebuilt.

The config file is parsed strictly (`UnmarshalStrict`), `kind: ClusterConfig`
is checked, `apiVersion` is not. A first draft of the template had an
`apiVersion` that was made up; it was removed after reading
`internal/cluster/config.go`.

### Restructured as a Galaxy collection

- The repo root is the collection. The three roles first written
  (prereqs, binary, node) shared `stratum_*` variables, which ansible-lint's
  production profile rejects (`var-naming[no-role-prefix]`: a role's variables
  must start with the role's name). Merged into one role `stratum` with entry
  points `prepare` and `node`: every variable is then correctly prefixed.
- `libvirt_vm` got `libvirt_vm_*` variables.
- Supported releases: v1.37.1, v1.36.5 and v1.35.9 (-stratum.0), the sha256 of
  each binary in a table in the role's defaults.

### Bugs found by running it

- **Loop variable shadowing.** The playbook passed `item.name` into the VM
  role, whose own loop also uses `item`: inside the role the VM name became
  `network-config` (a seed file's name). Fixed with `loop_var` on both sides.
- **`pkill -f` killed its own shell**: the pattern was in its own command line.
- **`stratum up` never returns.** The first version ran it with `async`; the
  play hung at the first control plane. Reading `pkg/cli/up.go` and upstream's
  test harness: `up` runs the node in the foreground, installs and enables
  `stratum.service` but does not start it. Now: start it detached, wait for
  `started kubelet`, SIGTERM it, `systemctl start stratum`.
- **Backslashes in folded YAML.** In a `>-` block there is no YAML unescaping,
  and Jinja does not unescape `\\` either: `'\\S+'` reached the regex as a
  literal backslash. Two effects: the derived MAC came out as `52:54:00:\1:\2:\3`
  (caught by a new MAC check), and the join command kept `--token <secret>` on
  the command line and pointed at the first control plane's key file path.
  The join failed safely, the token was revoked by hand. Now the join line is
  split into a list once and rebuilt with an explicit loop, unit-tested against
  the exact line formats from `pkg/cli/token.go`.
- **A `kubeconfig*` ignore pattern** would have kept the role's own
  `tasks/kubeconfig.yml` out of git. Anchored to the repo root.
- **galaxy-importer** warned that `community.crypto.openssh_keypair` and the
  playbook `omerfsen.stratum.cluster` could not be resolved: the importer lints
  without installing dependencies or the collection itself. Keys are now made
  with `ssh-keygen` (the `community.crypto` and `community.general`
  dependencies are gone) and `kvm.yml` imports `cluster.yml` by relative path.
- **Workers never registered (1): token revoked too early.** The token was
  revoked as soon as the join printed `started kubelet`. A worker's kubelet
  only then sends its bootstrap CSR, authenticated by that token: no client
  certificate, `unable to read client-cert .../kubelet-client-current.pem`.
  The token is now revoked after the node is Ready.
- **Workers never registered (2): two CSRs per worker.** Next run, each worker
  had two Pending CSRs nine seconds apart, and the Stratum operator approved
  neither: `another bootstrap CSR claims node stratum-w1`. Stratum turns off
  Kubernetes' own node-client auto-approval (`stratum-csr-autoapprove` binding
  with no subjects) and approves in its operator, refusing a node claimed by
  two CSRs (anti-takeover). The handover to systemd came right after
  `started kubelet`, before the first CSR was approved; the new kubelet filed
  a second. Now the handover waits for the Node to register.
- **Concurrent `token create`** on the first control plane failed for one of
  two workers joining in parallel; minting is now `throttle: 1`.
- **Merging into `~/.kube/config`** was added, then made opt-in
  (`stratum_kubeconfig_merge: false` by default): the controller's main
  kubeconfig is never touched unless asked.
- **Workers never registered (3): the wrong "bootstrap done" file.** The fix
  above waited for `kubelet.kubeconfig`, which a source audit named as written
  by the TLS bootstrap. It is, but at the bootstrap's start, before the
  certificate exists: the gate passed at once and the double CSR came back.
  The gate is now the certificate itself,
  `/var/lib/stratum/pki/kubelet/kubelet-client-current.pem`.

### Final verification (2026-10-30)

All VMs destroyed, then one run of `ansible-playbook playbooks/kvm.yml` on new
VMs with the final code: three control planes and two workers, 0 failures,
about 20 minutes.

| Check | Result |
|---|---|
| Nodes | 5/5 Ready, Kubernetes v1.37.1, Rocky Linux 10.2 |
| etcd | 3 started voting members |
| Certificate requests | one client CSR per worker, approved; serving CSRs approved |
| Join tokens / certificate keys | none left (`stratum token list`, `/tmp`) |
| Pods | 30/30 Running; `kubectl logs` works on workers |
| Second run | `changed=0` on every host |
| `--check` run | 0 changed, 0 failed |
| ansible-lint (production) / galaxy-importer 0.4.44 | pass / no warnings, no errors |

# omerfsen.stratum

Ansible collection for [CIQ Stratum](https://github.com/ctrliq/stratum), the
Kubernetes distribution that ships as one self-contained binary. It takes
RHEL-family hosts from a minimal install to a highly available cluster:
prerequisites, firewalld, a verified binary, the first control plane, more
control planes behind a VIP, and workers. Rerun it any time; it only changes
what is missing.

It can also provision the hosts: on one or more KVM/libvirt hypervisors it
creates Rocky Linux 10.2 VMs (static addresses, fixed MACs, one SSH key each)
and installs the cluster on them with the same role.

Built for production use: highly available control planes, SELinux
enforcing, firewalld configured, verified binaries, short-lived join tokens,
idempotent reruns. It comes with no guarantee of any kind; see
[License and disclaimer](#license-and-disclaimer).

## Contents

| Name | Type | Does |
|---|---|---|
| `omerfsen.stratum.stratum` | role | Prerequisites, firewalld, binary (`prepare`); bootstrap, HA control-plane and worker joins (`node`) |
| `omerfsen.stratum.libvirt_vm` | role | One cloud-image VM on libvirt/KVM with a static address and its own SSH key |
| `omerfsen.stratum.cluster` | playbook | A cluster on any inventory with groups `stratum_control_plane` and `stratum_workers` |
| `omerfsen.stratum.kvm` | playbook | VMs on the `kvm_hosts` hypervisors, then `cluster` on them |
| `omerfsen.stratum.kvm_destroy` | playbook | Destroy those VMs (asks for confirmation) |

## Support matrix

| | Supported |
|---|---|
| Stratum | v1.37.1-stratum.0 (Kubernetes 1.37.1, default), v1.36.5-stratum.0, v1.35.9-stratum.0 |
| Node OS | Rocky Linux 9 and 10, RHEL 9 and 10, RLC; x86_64 and aarch64; SELinux enforcing |
| Topology | 1, 3 or 5 control planes (a VIP with more than one), 0 or more workers |
| ansible-core | 2.16 or later |
| Provisioning | KVM/libvirt (NAT network or bridge, one or more hypervisors); Proxmox VE and bare metal planned |
| Tested | Stratum v1.37.1-stratum.0 on Rocky Linux 10.2 KVM VMs, 3 control planes + 2 workers, SELinux enforcing, firewalld on |

Each release's binary sha256 is pinned in the role. Another release works by
setting `stratum_version` and `stratum_sha256`.

## Quick start

### Existing hosts

```shell
ansible-galaxy collection install omerfsen.stratum
```

```yaml
# inventory.yml
all:
  vars:
    stratum_vip: 10.0.10.10            # free address on the node subnet
    stratum_node_cidr: 10.0.10.0/24
  children:
    stratum_control_plane:
      hosts:
        cp1: {ansible_host: 10.0.10.11}
        cp2: {ansible_host: 10.0.10.12}
        cp3: {ansible_host: 10.0.10.13}
    stratum_workers:
      hosts:
        worker1: {ansible_host: 10.0.10.21}
```

```shell
ansible-playbook -i inventory.yml omerfsen.stratum.cluster
export KUBECONFIG=~/.kube/stratum.yaml   # or set stratum_kubeconfig_merge: true
kubectl get nodes
```

### On KVM/libvirt

```shell
git clone https://github.com/omerfsen/stratum.git && cd stratum
make dev && source .venv/bin/activate
ansible-playbook playbooks/kvm.yml
export KUBECONFIG=~/.kube/stratum.yaml   # or set stratum_kubeconfig_merge: true
kubectl get nodes -o wide
```

```text
NAME          STATUS   ROLES    AGE   VERSION           INTERNAL-IP
stratum-cp1   Ready    <none>   9m    v1.37.1-stratum   192.168.122.81
stratum-cp2   Ready    <none>   7m    v1.37.1-stratum   192.168.122.82
stratum-cp3   Ready    <none>   5m    v1.37.1-stratum   192.168.122.83
stratum-w1    Ready    <none>   3m    v1.37.1-stratum   192.168.122.91
stratum-w2    Ready    <none>   3m    v1.37.1-stratum   192.168.122.92
```

## Documentation

- [How it works](https://github.com/omerfsen/stratum/blob/main/docs/how-it-works.md): every step, and why it is done that way
- [Existing hosts and bare metal](https://github.com/omerfsen/stratum/blob/main/docs/existing-hosts.md): requirements, inventory, air-gapped installs, signature checks
- [KVM/libvirt](https://github.com/omerfsen/stratum/blob/main/docs/kvm.md): VMs on one or more hypervisors, NAT or bridged, sizes, adding workers, destroy
- [SSH into the nodes](https://github.com/omerfsen/stratum/blob/main/docs/kvm.md#ssh-into-the-nodes): keys, `~/.ssh/config` block, jump hosts
- [Variables](https://github.com/omerfsen/stratum/blob/main/docs/variables.md): every variable, with defaults
- [Troubleshooting](https://github.com/omerfsen/stratum/blob/main/docs/troubleshooting.md)
- Role reference: `ansible-doc -t role omerfsen.stratum.stratum` and
  `ansible-doc -t role omerfsen.stratum.libvirt_vm`
- [Build log](https://github.com/omerfsen/stratum/blob/main/docs/build-log.md): how this collection was built and tested

## Security

- SELinux stays enforcing; firewalld is configured, not turned off.
- The binary's sha256 is always verified; its cosign signature can be.
- Join tokens are minted per join, passed by file (never on a command line),
  hidden from Ansible output, revoked once the node is Ready, and expire
  after 45 minutes at most. The control-plane certificate key is deleted from
  both nodes after use, also when a join fails.
- `/etc/stratum/config.yaml` and the exported kubeconfig are mode 0600.

## Roadmap

- Proxmox VE and bare-metal (PXE/Redfish) providers next to `libvirt_vm`
- Rolling upgrades (`stratum upgrade --cluster`), next release
- Node removal (drain, `stratum uninstall`, etcd member removal for control planes)
- NVIDIA GPU Operator on Stratum

## License and disclaimer

Licensed under the [Apache License 2.0](https://github.com/omerfsen/stratum/blob/main/LICENSE). The collection is provided
"AS IS", without warranties or conditions of any kind, and the authors accept
no liability for its use (sections 7 and 8 of the license). Test it on your
own systems before you rely on it.

This is an independent community project. It is not affiliated with, endorsed
or supported by CIQ. Stratum and CIQ are trademarks of their respective
owners; for supported Stratum, see Stratum PRO from CIQ.

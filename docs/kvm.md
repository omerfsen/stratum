# Stratum on KVM/libvirt

`omerfsen.stratum.kvm` creates the cluster's VMs on one or more KVM
hypervisors, then installs Stratum on them with the same role used for any
other host. Each VM is a Rocky Linux 10.2 cloud image with a static address,
a fixed MAC and its own SSH key, created idempotently with `virt-install`.

```text
  kvm_hosts (1 or more hypervisors; VMs placed round robin)
  ├── VIP          192.168.122.80   API server (kube-vip)
  ├── stratum-cp1  192.168.122.81   control plane, bootstraps   4 vCPU  6 GiB
  ├── stratum-cp2  192.168.122.82   control plane               4 vCPU  6 GiB
  ├── stratum-cp3  192.168.122.83   control plane               4 vCPU  6 GiB
  ├── stratum-w1   192.168.122.91   worker                      4 vCPU  4 GiB
  └── stratum-w2   192.168.122.92   worker                      4 vCPU  4 GiB
```

Defaults: three control planes and two workers on the controller itself, on
libvirt's NAT network `default`.

## Hypervisor requirements

- Linux with KVM (`/dev/kvm`), libvirt, `virt-install` 4.0+, `qemu-img`,
  `genisoimage`, Python 3; Ansible connects as root or with sudo.
- RAM for the VMs (default sizes: 26 GiB in total; CPUs may be overcommitted)
  and disk for thin qcow2 disks (about 3 GiB used per VM after install).
- Internet access for the first run, or a local mirror (`libvirt_vm_image_url`,
  `stratum_release_url`, the guests' package repositories).
- Several hypervisors: one L2 network shared by all VMs (a bridge on each
  host, `stratum_kvm_bridge`), so the VIP can move between control planes.

## Network modes

| | NAT (default) | Bridged |
|---|---|---|
| Setting | `stratum_kvm_network: default` | `stratum_kvm_bridge: br0` |
| VMs reachable from | the hypervisor (the controller jumps through it) | the whole LAN |
| Hypervisors | one | one or more |
| Addresses | libvirt subnet, outside its DHCP range | free addresses on the LAN |

Addresses are static (cloud-init, matched to the VM's MAC):
VIP `.80`, control plane N `.80+N`, worker N `.90+N` on `stratum_kvm_subnet`.
Each VM's MAC is derived from its name, so a rebuilt VM keeps its MAC and
address; `stratum_kvm_macs` pins your own (DHCP reservations, MAC-bound licences).

## Run

From a checkout:

```shell
git clone https://github.com/omerfsen/stratum.git && cd stratum
make dev && source .venv/bin/activate
ansible-playbook playbooks/kvm.yml
export KUBECONFIG=~/.kube/stratum.yaml   # or set stratum_kubeconfig_merge: true
kubectl get nodes -o wide
```

From Galaxy:

```shell
ansible-galaxy collection install omerfsen.stratum
cp -r ~/.ansible/collections/ansible_collections/omerfsen/stratum/inventory/kvm ./inventory
ansible-playbook -i inventory omerfsen.stratum.kvm
```

The first run takes about 25 minutes, mostly downloads (545 MB image, 704 MB
binary, packages). A second run changes nothing (`changed=0`).

## Configure

Settings: [inventory/kvm/group_vars/all.yml](../inventory/kvm/group_vars/all.yml).
Keep your changes in `inventory/kvm/group_vars/all/local.yml` (gitignored)
or pass them with `-e`.

```yaml
# inventory/kvm/group_vars/all/local.yml — three hypervisors, LAN bridge
stratum_kvm_bridge: br0
stratum_kvm_subnet: 10.0.20
stratum_kvm_gateway: 10.0.20.1
stratum_kvm_dns: [10.0.20.1]
stratum_kvm_control_plane_size: {vcpus: 8, memory_mb: 16384, disk_gb: 100}
stratum_kvm_worker_size: {vcpus: 16, memory_mb: 65536, disk_gb: 200}
stratum_kvm_worker_count: 6
stratum_control_plane_taint: true
```

```yaml
# inventory/kvm/hosts.yml
all:
  children:
    kvm_hosts:
      hosts:
        kvm1.example.com: {ansible_user: admin}
        kvm2.example.com: {ansible_user: admin}
        kvm3.example.com: {ansible_user: admin}
    stratum_control_plane:
    stratum_workers:
```

With three hypervisors each control plane lands on its own machine, so the
cluster survives the loss of one hypervisor.

### Sizes

vCPUs, memory and disk are set per node type, and per node where one must
differ:

```yaml
stratum_kvm_control_plane_size: {vcpus: 4, memory_mb: 6144, disk_gb: 40}
stratum_kvm_worker_size: {vcpus: 4, memory_mb: 4096, disk_gb: 40}
stratum_kvm_node_sizes:
  stratum-w1: {vcpus: 16, memory_mb: 65536, disk_gb: 200}
```

They apply when a VM is created. The qcow2 is grown to `disk_gb` before the
first boot, and cloud-init grows the root partition and filesystem into it on
that boot. The disk is thin: it takes only the space actually written.

Common changes:

```shell
# a single node (control plane runs the workloads)
ansible-playbook playbooks/kvm.yml -e stratum_kvm_control_plane_count=1 -e stratum_kvm_worker_count=0

# add workers to a running cluster
ansible-playbook playbooks/kvm.yml -e stratum_kvm_worker_count=4

# another Stratum release
ansible-playbook playbooks/kvm.yml -e stratum_version=v1.36.5-stratum.0
```

## SSH into the nodes

Every VM gets its own ed25519 keypair on the controller, in
`~/.ssh/stratum/<vm name>_ed25519`. The login user is `rocky` with
passwordless sudo; root login and passwords are disabled.

```shell
ssh -i ~/.ssh/stratum/stratum-cp1_ed25519 rocky@192.168.122.81   # control plane 1
ssh -i ~/.ssh/stratum/stratum-cp2_ed25519 rocky@192.168.122.82   # control plane 2
ssh -i ~/.ssh/stratum/stratum-cp3_ed25519 rocky@192.168.122.83   # control plane 3
ssh -i ~/.ssh/stratum/stratum-w1_ed25519  rocky@192.168.122.91   # worker 1
ssh -i ~/.ssh/stratum/stratum-w2_ed25519  rocky@192.168.122.92   # worker 2
```

To type just `ssh stratum-cp1`, add to `~/.ssh/config` on the controller:

```text
Host stratum-cp1 stratum-cp2 stratum-cp3 stratum-w1 stratum-w2
    User rocky
    IdentityFile ~/.ssh/stratum/%n_ed25519
    StrictHostKeyChecking accept-new

Host stratum-cp1
    HostName 192.168.122.81
Host stratum-cp2
    HostName 192.168.122.82
Host stratum-cp3
    HostName 192.168.122.83
Host stratum-w1
    HostName 192.168.122.91
Host stratum-w2
    HostName 192.168.122.92
```

VMs on a remote hypervisor's NAT network are only reachable through that
hypervisor; add `ProxyJump admin@kvm1.example.com` (its SSH login) to their
entries, or pass `-J admin@kvm1.example.com` to `ssh`. The playbook does the
same on its own (`stratum_kvm_proxy_jump`). On a bridged network the VMs are
reachable directly.

`stratum` lives in `/usr/local/bin`, which sudo's `secure_path` on Rocky does
not include, so give the full path:

```shell
sudo /usr/local/bin/stratum kubectl get pods -A
sudo /usr/local/bin/stratum etcdctl member list -w table
sudo journalctl -u stratum -f
```

A rebuilt VM keeps its address but gets new SSH host keys; `kvm_destroy`
removes the old ones from `~/.ssh/known_hosts`.

## Dashboard

`https://<vip>:30443`, sign in with the token from

```shell
kubectl -n stratum-dashboard get secret stratum-dashboard-admin-token \
  -o jsonpath='{.data.token}' | base64 -d; echo
```

## Destroy

```shell
ansible-playbook playbooks/kvm_destroy.yml -e stratum_kvm_destroy_confirm=yes
```

Irreversible: deletes the VMs, their disks and seeds, and the exported
kubeconfig. The base image, the binary cache (`~/.cache/stratum`) and the SSH
keys stay, so the next run downloads nothing.

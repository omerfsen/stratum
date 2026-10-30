# Existing hosts and bare metal

Stratum installs on any RHEL-family 9+ host you can reach over SSH: bare
metal, VMs on any hypervisor, cloud instances. Provisioning those hosts is up
to you; this collection takes them from a minimal install to a cluster.

## Host requirements

- Rocky Linux, RHEL or RLC 9 or 10, x86_64 or aarch64, cgroup v2 (the default
  on 9 and 10), SELinux enforcing.
- A user with sudo (Ansible `become`), Python 3 (present on a minimal install).
- Package repositories reachable (BaseOS, AppStream) for the SELinux packages
  and tools; or a local mirror.
- About 2 GB free under `/usr/local/bin` and `/var/lib/stratum` for the
  binary and its unpacked payload. CIQ does not publish CPU or RAM minimums;
  4 vCPU and 6 GiB per control plane run the stock add-ons comfortably.
- For more than one control plane: a free VIP on the controllers' L2 segment.
- Between the nodes: the ports in [How it works](how-it-works.md#1-prepare-all-nodes-at-once).
  With `stratum_firewalld: auto` (default) the role opens them where
  firewalld runs; for any other firewall, open them yourself.

## Inventory

```yaml
# inventory/production/hosts.yml
all:
  vars:
    ansible_user: admin
    stratum_vip: 10.0.10.10
    stratum_node_cidr: 10.0.10.0/24
    stratum_cluster_name: prod
    stratum_control_plane_taint: true
  children:
    stratum_control_plane:
      hosts:
        cp1.example.com: {ansible_host: 10.0.10.11}
        cp2.example.com: {ansible_host: 10.0.10.12}
        cp3.example.com: {ansible_host: 10.0.10.13}
    stratum_workers:
      hosts:
        worker1.example.com: {ansible_host: 10.0.10.21}
        worker2.example.com: {ansible_host: 10.0.10.22}
```

The first host of `stratum_control_plane` bootstraps the cluster. Order the
group so it is the one you want; do not change it afterwards.

The node address defaults to the host's default-route IPv4. On hosts with
several NICs set `stratum_node_ip` per host.

## Install

```shell
ansible-galaxy collection install omerfsen.stratum
ansible-playbook -i inventory/production omerfsen.stratum.cluster
```

## Air-gapped

```yaml
stratum_binary_source: controller        # default: nodes never download
stratum_binary_local_path: /srv/mirror/stratum-linux-amd64
```

The controller needs the binary (copied in by you; the sha256 is still
checked), the nodes need a reachable package mirror for the six RPMs. The
binary carries every container image, so no registry is needed.

## Supply chain

```yaml
stratum_verify_signature: true   # needs cosign on the controller
```

checks the keyless cosign signature of the binary against the certificate
identity `https://github.com/ctrliq/stratum/.github/workflows/release.yml@refs/tags/<version>`
and the GitHub Actions OIDC issuer.

Keyless verification needs the internet: the role downloads the `.sig` and
`.pem` from the release URL, and cosign checks them against Sigstore's
transparency log. In an air-gapped install verify the binary where you
download it, before copying it in, and leave `stratum_verify_signature` off;
the sha256 check still runs on every host.

## On the nodes

Log in as your usual admin user (the `ansible_user` of the inventory). The
binary is `/usr/local/bin/stratum`; with sudo give the full path, because
sudo's `secure_path` usually does not include `/usr/local/bin`:

```shell
sudo /usr/local/bin/stratum kubectl get nodes
sudo /usr/local/bin/stratum etcdctl member list -w table
sudo journalctl -u stratum -f
```

## Grow the cluster

Add hosts to the inventory and rerun the playbook. Existing nodes are left
alone; new workers join, and new control planes join one at a time. Keep the
number of control planes odd.

# Troubleshooting

## Where to look

| What | Where |
|---|---|
| First start of a node (bootstrap or join) | `/var/log/stratum-first-up.log` on the node |
| The node after that | `journalctl -u stratum` |
| Cluster state | `sudo /usr/local/bin/stratum kubectl get nodes,pods -A` on a control plane |
| etcd members | `sudo /usr/local/bin/stratum etcdctl member list -w table` |
| Join tokens | `sudo /usr/local/bin/stratum token list` |

When a first start fails, the role prints the last 40 lines of
`/var/log/stratum-first-up.log`.

## A join failed

- Failed before Stratum saved the node's state (`/var/lib/stratum/config/node.yaml`
  missing): rerun the playbook. The role mints a fresh token and repeats the
  join; for a control plane Stratum reconciles its join journal against the
  member list, so no second etcd member is added.
- Failed after it (the role says "never finished joining"): the node has to
  start clean, as below.
- A worker whose certificate request stays pending: the role prints the
  operator's reason. "another bootstrap CSR claims node" or a reused name
  needs the name claim cleared first, see "Reusing a node name" in Stratum's
  configuration documentation.

To start a node from scratch:

```shell
sudo /usr/local/bin/stratum uninstall --yes      # removes /var/lib/stratum AND /etc/stratum
```

then rerun the playbook, which rewrites `/etc/stratum/config.yaml`. For a
control plane that had become an etcd member, remove the member first from a
healthy control plane (`stratum etcdctl member remove <id>`), or the cluster
keeps counting it toward quorum.

## "Use an odd number of control planes"

etcd needs a majority: two members tolerate no failure, four no more than
three. Use 1, 3 or 5 control planes.

## The VIP does not answer

- The VIP must be free and on the same L2 segment as the controllers (ARP).
  `arping -I eth0 <vip>` from a controller shows who answers.
- kube-vip announces on the default-route interface; set
  `stratum_vip_interface` if the controllers' node network is another NIC.
- On routed networks, set `stratum_extra_config: {vip: {bgp: true}}` and
  configure BGP peers as in Stratum's high-availability documentation.

## Pods cannot reach each other, or nodes stay NotReady

- With firewalld on another backend than nftables, a `firewall-cmd --reload`
  flushes Cilium's rules. Restart Stratum on the node: `systemctl restart stratum`.
- Check the ports in [How it works](how-it-works.md#1-prepare-all-nodes-at-once)
  are open between every pair of nodes, and that the pod range is in the
  `trusted` zone: `firewall-cmd --zone=trusted --list-sources`.

## "No sha256 for v… "

The release is not in the role's table. Take both checksums from the
release's `sha256sums.txt` and set:

```yaml
stratum_version: v1.37.2-stratum.0
stratum_sha256:
  amd64: <sha256 of stratum-linux-amd64>
  arm64: <sha256 of stratum-linux-arm64>
```

## Config changes are not applied

`/etc/stratum/config.yaml` changes on a running node are reported, not
applied: restart with `systemctl restart stratum` when it suits you. The
network block (pod and service ranges, cluster domain) cannot change after the
cluster's first boot.

## KVM: a VM does not get its address

- The address is static, set by cloud-init and matched to the VM's MAC. Check
  the console: `virsh console <vm>` (exit with `Ctrl+]`).
- The address must be outside the libvirt network's DHCP range and unused.
- Rebuilding a VM keeps its MAC (derived from its name) but gets new SSH host
  keys; `kvm_destroy` forgets the old ones.

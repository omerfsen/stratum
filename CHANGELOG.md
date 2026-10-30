# Changelog

## 0.1.0

First release.

- `stratum` role: prerequisites, firewalld, verified binary (sha256, optional
  cosign), first control plane, HA control-plane joins through a VIP, workers.
- Supported Stratum releases: v1.37.1-stratum.0 (default), v1.36.5-stratum.0,
  v1.35.9-stratum.0.
- `libvirt_vm` role: Rocky Linux 10.2 cloud-image VMs on libvirt/KVM, NAT
  network or host bridge, static address, stable or pinned MAC.
- Playbooks: `cluster` (any inventory), `kvm` (VMs on one or more
  hypervisors, then the cluster) and `kvm_destroy`.
- Tested with Stratum v1.37.1-stratum.0 on Rocky Linux 10.2 KVM VMs, three
  control planes and two workers, SELinux enforcing, firewalld on.

# omerfsen.stratum.libvirt_vm

Creates one cloud-image VM on a libvirt/KVM host: an ed25519 keypair, a
cloud-init seed with a static address, and a UEFI virtio domain from
virt-install. Default image: Rocky Linux 10.2 GenericCloud, checksum pinned.
Idempotent; runs on the KVM host as root (needs `virt-install`, `qemu-img`,
`genisoimage`). x86_64 hypervisors (q35, UEFI, x86_64 image). vCPUs, memory
and disk apply when the VM is created; cloud-init grows the root filesystem
into the disk on first boot.

```yaml
- ansible.builtin.include_role:
    name: omerfsen.stratum.libvirt_vm
  vars:
    libvirt_vm_name: stratum-cp1
    libvirt_vm_ip: 192.168.122.81
    libvirt_vm_vcpus: 4
    libvirt_vm_memory_mb: 6144
```

All options: [meta/argument_specs.yml](meta/argument_specs.yml).

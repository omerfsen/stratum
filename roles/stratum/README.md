# omerfsen.stratum.stratum

Installs [CIQ Stratum](https://github.com/ctrliq/stratum) Kubernetes nodes on
RHEL-family 9+ hosts (Rocky Linux, RHEL, RLC), x86_64 or aarch64.

Entry points:

| Entry point | Does | Run on |
|---|---|---|
| `prepare` | platform, cgroup v2 and SELinux checks; packages; firewalld as CIQ tests it; the binary with sha256 (and optional cosign) verification | all nodes, in parallel |
| `node` | `/etc/stratum/config.yaml`, then bootstrap (first control plane), HA control-plane join through the VIP, or worker join; admin kubeconfig to the controller | one node at a time, control planes first |
| `main` | `prepare` then `node` | with `serial: 1` |

Inventory groups: `stratum_control_plane` (1, 3, 5...; the first one
bootstraps) and `stratum_workers` (0 or more). With more than one control
plane set `stratum_vip`, a free address on the node subnet.

```yaml
- hosts: stratum_control_plane:stratum_workers
  become: true
  tasks:
    - ansible.builtin.include_role:
        name: omerfsen.stratum.stratum
        tasks_from: prepare

- hosts: stratum_control_plane:stratum_workers
  become: true
  serial: 1
  tasks:
    - ansible.builtin.include_role:
        name: omerfsen.stratum.stratum
        tasks_from: node
```

This is what `omerfsen.stratum.cluster` does. Every variable, with its type
and default, is in [meta/argument_specs.yml](meta/argument_specs.yml) and
`ansible-doc -t role omerfsen.stratum.stratum`.

Security notes:

- The binary's sha256 is always checked; `stratum_verify_signature: true`
  also checks the keyless cosign signature against the release workflow.
- Join tokens are minted per join with a short TTL (`stratum_join_token_ttl`),
  passed with `--token-file` (never on a command line, never in Ansible
  output), and revoked once the node is Ready.
- The control-plane certificate key is copied over for the join and deleted
  from both nodes afterwards, also when the join fails.
- `stratum up` never runs again on a node that is in the cluster, and the
  binary of such a node is never swapped (`stratum_allow_binary_replace`).
- Check mode (`--check`) is safe: it reports and changes nothing.

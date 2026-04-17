# k8s-install

The `k8s-install` role is dedicated exclusively to the distribution-specific installation of the core Kubernetes components natively required to instantiate or join a cluster infrastructure. 

It handles:
- Locating and defining the appropriate Kubernetes package repositories for your cluster OS (e.g., RedHat `.repo` mappings vs. Debian `.list` mappings).
- Installing the precise `kubelet`, `kubeadm`, and `kubectl` binaries corresponding to `kubernetes_version`.
- **Configuring the Container Runtime Interface (CRI)**: The role is explicitly responsible for dynamically querying `kubeadm` to identify the exact `pause` image (sandbox container) mapped to the downloaded version. It utilizes this data to systematically inject correct daemon properties into the respective CRI config files (e.g. `crio.conf` or `containerd.toml`) and instructs `kubelet` on exactly which local Unix wrapper socket to utilize.

## How to Use

Invoke this immediately following the definition of your environment repositories and storage plugins:

```yaml
- name: Deploy Kubernetes Codebase
  hosts: all
  roles:
    - role: k8s-install
      tags: [ k8s, install ]
```

## Variables

Adjust settings appropriately depending on the ecosystem you reside within or whether internet requests need to be routed via corporate registries.

### User-Defined Variables (`defaults/main.yml`)

| Variable                       | Default               | Description                                                                                                                                           |
|--------------------------------|-----------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| `kubernetes_version`           | `"1.34"`              | The exact semantic version (down to the patch) of `kubelet` and `kubeadm` to install on the machine.                                                  |
| `container_runtime`            | `"crio"`              | String referencing the CRI you installed prior. It informs `kubelet` about the communication socket parameters it must utilize locally.              |
| `kubernetes_registry_mirror`   | `"registry.k8s.io"`   | The container image registry to validate configuration dependencies from. If your environment prohibits external internet access, change this!     |

### Role Internal Variables (`vars/`)
You shouldn't need to override these. They outline mapping architectures natively required.

| Variable | Origin | Description |
|---|---|---|
| (Various OS Mappings) | `vars/common.yml`... | Internal OS-level maps defining exact package architecture paths or names for the `kubelet` daemon dependencies. |

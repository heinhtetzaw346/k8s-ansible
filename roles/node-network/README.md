# node-network

The `node-network` role focuses strictly on preparing the underlying OS networking environment. Because Kubernetes relies heavily on complex container networking configurations (CNI plugins like Calico or Flannel), the host kernels need to have specific routing capabilities and modules loaded beforehand.

This role reliably handles:
- **Kernel Modules**: Loads persistent base modules (`overlay`, `br_netfilter`) necessary to virtualize networks across physical interfaces.
- **Sysctl Adjustments**: Applies routing metrics persistently (`net.ipv4.ip_forward`, `net.bridge.bridge-nf-call-*`) allowing traffic to be bridged into the container namespace successfully.
- **Optional IPVS Readiness**: If IPVS proxying is enabled, it ensures all `ip_vs` kernel structures are mounted and installs the administrative package (`ipvsadm`).

## How to Use

Integrate this tightly with your deployment after system packages and storage configurations have executed:

```yaml
- name: Prepare Host Networking
  hosts: all
  roles:
    - role: node-network
      tags: [ network ]
```

## Variables

This role provides highly customizable dictionaries allowing you to inject arbitrary tuning parameters if your specific network fabric requires it.

### User-Defined Variables (`defaults/main.yml`)
Use these fields to toggle proxy methods or supply advanced overlay routing parameters.

| Variable                      | Default | Description                                                                                             |
|-------------------------------|---------|---------------------------------------------------------------------------------------------------------|
| `ipvs_enabled`                | `false` | If toggled `true`, installs the `ipvsadm` package and mounts IP Virtual Server (`ip_vs_*`) kernel modules. |
| `additional_kernel_modules`   | `[]`    | A custom list where you can specify strings of extra kernel modules matching your host OS.              |
| `additional_sysctl_configs`   | `{}`    | A custom key-value dictionary for any distinct sysctl configuration variables required by your cluster. |

### Role Internal Variables (`vars/`)
These variables maintain the baseline stability. You should only override them if your OS behaves extraordinarily differently than the standard Linux kernel defaults.

| Variable                        | Origin               | Description                                                                              |
|---------------------------------|----------------------|------------------------------------------------------------------------------------------|
| `sysctl_configs`                | `vars/common.yml`    | The hardcoded dictionary that sets IP forwarding and bridge netfilter properties to `"1"`.|
| `base_kernel_modules`           | `vars/common.yml`    | Essential list mapping to `overlay` and `br_netfilter`.                                 |
| `ipvs_kernel_modules`           | `vars/common.yml`    | IPVS modules mapping to `ip_vs`, `ip_vs_rr`, `ip_vs_wrr`, `ip_vs_sh`, `nf_conntrack`.    |
| `ipvs_pkg_name`                 | `vars/common.yml`    | Defaults to `"ipvsadm"`.                                                                |
| `kernel_modules_extra_pkg_name` | `vars/RedHat.yml`    | (RedHat specific) Name of the extended kernel package required for loading modules.     |

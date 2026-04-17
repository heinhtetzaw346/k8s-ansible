# node-preflight

The `node-preflight` role is the critical first step in preparing your physical machines or virtual instances for a Kubernetes deployment. Its primary responsibility is to ensure the base OS environment conforms to the rigid requirements put forth by Kubernetes and its container runtimes.

This role executes a comprehensive checklist of system-level configurations including:
- Installing and configuring time synchronization (`chronyd` / NTP) to ensure distributed consensus remains stable.
- Managing and resolving domain name searches by dynamically writing custom DNS entries to `/etc/resolv.conf`.
- Setting the system timezone.
- Handling OS firewalls (e.g., stopping and disabling `firewalld` or `ufw`) if requested.
- Toggling mandatory system security features (SELinux, AppArmor) to prevent permission issues with container storage and networks.
- Systematically disabling filesystem `swap` across mount boundaries to ensure kubelet stability.
- Configuring cgroup v2 system specifications.
- Pulling down specialized dependencies like `eBPF` packages where necessary (e.g., for Alpine).

## How to Use

If running the entire suite, this runs first. By tying it to your environment playbooks, you can execute this role against all target nodes natively:

```yaml
- name: Prepare OS for Kubernetes
  hosts: all
  roles:
    - role: node-preflight
      tags: [ preflight, config ]
```

## Variables

The role is parameterized into two layers: variables you are expected to tune (`defaults`), and variables internal to the role used for cross-OS compatibility (`vars`).

### User-Defined Variables (`defaults/main.yml`)
These parameters dictate the behavioral characteristics of the preflight environment. You can override these in your playbook's `vars_files` (e.g., using `sample-vars.yml`).

| Variable             | Default                 | Description                                                                                                                                           |
|----------------------|-------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| `ntp_server`         | `"asia.pool.ntp.org"`   | The upstream target used to synchronize the hardware/system clock via `chrony`.                                                                       |
| `timezone`           | `"UTC"`                 | Global timezone configured across all cluster nodes.                                                                                                  |
| `dns_servers`        | `["1.1.1.1", "8.8.8.8", "9.9.9.9"]` | List of DNS resolvers applied to the machines. *It is heavily recommended to supply a local DNS server as your primary endpoint.*        |
| `search_domain`      | `"local"`               | Local DNS search domain suffix appended to hostname lookups.                                                                                          |
| `security_enabled`   | `false`                 | Controls SELinux on RedHat-based or AppArmor on Debian-based systems. Keeping this `false` drops the systems into permissive or disabled modes.      |
| `firewall_enabled`   | `true`                  | Determines if OS-level firewalls (like `firewalld` or `ufw`) should be retained. Typically disabled in heavily managed environments or overlay networking. |
| `swap_enabled`       | `false`                 | Controls the presence of filesystem swap. Kubernetes normally expects swap to be entirely disabled for the `kubelet` to run correctly.                |

### Role Internal Variables (`vars/`)
These variables define OS-specific mappings such as file paths and package names. You rarely need to override these unless interacting with a significantly heavily modified OS variant.

| Variable              | Origin               | Description                                                                 |
|-----------------------|----------------------|-----------------------------------------------------------------------------|
| `chrony_pkg_name`     | `vars/common.yml`    | The name of the `chrony` package to install. (`"chrony"`)                   |
| `chrony_service_name` | `vars/common.yml`    | The name of the `chrony` daemon service to manage. (`"chronyd"`)            |
| `resolv_conf_path`    | `vars/common.yml`    | The absolute routing path for DNS configurations. (`"/etc/resolv.conf"`)    |
| `fstab_path`          | `vars/common.yml`    | The absolute path to the system mounting table, used to disable swap.       |
| `chrony_conf_path`    | `vars/<OS_Family>.yml` | Path to the chrony configuration file. Varies notably based on OS (`/etc/chrony.conf` for RedHat vs. `/etc/chrony/chrony.conf` for Debian/Alpine). |

## Troubleshooting

> [!WARNING]
> **cgroup v2 Failures**: If a task fails on the cgroup v2 check, you must verify that the host node has `cgroupfs v2` enabled at the OS level, is utilized by the init system, and is mounted exactly at `/sys/fs/cgroup`.
>

- **Systemd (Debian & RedHat)**: Modern releases handle and utilize cgroup v2 natively by default. If it fails, ensure you don't have legacy kernel parameters active (like `systemd.unified_cgroup_hierarchy=0`).
- **OpenRC (Alpine)**: OpenRC uses a hybrid cgroups setup by default. You might need to explicitly set it to cgroup v2 hierarchy via your `/etc/rc.conf` file. See the official documentation for further guidance: [Alpine Linux OpenRC Cgroups Wiki](https://wiki.alpinelinux.org/wiki/OpenRC#Cgroups).
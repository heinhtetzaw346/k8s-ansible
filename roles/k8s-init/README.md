# k8s-init

The `k8s-init` role acts as the grand orchestrator of the entire automation playbook. Once all dependencies, network tools, runtime environments, and Kubernetes dependencies have been written to the hosts, this role assesses the cluster state and deploys your Control Planes, Worker Nodes, and High Availability instances systematically.

The logic operates conditionally to:
- Verify if a machine has already been joined to a cluster to prevent overwriting existing infrastructures.
- Automate **High Availability Control Planes**: If `k8s_control_plane` has more than one node mapped to it, the role automatically deploys Keepalived configs with a Virtual IP (`control_plane_lb_vip`) to balance requests perfectly!
- Pre-pull required Kubernetes images using `kubeadm config images pull` through your chosen registry structure.
- Generate valid `kubeadm init` commands containing all dynamic routing variables (Pod CIDR, Service CIDR, DNS domains) to construct the primary genesis node.
- Abstract and regenerate tokens securely to join all downstream Control Plane peers and Worker Nodes simultaneously.

## Architecture Details

### High Availability (HA) Load Balancing
For environments requiring robust uptime, orchestrating multiple control plane nodes is imperative. This role implements a native software load balancing approach officially endorsed inside the Kubernetes documentation regarding HA considerations:
[Options for Software Load Balancing](https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md#options-for-software-load-balancing). By utilizing `keepalived` married with a Virtual IP (VIP), requests aimed at the overarching `control_plane_endpoint` automatically float flawlessly between healthy master nodes, securing the cluster against hardware failures.

### Declarative Kubeadm Initialization
Rather than relying on convoluted command-line arguments that are difficult to track and maintain (e.g. `kubeadm init --pod-network-cidr=...`), this role dynamically generates a structured `kubeadm-config.yml` file. This declarative file governs the standard `InitConfiguration` and `ClusterConfiguration` properties flawlessly prior to execution, ensuring the deployment is stable, easily reviewable, and significantly easier to upgrade in the future.

## How to Use

Always declare this as the final block inside your operational execution strategy:

```yaml
- name: Orchestrate the Cluster
  hosts: all
  roles:
    - role: k8s-init
      tags: [ k8s, init ]
```

## Variables

This role consumes a massive amount of variables defined across the process to weave the cluster together flawlessly.

### User-Defined Variables (`defaults/main.yml`)
These flags dictate exactly how your subnets and cluster components are modeled. Adjust them safely within your global `.yml` values.

| Variable                   | Default           | Description                                                                                                                           |
|----------------------------|-------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| `pod_network_cidr`         | `""`              | Defines the subnet allocated internally for running container Pods. (Crucial for most CNIs like Calico).                              |
| `service_network_cidr`     | `""`              | The subnet allocated to local internal Services routing via `kube-proxy`.                                                             |
| `control_plane_endpoint`   | `""`              | Dictates the public-facing URL your workers map back to. Automatically inferred if using native HA features.                          |
| `cluster_dns_domain`       | `""`              | Re-adjusts the internal core-dns domain (defaults to `cluster.local` inside Kubernetes if unspecified).                               |
| `ipvs_enabled`             | `false`           | Activates proxy configurations utilizing `ipvs` strictly when executing `kubeadm`.                                                    |
| `kubernetes_registry_mirror` | `"registry.k8s.io"`| Upstream location to fetch control-plane images during execution initialization.                                                      |

**High Availability Parameters**
These are only parsed if you configure multiple control planes in your `inventory`:

| Variable                | Default      | Description                                                                                             |
|-------------------------|--------------|---------------------------------------------------------------------------------------------------------|
| `control_plane_lb_vip`  | `""`         | The Virtual IP (VIP) to assign to the Keepalived router cluster handling the API server locally.        |
| `control_plane_lb_port` | `"443"`      | Internal port forwarding proxy.                                                                         |
| `control_plane_lb_auth` | `"k8spass"`  | Authorization secret utilized natively by the Keepalived instances to perform VRRP handshakes securely! |
| `control_plane_lb_vrid` | `"51"`       | Isolated router grouping ID. Adjust if deploying multiple diverse instances onto the same local switch. |

### Role Internal Variables (`vars/`)

| Variable | Origin | Description |
|---|---|---|
| `kubernetes_cluster_existence_path` | `vars/common.yml` | Absolute path evaluated to determine if a node is already joined (`/etc/kubernetes/admin.conf` or `kubelet.conf`). |

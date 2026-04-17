# k8s-ansible Collection

A comprehensive collection of Ansible roles designed for initializing and managing Kubernetes clusters from the ground up. This repository handles the entire physical or virtual machine preparation lifecycle, configuring container runtimes, installing Kubernetes components, initializing the cluster, and setting up high availability for the control plane.

## Available Roles Overview

The collection is broken down into structured, modular roles, running sequentially to prepare, install, and instantiate your cluster instances. Below is the step-by-step order of execution:

1. **[node-preflight](./roles/node-preflight/README.md)**  
   *Prepares the base OS.* Disables swap, sets time zones, configures system firewalls/security parameters, ensures necessary packages exist, and handles underlying DNS configurations.
   
2. **[node-storage](./roles/node-storage/README.md)**  
   *Configures host storage.* Installs dependencies, packages, and clients required for dynamic storage backends (e.g., iSCSI, NFS client).
   
3. **[node-network](./roles/node-network/README.md)**  
   *Prepares network requirements.* Pushes necessary sysctl parameters for routing, sets up eBPF dependencies, configures IPVS if enabled, and handles kernel modules vital for container networking.

4. **[container-runtime](./roles/container-runtime/README.md)**  
   *Installs the runtime layer.* Sets up your choice of container runtime engine (e.g., CRI-O or containerd) to schedule and execute your pods.
   
5. **[k8s-install](./roles/k8s-install/README.md)**  
   *Installs Kubernetes packages.* Adds the official Kubernetes (or required mirror) repositories, and installs `kubelet`, `kubeadm`, and `kubectl` at the specified versions.

6. **[k8s-init](./roles/k8s-init/README.md)**  
   *Initializes the cluster.* Configures VIP load balancing (Keepalived) for HA deployments, executes `kubeadm init` on the primary control plane, manages tokens, and coordinates the joining process for all remaining control plane and worker nodes.

   > [!NOTE]
   > Control plane high availability is automatically configured based on your inventory—if more than one control plane node is defined, the HA setup runs; if only one is defined, it is safely skipped.

## How to Use the Sample Files

This repository ships with sample configurations so you can quickly get started without modifying the underlying roles. It is heavily recommended to copy or directly edit the `sample-*.yml` templates to suit your specific local network configuration.

### 1. `sample-inventory.yml`
This file defines groups of instances indicating what role each instance will play. 
- You should place the machine IPs (or FQDNs) under `k8s_control_plane` for nodes intended to run control plane components (API Server, ETCD).
  
  > [!NOTE]
  > **HA Auto-Detection**: The collection automatically sets up High Availability (HA) if you place more than one node in this group. If only one node is defined, the HA setup is skipped.

- You should place your worker machine IPs under `k8s_workers` meant to run your application pods. 

> [!NOTE]
> Be sure to adjust the `ansible_user` and `ansible_host` variables as necessary. 

### 2. `sample-vars.yml`
This file acts as a centralized directory for every single toggle and configuration option. It overrides all defaults globally using `vars_files` inside your playbook.
- Explore the document and ensure environmental expectations such as NTP, DNS servers, interface configurations, and container runtime versions are accurate.
- Adjust `control_plane_lb_vip` to dictate the virtual IP your loadbalancers should assign!

### 3. `sample-playbook.yml`
This is your primary entrypoint payload. It ties the inventory target (`hosts: all`), the global variables target (`vars_files`), and the execution roadmap (`roles`) together.
- You can execute individual tags to perform partial deployments (e.g., `ansible-playbook -i sample-inventory.yml sample-playbook.yml --tags preflight,config`).

## Tag Management and Task Isolation

Ansible allows targeting specific portions of the automation suite via tags. Within this collection, it is important to differentiate between **Playbook Tags (User-Defined)** and **Role Tags (Pre-existing/Functional)**.

### 1. Playbook Tags (Editable, Broad Scope)
These are custom tags you map to the roles within your playbook (e.g., in `sample-playbook.yml`). Applying a tag like `pre-k8s` or `network` to an entire role invocation means that pointing Ansible at that tag will blanket-execute every internal task inside that role.

### 2. Role Tags (Pre-existing, Granular Scope)
Internally, the underlying roles have been annotated with specific, functional tags that separate logical tasks. These allow you to skip massive role executions and focus on performing localized functions across your cluster. For example, running `ansible-playbook [...] --tags ntp,swap` only patches those features.

Here is a list of prominent internal pre-existing tags you can use to isolate specific tasks:

- **`node-preflight` role:** 
  - `dns`: Configures DNS resolvers.
  - `ntp`: Synchonizes NTP/chrony configurations.
  - `timezone`: Sets the base OS timezone.
  - `firewall`: Handles OS firewall setup/deactivation.
  - `security`: Masks/Disables system security components (SELinux, AppArmor).
  - `swap`: Disables filesystem swap.
  - `cgroup`: Validates and configures cgroupv2 routing.
  - `ebpf`: Handles specific packet filtering OS dependencies.
- **`node-storage` role:**
  - `iscsi`: Limits execution to iSCSI target dependencies.
  - `nfs`: Limits execution to NFS storage dependencies.
- **`node-network` role:**
  - `kernel`: Injects overlay/br_netfilter kernel modules.
  - `sysctl`: Applies bridge network routing sysctl bindings.
  - `ipvs`: Sets up IP Virtual Server balancing mechanisms.
- **`container-runtime` role:**
  - `container-runtime`: Executes everything related to runtime initialization.
- **`k8s-install` role:**
  - `k8s-components`: Isolates execution to installing `kubelet`, `kubeadm`, and `kubectl` dependencies.
  - `cri-config`: Forces container runtime interface socket configurations mappings for kubeadm.
- **`k8s-init` role:**
  - `control-plane-ha`: Safely isolates the setup structure specific to keepalived components and VIP load balancing without generating or running k8s tokens.

## Node Targeting and Scaling (--limit)

When expanding or managing an existing cluster, you rarely want to run tasks against every single node again. You can restrict playbook execution to specific nodes or groups using the `--limit <nodes>` flag.

This is especially useful when:
- **Adding new worker nodes**: Ensure the playbook only processes the newly defined workers rather than touching existing components. 
  `ansible-playbook -i sample-inventory.yml sample-playbook.yml --limit new-worker-node`
- **Adding new control plane nodes**: Target a new control plane member exclusively to join an existing HA cluster.
  `ansible-playbook -i sample-inventory.yml sample-playbook.yml --limit new-cp-node`
- **Patching a single host**: If one node failed a preflight check, you can run isolated tagging workflows securely against only that node.

## Quick Start

1. Edit `sample-inventory.yml` with your nodes' SSH details.
2. Edit `sample-vars.yml` and provide your network information.
3. Validate connection and deployment via the main playbook:
   ```bash
   ansible-playbook -i sample-inventory.yml sample-playbook.yml -K
   ```

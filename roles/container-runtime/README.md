# container-runtime

The `container-runtime` role bridges the gap between the operating system and Kubernetes by installing the core Container Runtime Interface (CRI). Kubernetes schedules and maintains workloads, but the underlying execution of containers fundamentally relies on these engines. 

The role is fully autonomous and operates dynamically across different OS environments (RedHat/Debian/Alpine). It intelligently:
- Asserts that the requested runtime engine exists and is supported.
- Resolves GPG encryption keys for downstream package managers automatically.
- Mounts necessary repositories specific to the OS distribution requested.
- Performs exact version orchestration, downloading specifically pinned `cri-o` or `containerd` daemons vs. rolling packages.

## How to Use

Integrate this into your playbook right after your OS and storage capabilities have been defined:

```yaml
- name: Prepare Container Engines
  hosts: all
  roles:
    - role: container-runtime
      tags: [ runtime ]
```

## Variables

This role lets you choose exactly which engine powers your pods, as well as precisely control the software versioning applied to your data plane nodes.

### User-Defined Variables (`defaults/main.yml`)
These variables drive the installation paths and versions. Update them securely via your `sample-vars.yml` global structure.

| Variable             | Default                 | Description                                                                                                                                           |
|----------------------|-------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| `container_runtime`  | `"crio"`                | String defining the desired container runtime. Valid options currently include: `"crio"` or `"containerd"`.                                           |
| `crio_version`       | `"1.34"`                | The *major* version footprint of CRI-O to install. (CRI-O versions typically mirror exact Kubernetes cluster major releases).                         |
| `containerd_version` | `"2.2.0"`               | The *full semantic* version footprint of containerd to install (e.g., `x.y.z`).                                                                       |

### Role Internal Variables (`vars/`)
Internal constants driving security validation checks to prevent botched setups.

| Variable                       | Origin            | Description                                                              |
|--------------------------------|-------------------|--------------------------------------------------------------------------|
| `supported_container_runtimes` | `vars/common.yml` | Strict list containing valid engines to prevent typos or unsupported CRIs (`["crio", "containerd"]`). |

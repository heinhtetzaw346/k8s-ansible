# node-storage

The `node-storage` role prepares your Kubernetes nodes by installing and configuring the foundational software dependencies required for dynamic storage backends. By default, Kubernetes doesn't ship with drivers or clients for external storage solutions. To mount dynamic persistent volumes like Network File System (NFS) or Internet Small Computer Systems Interface (iSCSI), the underlying base instances need these specific clients installed outright.

This role encapsulates tasks for deploying dependencies for:
- **iSCSI**: Installs `open-iscsi`/`iscsi-initiator-utils` and handles enabling/starting the `iscsid` initiator daemon.
- **NFS**: Installs `nfs-common`/`nfs-utils` to allow Kubernetes pods native NFS mounts.

## How to Use

Simply include it inside your deployment playbook immediately after your preflight configurations:

```yaml
- name: Prepare OS dependencies
  hosts: all
  roles:
    - role: node-storage
      tags: [ storage ]
```

## Variables

The parameters govern which storage backends should be configured on the hosts. The role abstracts away OS package deviations internally.

### User-Defined Variables (`defaults/main.yml`)
These parameters toggle the specific components. You can override these inside your root playbook's `sample-vars.yml` structure. By default, both are typically explicitly disabled.

| Variable             | Default                 | Description                                                                                                                                           |
|----------------------|-------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| `nfs_enabled`        | `false`                 | Activates the installation of NFS client binaries, allowing dynamic mounting.                                                                         |
| `iscsi_enabled`      | `false`                 | Activates the installation of the iSCSI daemon and targets, registering the node as a compliant iSCSI initiator.                                      |

### Role Internal Variables (`vars/`)
These variables resolve compatibility between RedHat, Debian, and Alpine derivatives. You generally do not need to override these.

| Variable              | Origin                   | Description                                                                    |
|-----------------------|--------------------------|--------------------------------------------------------------------------------|
| `iscsi_service_name`  | `vars/common.yml`        | The service name to start for iSCSI. (`"iscsid"`)                              |
| `iscsi_pkg_name`      | `vars/<OS_Family>.yml`   | The target package to locate and install (`"open-iscsi"` or `"iscsi-initiator-utils"`). |
| `nfs_pkg_name`        | `vars/<OS_Family>.yml`   | The target package to locate and install (`"nfs-common"` or `"nfs-utils"`).    |

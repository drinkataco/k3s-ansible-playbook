# Build a File Server and Kubernetes Cluster

![Ansible](https://github.com/drinkataco/home-server-ansible/actions/workflows/ansible-lint.yaml/badge.svg)

Build a high availability kubernetes cluster on your Raspberry Pi computers with K3s, as well as managing volume mounts and exports.

<!-- vim-md-toc format=bullets ignore=^TODO$ -->
* [Requirements](#requirements)
* [Usage](#usage)
  * [Getting Started](#getting-started)
  * [Deploying](#deploying)
  * [Post Deployment](#post-deployment)
* [Custom Kernel](#custom-kernel)
* [Next Steps](#next-steps)
<!-- vim-md-toc END -->

## Requirements

It is recommended that you have at least 1x Pi 4, although it is recommended to use a Pi 5 with an [SSD or NVMe for boot](https://www.makeuseof.com/boot-raspberry-pi-4-via-ssd-network/).

Before running this playbook it is also assumed that:

- You have ansible installed on your provisioning machine
- Has your SSH Public key for passwordless SSH
- Has the latest version of Raspberry Pi OS (recommended Lite 64-bit version)
- (optional) Have at least one harddisk connected to your Pi

## Usage

### Getting Started

First, we need to create inventory for the playbook.

```bash
cp -Rv ./inventory/example ./inventory/your-cluster
```

In this newly created directory we need to edit the `./inventory/your-cluster/hosts.ini` file to assign roles to devices. You can either use hostnames, or IP addresses. By adding multiple master hosts you can enable a high availability cluster to be provisioned.

In the `./inventory/your-cluster/group_vars/` directory exists two files containing host specific configuration.

Some useful variables to modify:

- all.yaml
    - `ansible_user` - `pi` is the default username for Raspberry Pi OS
    - `k3s_install_args` - anything to pass to the `INSTALL_K3S_EXEC` variable when installing the server, for example `--disable=traefik` if you wanted to manage traefik outside of k3s. Consult [K3s documentation](https://docs.k3s.io/installation/configuration) for more on this value.
- fileserver.yaml
    - `mounts` - these are any drives you want to mount to the Pi
    - `export_dirs` - directories to export with NFS
- k3s_cluster.yaml
    - `labels` - custom labels to add to nodes
    - `taints` - taints to add to nodes

It is worth looking at the variable files themselves to see all possible config and values, which includes a lot of documentation in the comments.

### Deploying

1. Install ansible module requirements with `ansible-galaxy collection install -r requirements.yaml`
1. Install python dependencies argcomplete / pyyaml

You can then deploy your cluster using the following command

```bash
ansible-playbook -i inventory/your-cluster install.yaml
```

Alternatively, by replacing `install.yaml` with `reset.yaml` you can reset all ansible commands

### Post Deployment

You'll probably want to grab your kubeconfig file from the server so that you can control your cluster from your machine.

If you will only manage this cluster, you can simply use `scp pi@master:.kube/config ~/.kube/config`. Or, to manage multiple you can:

1. Download the kubeconfig file for your cluster (where `master` is the master node)
    ```bash
    scp pi@master:.kube/config ~/.kube/pi-cluster
    ```

1. Change your `KUBECONFIG` environment variable to discover the new config
    ```bash
    export KUBECONFIG="${HOME}/.kube/config:${HOME}/.kube/pi-cluster
    ```

1. You can either place this `KUBECONFIG` export in your `.zshrc` or `.bashrc` file, or you could merge and flatten the files with the following:
    ```bash
    kubectl config view --flatten > /tmp/config
    mv /tmp/config "${HOME}/.kube/config"
    ```

From your host machine, run the following to verify the nodes are available and k3s has been set up correctly

```bash
$ kubectl get nodes
NAME        STATUS   ROLES                  AGE     VERSION
mccartney   Ready    control-plane,master   3m34s   v1.27.4+k3s1
lennon      Ready    <none>                 2m20s   v1.27.4+k3s1
harrison    Ready    <none>                 2m20s   v1.27.4+k3s1
starr       Ready    <none>                 2m36s   v1.27.4+k3s1
```

## Custom Kernel

Some workloads require kernel features not present in the default Raspberry Pi OS kernel. Two known cases for this cluster:

- **48-bit virtual address space (Pi 4 only)** — the default Pi 4 kernel uses a 39-bit VA space. Certain memory allocators (including TCMalloc, used by Cilium Envoy) assume a minimum 48-bit VA and will crash on 39-bit kernels. Compiling with `CONFIG_ARM64_VA_BITS_48=y` resolves this.
- **GENEVE tunnel support** — Cilium can use GENEVE as its overlay tunnel protocol, but `CONFIG_GENEVE` is not enabled in the default kernel.

The `compile-kernel.yaml` playbook handles building and deploying a custom kernel. It compiles on a designated build host (to avoid doing a 20-minute compile on every node) and deploys the resulting `.deb` package to target nodes. Builds are skipped when the cached kernel is already up to date, and deployments are skipped when the running kernel matches the build hash.

### Host groups

Add hosts to the relevant groups in your inventory:

| Group | Purpose |
|---|---|
| `pi4_kernel_builder` | Node that compiles the Pi 4 kernel (a Pi 5 is ideal for speed) |
| `pi4_kernel` | Pi 4 nodes to deploy the compiled kernel to |
| `pi5_kernel_builder` | Node that compiles the Pi 5 kernel |
| `pi5_kernel` | Pi 5 nodes to deploy the compiled kernel to |

### Running

```bash
ansible-playbook -i inventory/your-cluster compile-kernel.yaml
```

The playbook runs the build and deploy phases in sequence. You can safely re-run it at any time — if the kernel cache is current and the target nodes are already running the latest build, all tasks will be skipped.

## Next Steps

Check out the sister project [home-server-kubernetes](https://github.com/drinkataco/home-server-kubernetes) for home server related resources to deploy to your cluster.



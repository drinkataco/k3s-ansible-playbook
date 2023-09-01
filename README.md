# Build a File Server and Kubernetes Cluster

Build a server cluster on your Raspberry Pi (singular or plural), as well as managing volume mounts and exports.

<!-- vim-md-toc format=bullets ignore=^TODO$ -->
* [Requirements](#requirements)
* [Usage](#usage)
  * [Getting Started](#getting-started)
  * [Deploying](#deploying)
  * [Post Deployment](#post-deployment)
<!-- vim-md-toc END -->

## Requirements

It is recommended that you have at least 1x Raspberry Pi 3b, although it is recommended to use a Pi 4 with an [SSD for boot](https://www.makeuseof.com/boot-raspberry-pi-4-via-ssd-network/).

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

Once copied, you should edit the hosts.ini file to change the hosts to yours (by hostname or IP address).

It is worth to modify the variables if needed within the group_vars directory. It is worth having a look at all the variables, but here are the ones most probable of adjustment:

- [group_vars/all.yaml](./inventory/example/group_vars/all.yaml)
    - `ansible_user` - `pi` is the default username for Raspberry Pi OS
- [group_vars/fileserver.yaml](./inventory/example/group_vars/fileserver.yaml)
    - `mounts` - these are any drives you want to mount to the Pi
    - `binds` - these are cross-mounts that allow us to export with nfs in a predictable format from the mounted drives without having to adhere to a particular format at source.
    - `export_dirs` - directries to export with NFS

### Deploying

You can deploy your cluster using the following command

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

To check your drives are available over NFS, use `showmount -e` on the machine, or `showmount -e fileserverhostname` on an external machine on the same network.

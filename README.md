# Build a Kubernetes cluster using k3s via Ansible

Author: <https://github.com/itwars>

## K3s Ansible Playbook

Build a Kubernetes cluster using Ansible with k3s. The goal is easily install a Kubernetes cluster on machines running:

- [ ] Debian
- [X] Ubuntu
- [ ] CentOS

on processor architecture:

- [ ] x64
- [X] arm64
- [ ] armhf

## System requirements

Deployment environment must have Ansible 2.4.0+
Master and nodes must have passwordless SSH access

## Usage

First create a new directory based on the `sample` directory within the `inventory` directory:

```bash
$ cp -R inventory/sample inventory/pi-cluster
...
```

Second, edit `inventory/pi-cluster/hosts.ini` to match the system information gathered above. For example:

```bash
[master]
192.16.35.12

[node]
192.16.35.[10:11]

[k3s_cluster:children]
master
node
```

If needed, you can also edit `inventory/pi-cluster/group_vars/all.yml` to match your environment.
For instance, the `ansible_user` variable should probably be changed, as well as `ansible_sudo_pass` if needed.

Start provisioning of the cluster using the following command:

```bash
$ ansible-playbook site.yml -i inventory/pi-cluster/hosts.ini --extra-vars "ansible_sudo_pass=SPECIFY_PWD_HERE"
...
```

## Kubeconfig

To get access to your **Kubernetes** cluster just

```bash
$ scp <USER>@master_ip:~/.kube/config ~/.kube/config
...
```

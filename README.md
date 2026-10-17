# Kubernetes Cluster Setup Guide

Ansible role + a single playbook that provisions a Kubernetes cluster (1 control-plane node + any number of worker nodes) on Ubuntu Server in minutes.

It installs Docker Engine + cri-dockerd as the container runtime, kubeadm/kubelet/kubectl, initializes the control plane and installs Calico as the CNI.

## TLDR

1. Create Ubuntu Server machines (1 master, N workers).
2. Edit `inventory.yml` — only the host address (`ansible_host`) is required.
3. Run everything with one command:

```bash
ansible-playbook setupK8sCluster.ansible.yaml -i inventory.yml
```

4. Join worker nodes (manual, see [Join your worker nodes](#join-your-worker-nodes)).

## Requirements

### On the machines (nodes)

- Ubuntu Server (tested on 22.04 / 24.04 / 26.04 "Resolute Raccoon").
- Minimum 2 GB RAM and 2 CPUs per node.
- SSH access with a user that can `sudo` (passwordless `sudo` recommended — otherwise set `ansible_become_password`).
- `python3` present (installed by default on Ubuntu).
- [Required Kubernetes ports](https://kubernetes.io/docs/reference/networking/ports-and-protocols/) open between the nodes; all nodes must be able to reach each other.
- A fresh machine is best — the role disables swap and manages Docker/cri-dockerd system-wide.

### On the controller (where you run Ansible)

- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) installed.

## Inventory (`inventory.yml`)

The **only required variable is `ansible_host`** (the address/hostname of the node). Everything else is optional and can be omitted if you connect with an SSH key and use passwordless `sudo`:

```yaml
myhosts:
  hosts:
    master:
      ansible_host: 10.0.0.10
    worker1:
      ansible_host: 10.0.0.11
    worker2:
      ansible_host: 10.0.0.12
```

- **The control-plane host must be named `master`** — the playbook targets it by that name. Worker host names are free-form.
- Optional variables (add only if needed):
  - `ansible_user` — SSH user, if it differs from your local user.
  - `ansible_port` — SSH port, if not 22.
  - `ansible_password` / `ansible_ssh_pass` — SSH password, if not using keys.
  - `ansible_become_password` — sudo password, if sudo is not passwordless.

You do **not** need `ansible_connection: ssh` (it is the default) or the other fields that appeared in older versions of this inventory.

## Run the playbook

```bash
ansible-playbook setupK8sCluster.ansible.yaml -i inventory.yml
```

One invocation runs everything in the correct order.

## What the playbook does

**Play 1 — all nodes (`myhosts`):**

| Order | Task file | What it does |
|---|---|---|
| 1 | `docker.yml` | Installs Docker Engine + adds the current user to the `docker` group |
| 2 | `cri_dockerd.yml` | Downloads and installs cri-dockerd |
| 3 | `disable_swap.yml` | Disables swap (active swap, `/etc/fstab`, and zram) and verifies it |
| 4 | `cgroups.yml` | Sets the `systemd` cgroup driver for cri-dockerd and Docker |
| 5 | `kube_tools.yml` | Installs `kubeadm`, `kubelet`, `kubectl` and holds their versions |

**Play 2 — control plane (`master`):**

| Order | Task file | What it does |
|---|---|---|
| 6 | `init_cluster.yml` | `kubeadm init` (skipped if the cluster already exists) |
| 7 | `after_init.yml` | Fixes kubeconfig ownership and exports `KUBECONFIG` |
| 8 | `cni.yml` | Installs the Calico CNI |

## Versions and running parts of the role

All pinned versions live in `roles/k8sCluster/defaults/main.yml`:

```yaml
cri_dockerd_version: "0.4.9"
cri_dockerd_arch: amd64
kubernetes_version: "v1.37"
calico_version: "v3.33.0"
```

Bump a version there and re-run (or override on the command line with `-e kubernetes_version=v1.37`).

Each step is tagged, so you can run a single part:

```bash
ansible-playbook setupK8sCluster.ansible.yaml -i inventory.yml --tags cni
```

Tags: `docker`, `cri-dockerd`, `swap`, `cgroups`, `kube-tools`, `init`, `after-init`, `cni`.

## Repository layout

```
setupK8sCluster.ansible.yaml   # the single entry-point playbook
inventory.yml                  # node list (only ansible_host required)
roles/k8sCluster/
  defaults/main.yml            # pinned versions
  files/                       # dockerEngine.sh, daemon.json
  tasks/                       # one file per step (see the table above)
```

## Make sure everything works

SSH to the master node and check:

```bash
kubectl get nodes
kubectl get pods -A
```

The master should be `Ready` and all pods should eventually be `Running`. Pods can take a minute to pull images.

## Join your worker nodes

The role prepares the worker nodes (runtime, kubelet, etc.) but does **not** join them — run the join manually on each worker:

1. On the **master**, print the join command:

```bash
kubeadm token create --print-join-command
```

2. On **each worker**, run the printed command with `sudo` **and** append the cri-dockerd socket:

```bash
sudo kubeadm join <control-plane-host>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --cri-socket /var/run/cri-dockerd.sock
```

Example:

```bash
sudo kubeadm join 192.168.50.5:6443 --token gnwckma1dagi6a0 --discovery-token-ca-cert-hash sha256:db871b6b97101f508304894aa1e28236db7e529c7fe64887 --cri-socket /var/run/cri-dockerd.sock
```

Then verify on the master with `kubectl get nodes`.

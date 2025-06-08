# Kubernetes Cluster Setup Guide

This README provides a step-by-step guide to setup and configure Kubernetes (K8s) cluster in minutes.

## TLDR

If you are familiar with Ansible u can do following steps:

1. Setup ubuntu-server mashines with the amount u need. (In that repo setup only for 1 master node and any amount of worker nodes) 

2. Configure inventory file, it should contain fields such as "ansible_host", "ansible_user", "ansible_password", "ansible_become_password", "ansible_ssh_pass", and other if needed, look at [ansible inventory docs](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html) for reference.

3. Run playbooks in following order:

   3.1 installDockerEngine.ansible.yaml

   3.2 installCRI-Dokerd.ansible.yaml

   3.3 disableSwap.ansible.yaml

   3.4 configureCgroups.ansible.yaml

   3.5 installKubeadmKubeletKubectl.ansible.yaml

   3.6 initCluster.ansible.yaml

   3.7 afterInitSteps.ansible.yaml

   3.8 installCNI.ansible.yaml

After that u can join your worker nodes to your cluster.   

Use ```kubeadm token create --print-join-command``` on master to get join command.

Then go to worker and insert it (don't forget to put sudo at the start) but in the end also add ```--cri-socket /var/run/cri-dockerd.sock```.

Full command will look like this:

```sudo kubeadm join 192.168.50.5:6443 --token gnwckma1dagi6a0 --discovery-token-ca-cert-hash sha256:db871b6b97101f508304894aa1e28236db7e529c7fe64887 --cri-socket /var/run/cri-dockerd.sock```

# Full steps

## Prerequisites

- Ubuntu server (whatever amount you want).
- Minimum 2GB of RAM.
- At least 2 CPUs.
- 1 master node and any amount of worker nodes

## Short description of repo files

**binaries** - contains cri-dockerd binary file for communication between docker and k8s.

**configs** - contains config for for setting docker cgroupdriver to systemd. 

**playbooks** - ansible playbooks for setting up k8s cluster

**scripts** - contains script for docker engine installation

**inventory.yml** - ansible inventory file

## Edit inventory file

Edit inventory file so it contains needed info for connection to your nodes. Do only one master node and whatever amount of worker nodes.

The "master" name in the "hosts" used by playbook, so don't change it.

## Setup Network

After setting up inventory file, make sure u have a proper connection to hosts.

Also u need to be sure that [required ports](https://kubernetes.io/docs/reference/networking/ports-and-protocols/) are open for k8s communication between nodes. And be sure all nodes also can communicate between each other.

## Run playbooks

[Install ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) if you don't have it yet. 

To run playbooks use ```ansible-playbook <path-to-playbook> -i <path-to-inventory-file>``` 

Run Playbooks in following order:

1. installDockerEngine.ansible.yaml

2. installCRI-Dokerd.ansible.yaml

3. disableSwap.ansible.yaml

4. configureCgroups.ansible.yaml

5. installKubeadmKubeletKubectl.ansible.yaml

6. initCluster.ansible.yaml

7. afterInitSteps.ansible.yaml

8. installCNI.ansible.yaml

## Make sure everything works!

You can ssh to master node and check that everything works using command ```kubectl get nodes``` and ```kubectl get pods -A```. You should see that node is "Ready" and all pods are running succesfully. If not, wait a little bit, sometimes pods needs time to pull images and other processes.

## Join your worker nodes

After you can join your worker nodes to your cluster.   

Use ```kubeadm token create --print-join-command``` on master to get join command.

Then go to worker and insert it but in the end also add ```--cri-socket /var/run/cri-dockerd.sock```.

Full command will look like this:

```sudo kubeadm join 192.168.50.5:6443 --token gnwckma1dagi6a0 --discovery-token-ca-cert-hash sha256:db871b6b97101f508304894aa1e28236db7e529c7fe64887 --cri-socket /var/run/cri-dockerd.sock```
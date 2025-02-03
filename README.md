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

Then go to worker and insert it but in the end also add ```--cri-socket /var/run/cri-dockerd.sock```.

Full command will look like this:

```sudo kubeadm join 192.168.50.5:6443 --token gnwckma1dagi6a0 --discovery-token-ca-cert-hash sha256:db871b6b97101f508304894aa1e28236db7e529c7fe64887 --cri-socket /var/run/cri-dockerd.sock```




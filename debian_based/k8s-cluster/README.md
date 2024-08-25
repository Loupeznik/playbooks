# Self-hosted k8s setup playbook

This playbook installs a k8s cluster on debian based systems. Inventory file is preset for two nodes. Fill the inventory file with correct credentials (login and ssh keys) and IPs. It is intended to run on Debian 12.
In case of use with another distro/version, some package repos may need to be changed.

```shell
ansible-playbook setup-k8s.yaml -i inventory.yaml
```

The playbook will deploy:

- k8s v1.31 (kubelet, kubeadm, kubectl)
- Docker
- [cri-dockerd](https://github.com/Mirantis/cri-dockerd)
- Calico CNI plugin
- [Longhorn](https://github.com/longhorn/longhorn) for storage

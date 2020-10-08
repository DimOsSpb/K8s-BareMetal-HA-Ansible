# K8s Bare-Metal HA Load Balaner <!-- omit in toc -->
- [1. Цель проекта](#1-цель-проекта)
- [2. Необходимые условия для реализации проекта](#2-необходимые-условия-для-реализации-проекта)
- [3. Особенности реализации](#3-особенности-реализации)
  - [3.1. Автоматизация](#31-автоматизация)
  - [3.2. K8s](#32-k8s)
    - [3.2.1. Отказ от установки __Docker__ в пользу __Сontainerd__](#321-отказ-от-установки-docker-в-пользу-сontainerd)
- [4. Virtual IP / KeepaliveD configuration](#4-virtual-ip--keepalived-configuration)
- [5. Usage example](#5-usage-example)
  - [5.1. For debug state of keepalived, on LB host:](#51-for-debug-state-of-keepalived-on-lb-host)
  - [5.2. Build cluster](#52-build-cluster)
    - [5.2.1. KubeAdm](#521-kubeadm)
  - [5.3. After first master node initialized (kubeadm init...)](#53-after-first-master-node-initialized-kubeadm-init)
- [6. Links to information sources](#6-links-to-information-sources)

## 1. Цель проекта

  Реализация подхода - "Инфраструктура как код", на примере создания on-premise отказоустойчивого кластера K8S с работающим сервисом.




## 2. Необходимые условия для реализации проекта

  * 6 хостов подходящие к минимальным требованиям для кластера K8s. [см. Kubeadm's minimum requirements](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin).
  * Базовый дистрибутив хостов - Debian 10+
  * Установленый python (см. требования к Ansible)
  * 
  
## 3. Особенности реализации

### 3.1. Автоматизация

  Для автоматизации, и переносимости все действия производятся через Ansible.
### 3.2. K8s
#### 3.2.1. Отказ от установки __Docker__ в пользу __Сontainerd__ 
  
  [предпосылки](https://habr.com/ru/company/flant/blog/414875/) ([original](https://kubernetes.io/blog/2018/05/24/kubernetes-containerd-integration-goes-ga/)) 

## 4. LoadBalancing плоскости управления

[HA LB - Virtual IP / KeepaliveD configuration](./roles/HA-LoadBalancing/README.MD)

### 5.2. Build cluster
#### 5.2.1. KubeAdm

    sudo kubeadm init --control-plane-endpoint 10.10.4.100:6443 --apiserver-bind-port 6444 --upload-certs --cri-socket /run/containerd/containerd.sock

### 5.3. After first master node initialized (kubeadm init...) 

    To start using your cluster, you need to run the following as a regular user:

      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config

## 6. Links to information sources
  * [K8s High Availability Considerations](https://github.com/kubernetes/kubeadm/blob/master/docs/ha-considerations.md#keepalived-and-haproxy)
  * [Containerd CRICTL User Guide](https://github.com/containerd/cri/blob/master/docs/crictl.md#install-crictl)
  * [CNI - the Container Network Interface](https://github.com/containernetworking/cni#how-do-i-use-cni)
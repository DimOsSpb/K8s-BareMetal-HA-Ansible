# K8s Bare-Metal HA Load Balanser <!-- omit in toc -->
- [1. Цель проекта](#1-цель-проекта)
- [2. Необходимые условия для реализации проекта](#2-необходимые-условия-для-реализации-проекта)
- [3. Особенности реализации](#3-особенности-реализации)
  - [3.1. Отказ от установки __Docker__ в пользу __Сontainerd__](#31-отказ-от-установки-docker-в-пользу-сontainerd)
  - [3.2. Выбор CNI (Flannel vs Calico vs Cilium)](#32-выбор-cni-flannel-vs-calico-vs-cilium)
    - [3.2.1. __CNI Flannel__](#321-cni-flannel)
    - [3.2.2. __CNI Calico__](#322-cni-calico)
    - [3.2.3. __CNI Cilium__](#323-cni-cilium)
  - [3.3. Build cluster](#33-build-cluster)
    - [Инициализация плоскости управления (первый мастер в кластере)](#инициализация-плоскости-управления-первый-мастер-в-кластере)
    - [Using kubeadm init with a configuration file](#using-kubeadm-init-with-a-configuration-file)
  - [3.4. After first master node initialized (kubeadm init...)](#34-after-first-master-node-initialized-kubeadm-init)
  - [Безопастность](#безопастность)
    - [TLS](#tls)
  - [ETCD](#etcd)
    - [Диагностика](#диагностика)
    - [Резервирование](#резервирование)
  - [Dashboard](#dashboard)
- [Реализуем сервисы в кластере](#реализуем-сервисы-в-кластере)
  - [Диагностика, сбор статистики...](#диагностика-сбор-статистики)
- [4. Links to information sources](#4-links-to-information-sources)

## 1. Цель проекта

  Реализация подхода - "Инфраструктура как код", на примере создания on-premise отказоустойчивого кластера K8S с работающим сервисом.




## 2. Необходимые условия для реализации проекта

  * 6 хостов подходящие к минимальным требованиям для кластера K8s. [см. Kubeadm's minimum requirements](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin).
  * Базовый дистрибутив хостов - Debian 10+
  * Установленый python (см. требования к Ansible)
  * С управляющего хоста, должен быть настроен доступ к целевым системам по ssh ключу (требование Ansible)
  
## 3. Особенности реализации

  Хосты - это некое железо (barremetal on-premise) или виртуальные машины с установленной os, отвечающее минимальным необходимым требованиям для будущего проекта. Операционная система должна быть устанавлена практически автоматчески (pxe, preseed).    

  Для автоматизации, и переносимости все действия производятся через Ansible с использованием [__индемпотентных__](https://ru.wikipedia.org/wiki/%D0%98%D0%B4%D0%B5%D0%BC%D0%BF%D0%BE%D1%82%D0%B5%D0%BD%D1%82%D0%BD%D0%BE%D1%81%D1%82%D1%8C) плейбуков и ролей.

  Плейбуки:
  
  - __InitCluster__ - Основной плейбук, приводит кластер (создает, если его нет) к заданному состоянию.
  
  Роли в проекте:

   - [K8sHost](./roles/K8sHost/README.MD) 
   - [K8sAdminHost](./roles/K8sAdminHost/README.MD) 
   - [K8sHost](./roles/K8sHost/README.MD) 
   - [HA-LoadBalancing](./roles/HA-LoadBalancing/README.MD)

### 3.1. Отказ от установки __Docker__ в пользу __Сontainerd__ 
  
  [предпосылки](https://habr.com/ru/company/flant/blog/414875/) ([original](https://kubernetes.io/blog/2018/05/24/kubernetes-containerd-integration-goes-ga/)) 

  Т.о. Докер мы не ставим, а ставим только containerd. Причем на всех узлах кластера, на плоскость управления то-же, т.к. службы k8s работают на контейнерах. 
  [Важно следить за совместимостью версий](https://github.com/containerd/cri#support-metrics) (_[Btw, containerd will switch from deamon-level config to per-pod config with v2 runtime. But we're still on containerd 1.2.x, and even containerd 1.3 still uses v1 as a default runtime](https://github.com/containerd/containerd/blob/36cf5b690dcc00ff0f34ff7799209050c3d0c59a/releases/v1.3.0.toml#L54)_)

  На данный момент ([см. setup/production-environment/container-runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#systemd)) для использования драйвера systemd в файле настройки containerd /etc/containerd/config.toml надо установить:

    [plugins.cri]
      systemd_cgroup = true

  а пунктом ранее

      # Configure containerd
    sudo mkdir -p /etc/containerd
    sudo containerd config default > /etc/containerd/config.toml

  > В последних релизах containerd прописывает в config.toml первой строкой "version = 2" - Пока это надо удалять т.к. сервис containerd не работает с RuntimeApiVersion:v1alpha2. Данные действия прописаны в роли k8sHost.
  

  Для проверки среды используется crictl, т.е. на узлах можно проверить

    @K8sM1:~$ sudo crictl version
    Version:  0.1.0
    RuntimeName:  containerd
    RuntimeVersion:  1.2.13
    RuntimeApiVersion:  v1alpha2
  [General usage CriCtl](https://kubernetes.io/docs/tasks/debug-application-cluster/crictl/)
 

  По умолчанию k8s ориентируется на Docker. Чтобы просмотреть или отредактировать текущую конфигурацию /etc/crictl.yaml.

    cat /etc/crictl.yaml
    runtime-endpoint: unix:///var/run/dockershim.sock
    image-endpoint: unix:///var/run/dockershim.sock
    timeout: 10
    debug: true

  Приводим к виду:

    cat /etc/crictl.yaml
    runtime-endpoint: unix:///run/containerd/containerd.sock
    image-endpoint: unix:///run/containerd/containerd.sock
    timeout: 10
    debug: false
  
    sudo systemctl daemon-reload
    sudo systemctl start containerd
    sudo systemctl start kubelet

  В итоге, вот пример вывода работающих контейнеров на узле плоскости управления, на инициализированным мастере k8s
  
    @K8sM1:~$ sudo crictl ps
    CONTAINER ID        IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID
    b363dba3ab01d       d373dd5a8593a       27 hours ago        Running             kube-proxy                2                   e2d7e6c94ed63
    ffdcc33c3f88c       bfe3a36ebd252       27 hours ago        Running             coredns                   2                   84804ddf481df
    c315dd6859460       bfe3a36ebd252       27 hours ago        Running             coredns                   2                   4c54afc32d6a8
    dfe081ddbe097       8603821e1a7a5       27 hours ago        Running             kube-controller-manager   2                   e2840753f5d5f
    2d142ad7ed733       2f32d66b884f8       27 hours ago        Running             kube-scheduler            2                   89711eef7d30d
    cca1c234eee3d       607331163122e       27 hours ago        Running             kube-apiserver            2                   d0d5e7fce805b
    7763b9c0ea494       0369cf4303ffd       27 hours ago        Running             etcd                      2                   74cc8b296faeb    
  
  Для диагностики см. containerd service logs

    [root@master-1 ~]# journalctl  -xe -f -u containerd

### 3.2. Выбор CNI (Flannel vs Calico vs Cilium)

  Для связи подов между узлами нужен cni plugin
  
  Ниже выдержки из [Результаты бенчмарка сетевых плагинов Kubernetes (CNI)](https://habr.com/ru/company/southbridge/blog/448688/)
  
  >Мы были бы рады, если бы CNI автоматически определил MTU, а он влияет на производительность в разы!!, исходя из настройки адаптеров. Однако с этим справились только Cilium и Flannel
  нужно настроить MTU для Calico, Canal, Kube-router и WeaveNet для оптимальной производительности.
  
  >Что же касается реализации сетевой политики, то здесь преуспели Calico, Canal, Cilium и WeaveNet, в которых можно настраивать правила Ingress и Egress. Для Kube-router есть правила только для Ingress, а у Flannel вообще нет сетевых политик.
  
  >Все CNI хорошо показали себя в бенчмарке по TCP. CNI с шифрованием сильно отстают, потому что шифрование — вещь дорогая.

  >Потребление ЦП
  Calico, Canal, Flannel и Kube-router очень эффективно используют ЦП — всего на 2% больше, чем Kubernetes без CNI. Сильно отстает WeaveNet с лишними 5%, а за ним Cilium — 7%.


  Т.о. приоритетные CNI в зависимости от сценария:

  #### 3.2.1. __CNI Flannel__
  >Если в кластере есть узлы с небольшим количеством ресурсов (несколько ГБ оперативки, несколько ядер) и вам не нужны функции безопасности — выбирайте Flannel. Это один из самых экономичных CNI. И он совместим с самыми разными архитектурами (amd64, arm, arm64 и т. д.). К тому же это один из двух 

  #### 3.2.2. __CNI Calico__  
  >Calico - этот CNI широко используется в разных инструментах деплоя Kubernetes (Kops, Kubespray, Rancher и т. д.). Как и с WeaveNet, не забудьте настроить MTU в ConfigMap, если используете jumbo-кадры. Это многофункциональный инструмент, эффективный в плане потребления ресурсов, производительности и безопасности.

  #### 3.2.3. __CNI Cilium__
  >CNI, который может автоматически определить MTU, так что ничего настраивать не придется. У этого CNI очень активная команда, которая много работает над своим продуктом (функции, экономия ресурсов, производительность, безопасность, распределение по кластерам…), и у них очень интересные планы.
  Если нужно зашифровать сеть для безопасности, берите WeaveNet. Не забудьте указать размер MTU, если используете jumbo-кадры, и активировать шифрование, указав пароль через переменную окружения. Но о производительности лучше забыть — такова плата за шифрование.
  

  

### 3.3. Build cluster
> __[KubeAdm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)__

  Создание кластера разбито на этапы:
  #### [Инициализация плоскости управления](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/) (первый мастер в кластере)

      sudo kubeadm init
      --control-plane-endpoint {{ k8s_control_plane_endpoint_vip }}:{{ k8s_control_plane_endpoint_port }}
      --apiserver-bind-port {{ k8s_apiserver_bind_port }}
      --upload-certs
      --cri-socket /run/containerd/containerd.sock

  #### [Using kubeadm init with a configuration file](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/#config-file)

  > [Для получения дополнительной информации о полях и использовании конфигурации...](https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta2)

### 3.4. After first master node initialized (kubeadm init...) 

    To start using your cluster, you need to run the following as a regular user:

      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config
### Безопастность
#### TLS
  [K8s TLS bootstrapping](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/)
### ETCD 
#### Диагностика
#### Резервирование

### Dashboard
## Реализуем сервисы в кластере
### Диагностика, сбор статистики...

## 4. Links to information sources
  * [K8s High Availability Considerations](https://github.com/kubernetes/kubeadm/blob/master/docs/ha-considerations.md#keepalived-and-haproxy)
  * [Containerd CRICTL User Guide](https://github.com/containerd/cri/blob/master/docs/crictl.md#install-crictl)
  * [CNI - the Container Network Interface](https://github.com/containernetworking/cni#how-do-i-use-cni)
  * [Как pod в Kubernetes получает IP-адрес](https://habr.com/ru/company/flant/blog/521406/)

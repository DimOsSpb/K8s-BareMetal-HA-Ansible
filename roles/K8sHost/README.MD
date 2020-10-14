# K8s Host Role

Приводит узлы к состоянию для начала установки (инициализации) кластера

## Пример использования 

In ansible playbook: 

    - hosts: k8s                # На всех узлах кластера см. hosts.yml
      order: sorted
      become: true

      roles:
        - K8sHost


# K8s Admin Host Role

Приводит хост администрирования к настроенному состоянию. Устанавливает если надо утилиту администрирования и зная узлы кластера (из hosts.yml), при наличии настроенного кластера, забирает его конфигурацию для kubectl 

## Пример использования 

In ansible playbook: 

    - hosts: adm                
      order: sorted
      become: true

      roles:
        - K8sAdmHost


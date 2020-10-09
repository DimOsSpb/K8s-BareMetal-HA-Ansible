# K8s Bare-Metal HA Load Balanser

## Virtual IP / KeepaliveD configuration

## Usage example 

In ansible playbook: 

    - hosts: k8s-masters
      order: sorted
      become: true

      roles:
        - HA-LoadBalancing

### For debug state of keepalived, on LB host:
    sudo systemctl stop keepalived
    sudo keepalived -f /etc/keepalived/keepalived.conf --dont-fork --log-console

### Проверка HAProxy

    http://10.10.4.100:8080/haproxy?stats#kube_api_server


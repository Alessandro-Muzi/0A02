

    ## NFTABLES


```yaml

# roles/nftables/defaults/main.yml


nftables_wan_iface: "enp0s3"
nftables_lan_iface: "enp0s8"
nftables_zabbix_ip: "10.0.0.20"

nftables_koha_port: 8080
nftables_headscale_port: 8443
nftables_wireguard_port: 51820

```



# roles/nftables/handlers/main.yml

```yaml

- name: Ricarica nftables
  ansible.builtin.systemd:
    name: nftables
    state: restarted

```




# roles/nftables/tasks/main.yml

```yaml

- name: Installa nftables
  ansible.builtin.apt:
    name: nftables
    state: present
    update_cache: true

- name: Abilita l'IP forwarding nel kernel
  ansible.posix.sysctl:
    name: net.ipv4.ip_forward
    value: '1'
    sysctl_set: true
    state: present
    reload: true

- name: Distribuisci il file di configurazione nftables
  ansible.builtin.template:
    src: nftables.conf.j2
    dest: /etc/nftables.conf
    owner: root
    group: root
    mode: '0640'
  notify: Ricarica nftables

- name: Abilita e avvia il servizio nftables
  ansible.builtin.systemd:
    name: nftables
    state: started
    enabled: true

```




# roles/nftables/templates/nftables.conf.j2


```bash

#!/usr/sbin/nft -f
flush ruleset

table ip filter {
    chain input {
        type filter hook input priority filter; policy drop;
        iifname "lo" accept
        ct state established,related accept
        tcp dport 22 accept
        tcp dport {{ nftables_koha_port }} accept
        tcp dport {{ nftables_headscale_port }} accept
        udp dport {{ nftables_wireguard_port }} accept # Porta Headscale/WireGuard
        icmp type echo-request accept
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
        ct state established,related accept
        iifname "{{ nftables_wan_iface }}" oifname "{{ nftables_lan_iface }}" ip daddr {{ nftables_zabbix_ip }} tcp dport 22 accept
        iifname "{{ nftables_lan_iface }}" oifname "{{ nftables_wan_iface }}" accept
    }
}

table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        oifname "{{ nftables_wan_iface }}" masquerade
    }
}


```


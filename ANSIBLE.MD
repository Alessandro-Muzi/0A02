
## ANSIBLE

Ogni volta che il playbook viene eseguito, Ansible, via SSH con supporto proxyjump per la rete dedicata a Zabbix,
applica i ruoli assegnati a ciascun gruppo in playbook.yml e, per ognuno, esegue in ordine i task: installazione pacchetti, generazione dei file di configurazione dai template (con le variabili risolte da group_vars),avvio/riavvio dei servizi tramite handler quando necessario.
Rilanciare il playbook su un'infrastruttura già configurata verifica e corregge eventuali scostamenti dallo stato desiderato, ripristinando la configurazione dei servizi.




```bash

mkdir -p koha-ansible/{group_vars/all,roles/{bind9/{defaults,files,handlers,tasks,templates},headscale/{defaults,files,handlers,tasks,templates},headscale-client/{defaults,handlers,tasks},koha/{defaults,files,handlers,tasks,templates},mariadb/{defaults,files,handlers,tasks,templates},nftables/{defaults,handlers,tasks,templates},openldap/{defaults,files,tasks,templates},zabbix-server/{defaults,handlers,tasks}}}

```

o volendo:

```bash
#!/bin/bash
standard=(defaults files handlers tasks templates)
minimal=(defaults handlers tasks)

mkdir -p koha-ansible/roles

for role in bind9 headscale koha mariadb; do
    for dir in "${standard[@]}"; do
        mkdir -p "koha-ansible/roles/$role/$dir"
    done
done

for role in headscale-client zabbix-server; do
    for dir in "${minimal[@]}"; do
        mkdir -p "koha-ansible/roles/$role/$dir"
    done
done
```

```txt
.
├── koha-ansible
│   ├── ansible.cfg
│   ├── group_vars
│   │   └── all
│   │       ├── vars.yml
│   │       └── vault.yml
│   ├── hosts
│   ├── output.log
│   ├── playbook.yml
│   └── roles
│       ├── bind9
│       │   ├── defaults
│       │   ├── files
│       │   ├── handlers
│       │   │   └── main.yml
│       │   ├── tasks
│       │   │   └── main.yml
│       │   └── templates
│       │       ├── db.192.168.1.j2
│       │       ├── db.koha.lab.j2
│       │       ├── named.conf.local.j2
│       │       └── named.conf.options.j2
│       ├── headscale
│       │   ├── defaults
│       │   │   └── main.yml
│       │   ├── files
│       │   ├── handlers
│       │   │   └── main.yml
│       │   ├── tasks
│       │   │   └── main.yml
│       │   └── templates
│       │       └── config.yaml.j2
│       ├── headscale-client
│       │   ├── defaults
│       │   │   └── main.yml
│       │   ├── handlers
│       │   │   └── main.yml
│       │   └── tasks
│       │       └── main.yml
│       ├── koha
│       │   ├── defaults
│       │   ├── files
│       │   ├── handlers
│       │   │   └── main.yml
│       │   ├── tasks
│       │   │   ├── instance.yml
│       │   │   └── main.yml
│       │   └── templates
│       ├── mariadb
│       │   ├── defaults
│       │   ├── files
│       │   ├── handlers
│       │   │   └── main.yml
│       │   ├── my-koha.cnf
│       │   ├── tasks
│       │   │   └── main.yml
│       │   └── templates
│       │       └── koha.cnf.j2
│       ├── nftables
│       │   ├── defaults
│       │   │   └── main.yml
│       │   ├── handlers
│       │   │   └── main.yml
│       │   ├── tasks
│       │   │   └── main.yml
│       │   └── templates
│       │       └── nftables.conf.j2
│       ├── openldap
│       │   ├── defaults
│       │   ├── files
│       │   ├── tasks
│       │   │   └── main.yml
│       │   └── templates
│       └── zabbix-server
│           ├── defaults
│           │   └── main.yml
│           ├── handlers
│           │   └── main.yml
│           └── tasks
│               └── main.yml
└── tree.txt

47 directories, 35 files
```


-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


/koha-ansible/ansible.cfg

```ini

[defaults]
inventory = hosts
host_key_checking = False
pipelining = True

```




/koha-ansible/playbook.yml

```yaml

- name: Configurazione Server Koha
  hosts: debian
  become: yes
  roles:
    - mariadb
    - headscale
    - bind9
    - openldap
    - koha

- name: Configurazione Server Zabbix
  hosts: zabbix
  become: yes
  roles:
    - zabbix-server
    - headscale-client

```



/koha-ansible/hosts

```ini

[debian]
SERVER ansible_host=192.168.1.4 ansible_user=root

[zabbix]
ZABBIX ansible_host=10.0.0.20 ansible_user=root ansible_ssh_common_args='-o ProxyJump=root@192.168.1.4'

```




/koha-ansible/group_vars/vars.yml

```yaml

ldap_organization: "Biblioteca di Alessandro"
ldap_host: "localhost" # O l'IP del server dove gira OpenLDAP
ldap_domain: "biblioteca.local"
ldap_base_dn: "dc=biblioteca,dc=local"
ldap_admin_dn: "cn=admin,dc=biblioteca,dc=local"
ldap_admin_pass: "1234"
zabbix_db_password: "1234"
zabbix_db_name: zabbix
zabbix_db_user: zabbix
koha_instance_name: "biblioteca"
ansible_python_interpreter: /usr/bin/python3.13
# group_vars/all.yml o simile
headscale_server_url: "http://192.168.1.4:8443"
headscale_login_server: "http://192.168.1.4:8443"
headscale_listen_port: 8443
headscale_preauth_key: "{{ vault_headscale_preauth_key }}"
headscale_version: "0.29.2"

```




group_vars/all/ contiene le variabili condivise da tutti gli host: vars.yml per la configurazione in chiaro, vault.yml per i secret (password database, preauth key Headscale), cifrato tramite Ansible Vault e sicuro anche se pubblicato nel repository.

```


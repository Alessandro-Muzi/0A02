
##ANSIBLE

/koha-ansible/ansible.cfg

```

[defaults]
inventory = inventory.ini
host_key_checking = False
pipelining = True

```




/koha-ansible/playbook.cfg

```

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

```

[debian]
SERVER ansible_host=192.168.1.4 ansible_user=root

[zabbix]
ZABBIX ansible_host=10.0.0.20 ansible_user=root ansible_ssh_common_args='-o ProxyJump=root@192.168.1.4'

```




/koha-ansible/group_vars/vars.yml

```

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




/koha-ansible/group_vars/vault.yml

```

$ANSIBLE_VAULT;1.1;AES256
64653561323430363461326162396662326234363338346633306134636636373338323065353433
6538363062663030633730616236363531396430376364330a303265383363656131386363383864
63306466646132616131663665623538626562363730343237643561643833333463623539613738
6263303132373331370a633733653462653133643661616363373163373634343433386166633439
64653935346333363531303939313765663338623639616264363835316536303332303330623631
65383635623234393933343933333534316535383066396462646162333130306265386163653936
33333562336233303966363563393138613133383338346133623161363931646466333661646362
37313265323532373030396661383237376333626430626237303736313635383738656566356163
38643863313036613039333564656330313964613566376637323061383730336335303339646365
66383635353436386361623835366238313165343437373531613537306437396239373564616335
34323963306439663062666635613537653236313861376532653161373161313165343164306436
34333163343932303837

```


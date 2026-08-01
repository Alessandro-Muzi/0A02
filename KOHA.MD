
## KOHA



```yaml

/koha-ansible/roles/koha/handlers/main.yml

- name: Riavviare Apache
  ansible.builtin.service:
    name: apache2
    state: restarted

```




/koha-ansible/roles/koha/tasks/instance.yml

```yaml

- name: Abilito il modulo rewrite di Apache
  community.general.apache2_module:
    name: rewrite
    state: present
  notify: Riavviare Apache

- name: Abilito il modulo cgi di Apache
  community.general.apache2_module:
    name: cgi
    state: present
  notify: Riavviare Apache

- name: Verifico se l'istanza Koha esiste
  ansible.builtin.command: koha-list --enabled
  register: koha_instances
  changed_when: false

- name: Creo l'istanza Koha
  ansible.builtin.command: "koha-create --create-db {{ koha_instance_name }}"
  when: koha_instance_name not in koha_instances.stdout
  vars:
    koha_instance_name: "biblioteca"
  notify: Riavviare Apache

- name: Abilito il sito Koha in Apache
  ansible.builtin.command: "a2ensite {{ koha_instance_name }}"
  args:
    creates: "/etc/apache2/sites-enabled/{{ koha_instance_name }}.conf"
  notify: Riavviare Apache


```




/koha-ansible/roles/koha/tasks/main.yml



```yaml                                                                              

- name: Mi assicuro che la directory dei keyring esista
  ansible.builtin.file:
    path: /etc/apt/keyrings
    state: directory
    mode: '0755'

- name: Scarico la chiave GPG di Koha
  ansible.builtin.get_url:
    url: https://debian.koha-community.org/koha/gpg.asc
    dest: /etc/apt/keyrings/koha-keyring.asc
    mode: '0644'

- name: Aggiungo il repository ufficiale di Koha
  ansible.builtin.apt_repository:
    repo: "deb [signed-by=/etc/apt/keyrings/koha-keyring.asc] https://debian.koha-community.org/koha stable main"
    state: present
    filename: koha
    update_cache: yes

- name: Aggiorno la cache di APT e installo Koha e le sue dipendenze
  ansible.builtin.apt:
    name:
      - koha-common
      - apache2
      - libdbd-mysql-perl
      - libnet-ldap-perl
      - libcrypt-eksblowfish-perl
      - libauthen-sasl-perl
    state: present
    update_cache: yes

- name: Includo i task per la creazione dell'istanza
  ansible.builtin.include_tasks: instance.yml

- name: Configuro l'autenticazione OpenLDAP in koha-conf.xml
  ansible.builtin.blockinfile:
    path: "/etc/koha/sites/{{ koha_instance_name }}/koha-conf.xml"
    marker: "<!-- {mark} ANSIBLE MANAGED BLOCK: LDAP -->"
    insertbefore: "</config>"
    block: |
      <useldapserver>1</useldapserver>
      <ldapserver id="ldapserver">
        <hostname>{{ ldap_host }}</hostname>
        <port>389</port>
        <base>ou=users,{{ ldap_base_dn }}</base>
        <user>{{ ldap_admin_dn }}</user>
        <pass>{{ ldap_admin_pass }}</pass>
        <replicate>1</replicate>
        <update>1</update>
        <mapping>
          <firstname    is="givenName"      ></firstname>
          <surname      is="sn"             ></surname>
          <address      is="postalAddress"  ></address>
          <email        is="mail"           ></email>
          <branchcode   is="branch"         >MAIN</branchcode>
          <categorycode is="employeetype"   >PT</categorycode>
          <userid       is="uid"            ></userid>
          <password     is="userPassword"   ></password>
        </mapping>
      </ldapserver>
  notify: Riavviare Apache

```



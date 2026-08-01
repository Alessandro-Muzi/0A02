----------------------------------------------------------------------------------
          ##OPENLDAP
----------------------------------------------------------------------------------

```

/koha-ansible/roles/openldap/tasks/main.yml


- name: Installo OpenLDAP e utility di gestione
  ansible.builtin.apt:
    name:
      - slapd
      - ldap-utils
    state: present
    update_cache: yes

- name: Genero l'hash della password amministratore
  ansible.builtin.command: "slappasswd -s {{ ldap_admin_pass }}"
  register: ldap_password_hash
  changed_when: false

- name: Creo il file LDIF temporaneo per la configurazione del database
  ansible.builtin.copy:
    dest: /tmp/db.ldif
    mode: '0600'
    content: |
      dn: olcDatabase={1}mdb,cn=config
      changetype: modify
      replace: olcSuffix
      olcSuffix: {{ ldap_base_dn }}
      -
      replace: olcRootDN
      olcRootDN: {{ ldap_admin_dn }}
      -
      replace: olcRootPW
      olcRootPW: {{ ldap_password_hash.stdout }}

- name: Verifico il valore attuale della configurazione del database
  ansible.builtin.command: >
    ldapsearch -Y EXTERNAL -H ldapi:/// -b "olcDatabase={1}mdb,cn=config"
    -LLL olcSuffix olcRootDN
  register: ldap_current_config
  changed_when: false

- name: Applico la configurazione iniziale del database OpenLDAP
  ansible.builtin.command: ldapmodify -Y EXTERNAL -H ldapi:/// -f /tmp/db.ldif
  when: >
    ldap_base_dn not in ldap_current_config.stdout or
    ldap_admin_dn not in ldap_current_config.stdout

- name: Creo la struttura base (Organizzazione e Gruppo Utenti)
  ansible.builtin.copy:
    dest: /tmp/structure.ldif
    mode: '0600'
    content: |
      dn: {{ ldap_base_dn }}
      objectClass: top
      objectClass: dcObject
      objectClass: organization
      o: {{ ldap_organization }}
      dc: {{ ldap_domain.split('.')[0] }}

      dn: ou=users,{{ ldap_base_dn }}
      objectClass: top
      objectClass: organizationalUnit
      ou: users

- name: Applico la struttura base nel database OpenLDAP
  ansible.builtin.command: "ldapadd -x -D {{ ldap_admin_dn }} -w {{ ldap_admin_pass }} -f /tmp/structure.ldif"
  register: ldap_add_result
  failed_when:
    - ldap_add_result.rc != 0
    - "'Already exists' not in ldap_add_result.stderr"
  changed_when: "'Already exists' not in ldap_add_result.stderr"

```
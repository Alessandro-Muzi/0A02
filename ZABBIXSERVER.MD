

## ZABBIX-SERVER



/koha-ansible/roles/zabbix-server/defaults/main.yml


```yml

zabbix_version: "7.0"
# Nome in codice della release Debian (es. "trixie", "bookworm") — verificata
# esistente su https://repo.zabbix.com/zabbix/{{ zabbix_version }}/debian/dists/
zabbix_debian_release: "trixie"

zabbix_db_name: zabbix
zabbix_db_user: zabbix
zabbix_db_password: "1234"

zabbix_server_listen_port: 10051
zabbix_frontend_timezone: "Europe/Rome"

# Se true, il ruolo importa lo schema SQL iniziale (solo al primo run)
zabbix_import_schema: true

```


/koha-ansible/roles/zabbix-server/handlers/main.yml

```yml

- name: Riavviare Zabbix server
  ansible.builtin.systemd:
    name: zabbix-server
    state: restarted

- name: Riavviare Apache
  ansible.builtin.systemd:
    name: apache2
    state: restarted

```



/koha-ansible/roles/zabbix-server/tasks/main.yml

```yml

- name: Installa dipendenze base (gnupg per repository esterni)
  ansible.builtin.apt:
    name: gnupg
    state: present
    update_cache: true

- name: Scarica la chiave GPG ufficiale di Zabbix
  ansible.builtin.get_url:
    url: "https://repo.zabbix.com/zabbix-official-repo.key"
    dest: "/usr/share/keyrings/zabbix-official-repo.key"
    mode: '0644'

- name: Aggiungi il repository APT di Zabbix
  ansible.builtin.apt_repository:
    repo: "deb [signed-by=/usr/share/keyrings/zabbix-official-repo.key] https://repo.zabbix.com/zabbix/{{ zabbix_version }}/debian {{ zabbix_debian_release }} main"
    # zabbix_debian_release: "trixie" — build ufficiale confermata esistente,
    # nonostante il pacchetto .deb per nome file non fosse trovabile (vedi troubleshooting)
    state: present
    filename: zabbix

- name: Aggiorna la cache apt dopo l'aggiunta del repository Zabbix
  ansible.builtin.apt:
    update_cache: true

- name: Installa MariaDB server
  ansible.builtin.apt:
    name:
      - mariadb-server
      - python3-pymysql
    state: present

- name: Abilita e avvia MariaDB
  ansible.builtin.systemd:
    name: mariadb
    state: started
    enabled: true

- name: Installa Zabbix server, frontend e agent
  ansible.builtin.apt:
    name:
      - zabbix-server-mysql
      - zabbix-frontend-php
      - zabbix-apache-conf
      - zabbix-sql-scripts
      - zabbix-agent2
    state: present

- name: Crea il database Zabbix
  community.mysql.mysql_db:
    name: "{{ zabbix_db_name }}"
    encoding: utf8mb4
    collation: utf8mb4_bin
    state: present
    login_unix_socket: /run/mysqld/mysqld.sock

- name: Crea l'utente MariaDB per Zabbix
  community.mysql.mysql_user:
    name: "{{ zabbix_db_user }}"
    password: "{{ zabbix_db_password }}"
    priv: "{{ zabbix_db_name }}.*:ALL"
    login_unix_socket: /run/mysqld/mysqld.sock
    state: present

- name: Verifica se lo schema Zabbix è già stato importato
  ansible.builtin.command: >
    mysql -u{{ zabbix_db_user }} -p{{ zabbix_db_password }} {{ zabbix_db_name }}
    -e "SHOW TABLES LIKE 'users';"
  register: zabbix_schema_check
  changed_when: false
  failed_when: false

- name: Importa lo schema SQL iniziale di Zabbix (solo al primo run)
  ansible.builtin.shell: >
    zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz |
    mysql -u{{ zabbix_db_user }} -p{{ zabbix_db_password }} {{ zabbix_db_name }}
  when:
    - zabbix_import_schema
    - "'users' not in zabbix_schema_check.stdout"
  register: zabbix_schema_import
  changed_when: zabbix_schema_import.rc == 0

- name: Configura DBPassword in zabbix_server.conf
  ansible.builtin.lineinfile:
    path: /etc/zabbix/zabbix_server.conf
    regexp: '^# ?DBPassword='
    line: "DBPassword={{ zabbix_db_password }}"
  notify: Riavviare Zabbix server

- name: Configura il timezone nel frontend PHP di Zabbix
  ansible.builtin.lineinfile:
    path: /etc/zabbix/apache.conf
    regexp: 'php_value date.timezone'
    line: "        php_value date.timezone {{ zabbix_frontend_timezone }}"
  notify: Riavviare Apache

- name: Abilita e avvia Zabbix server
  ansible.builtin.systemd:
    name: zabbix-server
    state: started
    enabled: true

- name: Abilita e avvia Zabbix agent2
  ansible.builtin.systemd:
    name: zabbix-agent2
    state: started
    enabled: true

- name: Riavvia Apache per applicare la configurazione Zabbix
  ansible.builtin.systemd:
    name: apache2
    state: restarted

```

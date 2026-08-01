
## MARIADB


```yaml

/koha-ansible/roles/mariadb/handlers/main.yml

- name: Riavvia MariaDB
  service:
    name: mariadb
    state: restarted

```


/koha-ansible/roles/mariadb/tasks/main.yml


```yaml

- name: Installo libreria Python per MySQL
  apt:
    name: python3-pymysql
    state: present

- name: Verifico che MariaDB sia installato
  ansible.builtin.apt:
    name: mariadb-server
    state: present

- name: Applico la configurazione ottimizzata per Koha
  template:
    src: koha.cnf.j2
    dest: /etc/mysql/mariadb.conf.d/koha.cnf
#ponte che collega il task all'handler
  notify: Riavvia MariaDB

- name: Creo il database per Koha
  mysql_db:
    name: kohaDB
    state: present

- name: Creo l'utente per Koha
  community.mysql.mysql_user:
    name: alessandro
    password: "1234"
    priv: 'kohaDB.*:ALL,GRANT'
    state: present
    host: localhost
    column_case_sensitive: true

```




/koha-ansible/roles/mariadb/templates/koha.cnf.j2

```ini

[mysqld]
# Impostazioni ottimizzate per Koha
innodb_buffer_pool_size = 512M  # Regola in base alla tua RAM
innodb_log_file_size = 128M
innodb_flush_log_at_trx_commit = 2
innodb_file_per_table = 1
max_connections = 100
collation-server = utf8mb4_unicode_ci
character-set-server = utf8mb4

```

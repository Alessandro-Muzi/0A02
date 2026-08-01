
## BIND9


/koha-ansible/roles/bind9/handlers/main.yml

```yaml

- name: riavvia bind9
  service:
    name: bind9
    state: restarted
    enabled: yes

```



/koha-ansible/roles/bind9/tasks/main.yml

```yaml

- name: Installo Bind9 e strumenti di utilità
  apt:
    name:
      - bind9
      - bind9utils
      - bind9-dnsutils
    state: present
    update_cache: yes

- name: Copio la configurazione principale (options)
  template:
    src: named.conf.options.j2
    dest: /etc/bind/named.conf.options
    owner: root
    group: bind
    mode: '0644'
  notify: riavvia bind9

- name: Copio la configurazione locale (zone)
  template:
    src: named.conf.local.j2
    dest: /etc/bind/named.conf.local
    owner: root
    group: bind
    mode: '0644'
  notify: riavvia bind9

- name: Copio il file della zona diretta
  template:
    src: db.koha.lab.j2
    dest: /etc/bind/db.koha.lab
    owner: root
    group: bind
    mode: '0644'
  notify: riavvia bind9

- name: Copio il file della zona inversa
  template:
    src: db.192.168.1.j2
    dest: /etc/bind/db.192.168.1
    owner: root
    group: bind
    mode: '0644'
  notify: riavvia bind9

- name: Verifica che la configurazione sia corretta
  command: named-checkconf
  changed_when: false

```


/koha-ansible/roles/bind9/templates/db.192.168.1.j2

```text

$TTL 604800
@   IN  SOA ns1.koha.lab. admin.koha.lab. (
            2026070401 ; Serial
            604800
            86400
            2419200
            604800 )

@       IN  NS  ns1.koha.lab.

; Mapping dell'host (usa l'ultimo ottetto dell'IP, es. 5)
5      IN  PTR ns1.koha.lab.
5      IN  PTR opac.koha.lab.
5      IN  PTR intranet.koha.lab.

```



/koha-ansible/roles/bind9/templates/db.koha.lab.j2

```text

$TTL 604800
@   IN  SOA ns1.koha.lab. admin.koha.lab. (
            2026070401 ; Serial
            604800     ; Refresh
            86400      ; Retry
            2419200    ; Expire
            604800 )   ; Negative Cache TTL

@       IN  NS  ns1.koha.lab.
ns1     IN  A   192.168.1.4
opac    IN  A   192.168.1.4
intranet IN A   192.168.1.4

```



/koha-ansible/roles/bind9/templates/named.conf.local.j2


```text

zone "koha.lab" {
    type master;
    file "/etc/bind/db.koha.lab";
};

// Sostituisci 1.168.192 con la tua subnet invertita (es. se la rete è 192.168.1.0, metti 1.168.192)
zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.1";
};

```



/koha-ansible/roles/bind9/templates/named.conf.options.j2

```text

options {
    directory "/var/cache/bind";

    // Imposta l'IP su cui Bind deve ascoltare (cambia 192.168.1.10 con l'IP reale di Debby Ann)
    listen-on { 127.0.0.1; 192.168.1.4; };

    // Sicurezza: solo le reti locali possono fare query
    allow-query { localhost; 192.168.1.0/24; };

    // Inoltra le query che non conosci a questi DNS
    forwarders {
        1.1.1.1;
        8.8.8.8;
    };

    dnssec-validation auto;
    listen-on-v6 { any; };
};

```

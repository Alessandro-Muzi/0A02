
          ## HEADSCALE



/koha-ansible/roles/headscale/defaults/main.yml

```yaml
# Dominio pubblico (o interno) su cui sarà raggiungibile Headscale
headscale_server_url: "https://192.168.1.4:8443"

# Porta di ascolto interna di Headscale 
headscale_listen_port: 8443

# Percorso base dei dati e della configurazione
headscale_config_dir: /etc/headscale
headscale_data_dir: /var/lib/headscale

# Range IP della tailnet (default Tailscale/Headscale)
headscale_ip_prefix_v4: "100.64.0.0/10"

# Nome utente/namespace di default da creare al provisioning
headscale_default_user: homelab

# Base domain per MagicDNS
headscale_base_domain: "lab.internal"

```


/koha-ansible/roles/headscale/handlers/main.yml

```yaml

- name: riavvia headscale
  ansible.builtin.systemd:
    name: headscale
    state: restarted

```


/koha-ansible/roles/headscale/tasks/main.yml


```yaml

- name: Scarica il pacchetto .deb di Headscale dalla release ufficiale GitHub
  ansible.builtin.get_url:
    url: "https://github.com/juanfont/headscale/releases/download/v{{ headscale_version }}/headscale_{{ headscale_version }}_linux_amd64.deb"
    dest: "/tmp/headscale_{{ headscale_version }}.deb"
    mode: '0644'
  register: headscale_download

- name: Installa il pacchetto Headscale
  ansible.builtin.apt:
    deb: "/tmp/headscale_{{ headscale_version }}.deb"
    state: present
  when: headscale_download is changed

- name: Crea le directory dati e configurazione se non esistono
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    owner: headscale
    group: headscale
    mode: '0750'
  loop:
    - "{{ headscale_config_dir }}"
    - "{{ headscale_data_dir }}"

- name: Distribuisci il file di configurazione config.yaml
  ansible.builtin.template:
    src: config.yaml.j2
    dest: "{{ headscale_config_dir }}/config.yaml"
    owner: headscale
    group: headscale
    mode: '0640'
  notify: riavvia headscale

- name: Abilita e avvia il servizio headscale
  ansible.builtin.systemd:
    name: headscale
    state: started
    enabled: true

- name: Verifica se l'utente di default esiste già
  ansible.builtin.command: "headscale users list --output json"
  register: headscale_users_list
  changed_when: false

- name: Crea l'utente/namespace di default nella tailnet
  ansible.builtin.command: "headscale users create {{ headscale_default_user }}"
  when: headscale_default_user not in headscale_users_list.stdout
  register: headscale_user_create
  changed_when: headscale_user_create.rc == 0

```



/koha-ansible/roles/headscale/templates/config.yaml.j2


```yaml

server_url: {{ headscale_server_url }}
listen_addr: 0.0.0.0:{{ headscale_listen_port }}  # NON 127.0.0.1: vedi troubleshooting "chain input"
metrics_listen_addr: 127.0.0.1:9090

private_key_path: {{ headscale_data_dir }}/private.key
# Path in /var/lib, non /etc: ProtectSystem=strict del service blocca la scrittura su /etc
noise:
  private_key_path: {{ headscale_data_dir }}/noise_private.key

prefixes:
  v4: {{ headscale_ip_prefix_v4 }}

derp:
  server:
    enabled: false
  urls:
    - https://controlplane.tailscale.com/derpmap/default
  auto_update_enabled: true
  update_frequency: 24h

disable_check_updates: true

database:
  type: sqlite
  sqlite:
    path: {{ headscale_data_dir }}/db.sqlite

log:
  level: info

dns:
  magic_dns: true
  base_domain: {{ headscale_base_domain }}
  nameservers:
    global:
      - 1.1.1.1
      - 9.9.9.9

```



             ## HEADSCALE-CLIENT



```

/koha-ansible/roles/headscale-client/defaults/main.yml

headscale_login_server: "http://192.168.1.4:8443"
headscale_preauth_key: "{{ vault_headscale_preauth_key }}"

```



/koha-ansible/roles/headscale-client/defaults/main.yml

```

headscale_login_server: "http://192.168.1.4:8443"
headscale_preauth_key: "{{ vault_headscale_preauth_key }}"

```


/koha-ansible/roles/headscale-client/handlers/main.yml

```

- name: Aggiorna cache apt
  ansible.builtin.apt:
    update_cache: true

```



/koha-ansible/roles/headscale-client/tasks/main.yml


```

- name: Installa dipendenze per il repository Tailscale
  ansible.builtin.apt:
    name:
      - curl
      - gnupg
    state: present
    update_cache: true

- name: Aggiungi la chiave GPG del repository Tailscale
  ansible.builtin.get_url:
    url: "https://pkgs.tailscale.com/stable/debian/trixie.noarmor.gpg"
    dest: /usr/share/keyrings/tailscale-archive-keyring.gpg
    mode: '0644'

- name: Aggiungi il repository APT di Tailscale
  ansible.builtin.get_url:
    url: "https://pkgs.tailscale.com/stable/debian/trixie.tailscale-keyring.list"
    dest: /etc/apt/sources.list.d/tailscale.list
    mode: '0644'
  notify: Aggiorna cache apt

- name: Applica subito gli handler in sospeso (repository appena aggiunto)
  ansible.builtin.meta: flush_handlers

- name: Installa il pacchetto tailscale
  ansible.builtin.apt:
    name: tailscale
    state: present
    update_cache: true

- name: Abilita e avvia il servizio tailscaled
  ansible.builtin.systemd:
    name: tailscaled
    state: started
    enabled: true

- name: Verifica lo stato attuale della connessione tailscale
  ansible.builtin.command: tailscale status --json
  register: tailscale_status
  changed_when: false
  failed_when: false

- name: Connetti il nodo alla tailnet Headscale tramite preauth key
  ansible.builtin.command: >
    tailscale up
    --login-server={{ headscale_login_server }}
    --authkey={{ headscale_preauth_key }}
    --accept-routes
  register: tailscale_up_result
  changed_when: "'Success' in tailscale_up_result.stdout or tailscale_up_result.rc == 0"
  when: >
    tailscale_status.rc != 0 or
    '"BackendState":"Running"' not in tailscale_status.stdout

```



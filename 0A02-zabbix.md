# Zabbix: monitoraggio su macchina dedicata

## Perché una VM separata

A differenza degli altri servizi, Zabbix vive su una macchina distinta dal server principale (Koha/OpenLDAP/Headscale), collegata tramite una rete interna dedicata (LAN03). La scelta non è casuale: un sistema di monitoraggio che condivide l'host con ciò che deve sorvegliare eredita gli stessi punti di guasto — se il server principale ha un problema serio, il monitoring rischia di scomparire proprio nel momento in cui servirebbe di più.

## Installazione

Zabbix 7.0 LTS, con MariaDB dedicata (non condivisa con quella di Koha, per lo stesso principio di isolamento) e frontend su Apache.

Il ruolo gestisce l'aggiunta del repository in modo dichiarativo, viene definita la struttura APT standard puntando alla release Debian corretta , in questo caso Trixie ,definita nel file .../zabbix-server/defaults/main.yml, con la chiave GPG indicata espressamente :

```yaml
- name: Aggiungi il repository APT di Zabbix
  ansible.builtin.apt_repository:
    repo: "deb [signed-by=/usr/share/keyrings/zabbix-official-repo.key] https://repo.zabbix.com/zabbix/{{ zabbix_version }}/debian {{ zabbix_debian_release }} main"
    state: present
    filename: zabbix
```

Il resto del ruolo segue la sequenza tipica: installazione dei pacchetti (`zabbix-server-mysql`, `zabbix-frontend-php`, `zabbix-apache-conf`, `zabbix-agent2`), creazione di database e utente dedicati, import dello schema SQL iniziale (una tantum, verificato controllando l'esistenza della tabella `users` prima di procedere), configurazione di `DBPassword` e timezone.

## Un imprevisto durante il test: root bloccato fuori da MariaDB

Durante un test di cambio password sull'utente `root` di MariaDB, un passaggio saltato (il `.my.cnf` non aggiornato in coppia con la password nel database) ha lasciato il sistema in uno stato incoerente: né la vecchia password né quella nuova risultavano valide.

```text
mysql -u root
Access denied for user 'root'@'localhost'
```

**Recovery**: avviare MariaDB in modalità di emergenza, che bypassa temporaneamente l'autenticazione, per poter reimpostare la password da zero.

```bash
sudo systemctl stop mariadb
sudo pkill -9 mariadbd
sudo mariadbd-safe --skip-grant-tables --skip-networking &
```

```bash
mysql -u root
```
```sql
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'lapassword';
FLUSH PRIVILEGES;
```

```bash
sudo pkill mariadbd
sudo systemctl start mariadb
```

Verificato l'accesso con la nuova password, coerente sia nel database sia in `/root/.my.cnf`, il servizio è tornato regolare.

## Setup iniziale

Al primo accesso all'interfaccia web (`http://<ip>/zabbix`), il wizard guida attraverso la verifica dei requisiti PHP, la configurazione della connessione al database (stesse credenziali definite in `group_vars`) e l'assegnazione di un nome all'istanza, prima di arrivare alla schermata di login (credenziali di default `Admin`/`zabbix`, da cambiare subito).

# I primi ostacoli: autenticazione, repository, dipendenze Apache

## MariaDB: l'autenticazione di root

```text
fatal: [SERVER]: FAILED! => {"changed": false, "msg": "unable to find /root/.my.cnf.
Exception message: (1698, \"Access denied for user 'root'@'localhost'\")"}
```

Il modulo Ansible non riusciva a connettersi come utente `root` su `localhost`: sulle versioni recenti di MariaDB, l'utente root usa di default il plugin **`unix_socket`** (autenticazione basata sull'utente di sistema, non su password), che non è compatibile con l'autenticazione via `.my.cnf` che Ansible si aspetta.

**Fix**: impostare esplicitamente una password per root, e creare il file di credenziali che Ansible legge automaticamente.

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'lapassword';
FLUSH PRIVILEGES;
```

```bash
sudo nano /root/.my.cnf
```
```ini
[client]
user=root
password=lapassword
```

```bash
sudo chmod 600 /root/.my.cnf
mysql --defaults-file=/root/.my.cnf -e "SHOW DATABASES;"
```

### Nota tecnica: `IDENTIFIED BY` vs `IDENTIFIED VIA`

Il comando `ALTER USER`/`CREATE USER` permette di specificare il metodo di autenticazione in due modi diversi:

- **`IDENTIFIED BY 'stringa'`** : password classica; il server la cifra con l'algoritmo predefinito per connessioni standard.
- **`IDENTIFIED VIA plugin`** : delega il login a un modulo esterno. I casi più comuni sono `unix_socket` (autenticazione basata sull'utente Linux di sistema, senza password) o `gssapi`/`pam` (integrazione con domini aziendali).

Confondere i due, provando a combinarli, o usando `VIA` quando serve semplicemente `BY`, è la causa più comune di errori di sintassi quando si configura root da zero.

---

## Repository Tailscale: chiave GPG non gestibile

```text
[ERROR]: Module failed: Either apt-key or gpg binary is required, but neither could
be found. The apt-key CLI has been deprecated and removed in modern Debian and
derivatives, you might want to use "deb822_repository" instead.
```

Il modulo per aggiungere il repository APT di Tailscale richiedeva `apt-key` o `gpg`, entrambi assenti sul sistema, `apt-key` è deprecato e rimosso nelle versioni moderne di Debian.

**Fix**: installare `gnupg` come dipendenza preliminare del ruolo.

```bash
sudo apt update && sudo apt install -y gnupg
```

---

## Koha: dipendenze Apache mancanti

```text
Koha requires mod_rewrite to be enabled within Apache in order to run.
```
```text
Koha requires mod_cgi to be enabled within Apache in order to run.
```

`koha-create` richiede alcuni moduli Apache non abilitati di default su un'installazione pulita — prima `mod_rewrite`, poi, al tentativo successivo, `mod_cgi`.

**Fix**: task dedicati con `community.general.apache2_module`, aggiunti in `roles/koha/tasks/instance.yml` prima della creazione dell'istanza, così le dipendenze sono garantite ad ogni run.

```yaml
- name: Abilitare il modulo rewrite di Apache
  community.general.apache2_module:
    name: rewrite
    state: present
  notify: Riavviare Apache

- name: Abilitare il modulo cgi di Apache
  community.general.apache2_module:
    name: cgi
    state: present
  notify: Riavviare Apache
```

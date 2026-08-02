# Dall'installazione al servizio stabile: il debug di Headscale

Prima di arrivare a Headscale, il primo tentativo di VPN mesh per questo progetto era stato Tailscale nella sua forma originale (coordination server cloud ufficiale). La scelta è poi ricaduta su Headscale , l'implementazione open source e self-hosted dello stesso protocollo , per restare coerente con l'approccio self-hosted del resto dell'infrastruttura, senza dipendenze da un coordination server di terzi.

Il primo avvio del servizio non è stato indolore: tre problemi distinti, ciascuno diagnosticato con `journalctl` prima di trovare il fix.

## 1. Variabile di versione non definita

```
Error while resolving value for 'dest': 'headscale_version' is undefined
```

Il ruolo Ansible richiede esplicitamente la versione di Headscale da scaricare, senza un valore di default preimpostato — una scelta voluta, per evitare di installare silenziosamente una versione obsoleta. Verificata su GitHub l'ultima release stabile, il fix è stata una semplice aggiunta in `group_vars/all.yml`:

```yaml
headscale_version: "0.29.2"
```

## 2. Chiave privata su filesystem in sola lettura

Il servizio partiva e moriva subito, in loop di restart. `journalctl -u headscale` ha rivelato la causa:

```
Error: initializing: creating new headscale: reading or creating Noise protocol
private key: saving private key to disk at path "/etc/headscale/noise_private.key":
open /etc/headscale/noise_private.key: read-only file system
```

Il file `.service` del pacchetto Debian applica hardening di sicurezza con `ProtectSystem=strict`, che rende `/etc` di sola lettura per il processo. Il file stesso prevedeva già la soluzione, ma commentata di default:

```
# WorkingDirectory=/var/lib/headscale
# ReadWritePaths=/var/lib/headscale
```

Invece di forzare un override systemd, la scelta è stata allinearsi al comportamento già previsto dal pacchetto: spostare il path delle chiavi da `/etc/headscale` a `/var/lib/headscale` direttamente nel template `config.yaml.j2` — coerente con la distinzione tra configurazione statica (`/etc`) e dati generati a runtime (`/var/lib`).

```yaml
private_key_path: {{ headscale_data_dir }}/private.key
noise:
  private_key_path: {{ headscale_data_dir }}/noise_private.key
```

## 3. Conflitto di porta con Apache/Koha

Risolto il problema delle chiavi, il servizio continuava comunque a fallire:

```
Error: headscale ran into an error and had to shut down: binding to TCP address:
listen tcp 127.0.0.1:8080: bind: address already in use
```

La porta 8080 era già occupata da Apache, che su questa stessa macchina serve anche la staff interface di Koha. Il fix richiedeva cambiare la porta di Headscale, ma il primo tentativo è stato incompleto: aggiornata solo `headscale_server_url` a `8443` in `group_vars/all/vars.yml`, lasciando invariato `headscale_listen_port` (ancora al default `8080`). Il servizio continuava a fallire con lo stesso errore.

La verifica diretta del file generato sul server ha reso il problema visibile a colpo d'occhio:

```bash
/etc/headscale/config.yaml
```
```yaml
server_url: http://192.168.1.4:8443
listen_addr: 127.0.0.1:8443
```

```bash
/koha-ansible/group_vars/all/vars.yml

```yaml
headscale_server_url: "http://192.168.1.4:8443"
headscale_listen_port: 8443
```

Da qui in avanti, servizio stabile e `headscale users list` finalmente funzionante.

## Conclusione

I tre problemi, per quanto diversi, condividono lo stesso metodo di risoluzione: **non fidarsi del sintomo superficiale** (un timeout di connessione al socket può nascondere cause completamente diverse ; una variabile mancante, un permesso negato, una porta occupata) e **verificare sempre lo stato reale** con `journalctl` e ispezionando il file di configurazione effettivamente generato, piuttosto che presumere che le modifiche fatte a monte (in `group_vars`) si siano propagate correttamente.

## Un pattern ricorrente: variabili non definite

Due episodi distinti nel progetto, in due ruoli diversi (OpenLDAP e Headscale), condividono la stessa identica causa: una variabile referenziata in un task ma mai dichiarata in `group_vars/all.yml`.

**OpenLDAP** — il task che genera la struttura base (`structure.ldif`) usava `ldap_organization` nel contenuto del file, mai definita:

```
Error while resolving value for 'content': 'ldap_organization' is undefined
```

**Headscale** — il task di download del pacchetto `.deb` referenziava `headscale_version` per costruire URL e percorso di destinazione, anch'essa mai definita:

```
Error while resolving value for 'dest': 'headscale_version' is undefined
```

In entrambi i casi il fix è stato lo stesso: popolare `group_vars/all.yml` con le variabili mancanti prima del primo run del ruolo.

```yaml
# esempio OpenLDAP
ldap_organization: "La Mia Biblioteca"
ldap_domain: "biblioteca.local"
ldap_root_dn: "cn=admin,dc=biblioteca,dc=local"
ldap_root_pass: "PasswordSicuraLDAP"

# esempio Headscale
headscale_version: "0.29.2"
```

Non è un bug, è un promemoria: quando si scrive un ruolo Ansible nuovo, Ansible non "immagina" un default sensato per una variabile mai vista — la finalizzazione del task fallisce subito, prima ancora di connettersi all'host. Una lettura veloce del task file, chiedendosi esplicitamente "ogni `{{ variabile }}` qui dentro è già dichiarata da qualche parte?", avrebbe intercettato entrambi i casi prima del primo run.

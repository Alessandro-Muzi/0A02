# Errore 500 al login: il mapping LDAP e l'attributo `is`

## Sintomo

Dopo aver configurato l'autenticazione LDAP in `koha-conf.xml`, il login all'interfaccia staff di Koha restituiva un Errore 500 generico. Nulla di utile nella pagina d'errore — la causa reale era solo nei log Apache.

```bash
tail -n 20 /var/log/koha/biblioteca/intranet-error.log
```

```
AH01215: stderr from mainpage.pl: Can't use string ("MAIN") as a HASH ref
while "strict refs" in use at /usr/share/koha/lib/C4/Auth_with_ldap.pm line 65
```

## Prima ipotesi

Il blocco LDAP generato da Ansible aveva questa forma:

```xml
<mapping>
  <firstname field="givenName" />
  <surname field="sn" />
  <branchcode>MAIN</branchcode>
  <categorycode>PT</categorycode>
  <userid field="uid" />
</mapping>
```

L'attributo `field=` sembrava ragionevole (è il nome più intuitivo), ma non trovava riscontro nel codice. Un primo tentativo di correzione con `is_key="0"` ha solo spostato il problema: l'errore 500 è sparito, ma è comparso un nuovo errore diverso.

```
No ldapserver "mapping for 'userid'" defined in KOHA_CONF
at Auth_with_ldap.pm line 97
```

## La causa reale

Leggendo direttamente il sorgente Perl (`Auth_with_ldap.pm`), il codice cerca esplicitamente la chiave `is`, non `field` né `is_key`:

```perl
my $uid_field = $mapping{userid}->{is}
    or die ldapserver_error("mapping for 'userid'");
```

Il file contiene anche, come commento POD (righe 555-595), la documentazione ufficiale della sintassi attesa — con tanto di esempio completo:

```perl
<firstname    is="givenname"      ></firstname>
<branchcode   is="branch"         >MAIN</branchcode>
<categorycode is="employeetype"   >PT</categorycode>
```

Il nome dell'elemento XML è il campo Koha (es. `branchcode`), l'attributo `is` indica l'attributo LDAP corrispondente, e il testo tra i tag (se presente) è il valore di default statico usato quando LDAP non restituisce nulla per quel campo.

## Il fix

Riscritto l'intero blocco `<mapping>` con la sintassi corretta, e integrato nel ruolo Ansible con `blockinfile` per restare gestito automaticamente ad ogni run:

```yaml
- name: Configurare l'autenticazione OpenLDAP in koha-conf.xml
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

## Lezione

Il nome più "intuitivo" per un attributo di configurazione (`field=`) non è sempre quello corretto: quando la documentazione ufficiale scarseggia o è ambigua, il sorgente del software resta la fonte di verità più affidabile — in questo caso, letteralmente commentato nel codice stesso.

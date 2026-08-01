## Perché queste scelte

Mi interessava provare un gestionale per biblioteche, così dopo una rapida ricerca ho scoperto Koha: oltre a essere open source, è l'Integrated Library System più diffuso al mondo, con un ecosistema particolarmente ricco e attivo. Ho maturato un interesse specifico per questo prodotto, ed è la ragione per cui l'ho scelto come cuore di questo progetto.

La scelta di accompagnarlo con Debian, Apache, OpenLDAP e MariaDB è avvenuta invece per ragioni di integrazione, compatibilità e prestazioni , oltre logicamente al fattore open source, senza precludere in futuro l'adozione di soluzioni come FreeIPA o alternative analoghe
Per la VPN cercavo una soluzione più scalabile rispetto a WireGuard puro, già usato in diverse esercitazioni al corso CEFI e nel progetto 0A01. Ho preso in considerazione prima Tailscale, per poi passare a Headscale: a differenza del primo ,un servizio cloud gestito da terzi, Headscale mi offre controllo pieno e gestione diretta di ogni operazione, restando coerente con l'approccio self-hosted del resto dell'infrastruttura.

Per il monitoraggio in tempo reale la scelta è caduta su Zabbix. E in tema di scalabilità e automazione, pur essendo l'infrastruttura pensata su due sole macchine, ho scelto comunque di affidare l'intera installazione ad Ansible ,proprio per approfondirne l'uso e le peculiarità anche in un contesto di dimensioni ridotte.

### Stack tecnologico

| Componente | Ruolo |
|---|---|
| Debian | Sistema operativo |
| Ansible | Automazione e provisioning |
| nftables | Firewall e routing |
| Koha | Integrated Library System |
| Apache | Web server |
| MariaDB | Database |
| OpenLDAP | Autenticazione e gestione utenza |
| Headscale | VPN mesh self-hosted |
| Zabbix | Monitoring |

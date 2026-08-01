# Il gateway invisibile: WSL, routing e ProxyJump SSH

Con la VM Zabbix collegata alla nuova rete interna LAN03, il primo ostacolo non è arrivato dal ruolo Ansible, ma da qualcosa di più a monte: il control node stesso.

## Sintomo

`ansible-playbook` falliva sull'host Zabbix con un timeout SSH:

```
Data could not be sent to remote host "10.0.0.20". Make sure this host can be
reached over ssh: ssh: connect to host 10.0.0.20 port 22: Connection timed out
```

Il server principale (Koha), sulla stessa esecuzione, funzionava senza problemi — il timeout era isolato alla nuova macchina.

## Prima ipotesi: il firewall

Il primo sospetto è caduto sulla chain `forward` del gateway, che permetteva solo traffico verso una porta specifica:

```
chain forward {
    type filter hook forward priority filter; policy drop;
    ct state established,related accept
    iifname "enp0s3" oifname "enp0s8" ip daddr 10.0.0.20 tcp dport 22 accept
}
```

Ipotesi corretta solo in parte: mancava effettivamente la regola per il traffico generico in uscita da LAN03 verso l'esterno (nessun `ping` verso indirizzi pubblici passava), risolta aggiungendo:

```
iifname "enp0s8" oifname "enp0s3" accept
```

Ma anche dopo questo fix, il ping da WSL verso `10.0.0.20` continuava a fallire — mentre lo stesso ping, lanciato *dal server* verso la VM Zabbix, funzionava perfettamente.

## La causa reale: WSL2 non è sulla rete locale

WSL2 non ha un'interfaccia di rete fisica diretta: gira in una VM Hyper-V gestita da Windows, con una propria subnet NAT interna (nell'esempio, `172.27.176.0/20`). Comunica con l'esterno tramite un livello di traduzione gestito da Windows — il fatto che riuscisse a raggiungere il server (`192.168.1.4`) non significa che avesse una vera rotta verso reti *dietro* quel server, come `10.0.0.0/24` (LAN03).

```bash
ip route
```
```
default via 172.27.176.1 dev eth0 proto kernel
172.27.176.0/20 dev eth0 proto kernel scope link src 172.27.178.197
```

Nessuna rotta verso `10.0.0.0/24`. Il tentativo di aggiungerla manualmente ha confermato il limite strutturale:

```bash
sudo ip route add 10.0.0.0/24 via 192.168.1.4
# Error: Nexthop has invalid gateway.
```

WSL2 non accetta un gateway che non sia sulla propria subnet diretta — la modalità di rete "mirrored" che risolverebbe il problema alla radice richiede Windows 11, non disponibile in questo caso.

## Il fix: ProxyJump SSH

Invece di forzare il routing di WSL, la soluzione è stata sfruttare la connettività già funzionante del server verso LAN03, usandolo come bastion host:

```bash
ssh -J root@192.168.1.4 root@10.0.0.20
```

Reso permanente nell'inventory Ansible:

```ini
[zabbix]
ZABBIX ansible_host=10.0.0.20 ansible_user=root ansible_ssh_common_args='-o ProxyJump=root@192.168.1.4'
```

## Un ultimo ostacolo: PermitRootLogin

Il primo tentativo di autenticazione con password, attraverso il ProxyJump, veniva comunque rifiutato:

```
Permission denied (publickey,password)
```

La causa: `/etc/ssh/sshd_config` sulla VM Zabbix aveva `PermitRootLogin prohibit-password` — root può accedere solo con chiave SSH, mai con password, indipendentemente da quanto questa sia corretta. La chiave pubblica è stata quindi copiata manualmente in `/root/.ssh/authorized_keys` sulla VM, passando dal server già raggiungibile:

```bash
ssh root@10.0.0.20 "mkdir -p /root/.ssh && echo 'ssh-ed25519 AAAA...' >> /root/.ssh/authorized_keys && chmod 700 /root/.ssh && chmod 600 /root/.ssh/authorized_keys"
```

Da quel momento, l'autenticazione via ProxyJump è passata a chiave, e il playbook ha raggiunto `ZABBIX` senza ulteriori intoppi.

## Un ultimo strato: la chain input

Risolto il ProxyJump, un problema simile si è ripresentato il giorno successivo — stavolta non su SSH, ma sull'accesso via browser a due servizi diversi: l'interfaccia web di Zabbix e la staff interface di Koha, entrambi irraggiungibili nonostante i servizi risultassero attivi e in ascolto correttamente.

**Headscale (porta 8443)**: il client sulla VM Zabbix non riusciva a completare la registrazione presso il coordination server:

```
Received error: fetch control key: Get "http://192.168.1.4:8443/key?v=138":
dial tcp 192.168.1.4:8443: connect: connection timed out
```

Prima ipotesi verificata (e confermata, ma non sufficiente da sola): Headscale ascoltava solo su `127.0.0.1:8443`, non raggiungibile da altre macchine. Corretto il `listen_addr` a `0.0.0.0:8443` nel template, il problema persisteva comunque — mancava ancora una regola nella **chain input** del gateway (non `forward`, perché il traffico è diretto al servizio in esecuzione *sul* gateway stesso, non lo attraversa per raggiungere un'altra macchina):

```
tcp dport 8443 accept
```

**Koha (porta 8080)**: stesso pattern, un giorno dopo. Da WSL il `ping` verso il server funzionava, ma `curl` andava in timeout dopo lunga attesa:

```
curl: (28) Failed to connect to 192.168.1.4 port 8080 after 21004 ms
```

Apache era correttamente in ascolto su `*:8080` (verificato con `ss -tlnp`), e rispondeva regolarmente quando interrogato localmente dal server stesso. Anche qui, la causa era la stessa: mancava la regola input dedicata alla porta 8080.

## Lezione

Il sintomo iniziale (timeout SSH, poi timeout HTTP) suggeriva ogni volta un problema di firewall — ed era vero, ma non nel punto in cui si guardava per primo. Vale la pena distinguere sempre le due chain: **`forward`** per il traffico che il gateway deve *instradare* verso un'altra macchina, **`input`** per il traffico diretto *al gateway stesso*. Un servizio che gira sul gateway (come Headscale o, più in generale, qualsiasi demone in ascolto su quella macchina) richiede sempre una regola input dedicata — indipendentemente da quanto sia permissiva la chain forward, che governa tutt'altro traffico.

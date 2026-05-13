# Exercice 2 — DHCP : observer DORA paquet par paquet

**Durée estimée :** 45 min
**Objectif :** capturer un échange DHCP complet (Discover / Offer / Request /
ACK), identifier les options portées par chaque message, et comprendre
pourquoi DHCP utilise un broadcast L2 alors qu'IP n'est pas encore
configuré.

## Manipulation

Côté `dhcp-server`, démarrez une capture filtrée sur les ports DHCP (67/68)&nbsp;:

```bash
docker exec -it lab_dhcp_server tcpdump -i eth0 -nn -e -v port 67 or port 68
```

> Note&nbsp;: `-e` affiche les adresses MAC, indispensables pour comprendre
> le broadcast L2.

Côté `client`, déclenchez une nouvelle demande de bail&nbsp;:

```bash
docker exec lab_client bash -c "dhclient -r eth0 2>/dev/null; dhclient -v eth0"
```
<img width="1979" height="416" alt="image" src="https://github.com/user-attachments/assets/e99033fb-528b-4867-b942-8b266e347fb0" />

Observez les **4 paquets** DORA dans la capture, puis arrêtez tcpdump (Ctrl+c).

Affichez aussi les journaux applicatifs du serveur&nbsp;:

```bash
docker logs --tail 40 lab_dhcp_server
```
<img width="1886" height="1121" alt="image" src="https://github.com/user-attachments/assets/1a24100c-0e23-43e2-a08d-b2e945311fc9" />
<img width="1671" height="61" alt="image" src="https://github.com/user-attachments/assets/dd067892-9224-4bcf-a940-06c2bf452859" />

## À rendre — répondez directement dans ce fichier

### 1. Tableau DORA

Complétez en vous appuyant sur **votre propre capture**&nbsp;:

| Étape       | Émetteur (IP src) | Destinataire (IP dst) | MAC src / dst | Options DHCP notables |
| ----------- | ----------------- | --------------------- | ------------- | --------------------- |
| 1. Discover | `0.0.0.0`         | `255.255.255.255`     | `0a:46:31:0a:e8:31 → ff:ff:ff:ff:ff:ff`| option 53 =DHCPDISCOVER, opt 55 = liste des options demandées (1,28,2,3,15,6,119,12,44,47,26,121,42) |
| 2. Offer    | `172.20.1.2`      | `172.20.1.168`        | `MAC serveur → 0a:46:31:0a:e8:31`| opt 53 = DHCPOFFER, opt 54 = server-id 172.20.1.2, opt 51 = lease 12h, opt 58 = T1 6h, opt 59 = T2 10h30, opt 1 = netmask 255.255.255.0, opt 3 = router 172.20.1.254, opt 6 = dns 1.1.1.1, 8.8.8.8, opt 15 = domain lab.local |
| 3. Request  | `0.0.0.0`         | `255.255.255.255`     | `0a:46:31:0a:e8:31 → ff:ff:ff:ff:ff:ff`|opt 53 = DHCPREQUEST, opt 54 = server-id 172.20.1.2, opt 50 = requested IP 172.20.1.168  |
| 4. ACK      | `172.20.1.2`      | `172.20.1.168`        | `MAC serveur → 0a:46:31:0a:e8:31`| opt 53 = DHCPACK, opt 54 = server-id 172.20.1.2, opt 51 = lease 12h, opt 58 = T1 6h, opt 59 = T2 10h30, opt 1 = netmask, opt 28 = broadcast 172.20.1.255, opt 12 = hostname client, opt 15 = lab.local, opt 6 = dns, opt 3 = router |

### 2. Configuration finale du client

```bash
docker exec lab_client ip -4 addr show eth0
docker exec lab_client ip route
docker exec lab_client cat /etc/resolv.conf   # peut être vide si non géré par dhclient
```

Notez **l'IP attribuée, le masque, la passerelle, les DNS, la durée de bail**.

> 💬 **Votre réponse :**
>
> _Remplacez ce texte par votre réponse (IP / masque / GW / DNS / bail)._

### 3. Questions de réflexion

**Question 1.** Pourquoi le client utilise-t-il **`0.0.0.0` comme IP
source** pour le Discover, alors que c'est une adresse non routable&nbsp;?
Que se passerait-il avec n'importe quelle autre adresse&nbsp;?

> 💬 **Votre réponse :**
>
> _Remplacez ce texte par votre réponse._

**Question 2.** Pourquoi le **Request** est-il **rediffusé en broadcast**
alors que le client connaît déjà l'IP du serveur après l'Offer&nbsp;?

> 💬 **Votre réponse :**
>
> _Remplacez ce texte par votre réponse._

**Question 3.** À quoi sert le **transaction ID (xid)** présent dans les
4 paquets&nbsp;? Que se passerait-il s'il était omis dans un réseau avec
plusieurs serveurs DHCP&nbsp;?

> 💬 **Votre réponse :**
>
> _Remplacez ce texte par votre réponse._

**Question 4.** Que renvoie le serveur si vous demandez explicitement une
adresse hors du pool (essayez `dhclient -v -s 172.20.1.99 eth0`)&nbsp;?
Justifiez.

> 💬 **Votre réponse :**
>
> _Remplacez ce texte par votre réponse._

**Question 5.** La directive `dhcp-authoritative` est active sur notre
serveur. Quel est son effet **comportemental** sur les NAK&nbsp;?

> 💬 **Votre réponse :**
>
> _Remplacez ce texte par votre réponse._

### 4. Renouvellement de bail (T1/T2)

Le bail est de 12&nbsp;h, T1 (renouvellement) à 6&nbsp;h, T2 (rebind) à 10&nbsp;h30.
En **2-3 phrases**, décrivez la différence entre un renouvellement T1 et
un rebind T2 (destinataire du paquet, comportement attendu).

> 💬 **Votre réponse :**
>
> _Remplacez ce texte par votre réponse._

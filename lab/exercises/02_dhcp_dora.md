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
> Le client utilise 0.0.0.0 car il n'a pas encore d'adresse IP configurée — c'est précisément l'objet du Discover. 0.0.0.0 est l'adresse conventionnelle signifiant "source inconnue/non configurée" en RFC 2131. Si le client utilisait une > IP quelconque inventée (ex: 192.168.0.1), deux problèmes : premièrement elle pourrait entrer en conflit avec un équipement existant, deuxièmement les routeurs refuseraient de router un paquet source avec une IP non attribuée (filtrage anti-spoofing).

**Question 2.** Pourquoi le **Request** est-il **rediffusé en broadcast**
alors que le client connaît déjà l'IP du serveur après l'Offer&nbsp;?

> 💬 **Votre réponse :**
>
> Même si le client connaît l'IP du serveur après l'Offer, il envoie le Request en broadcast pour deux raisons :
>
> 1. Informer les autres serveurs DHCP du réseau qu'il a choisi une offre (ils doivent libérer leurs offres concurrentes)
> 2. Le client n'a toujours pas d'IP configurée — techniquement il ne peut pas encore faire de l'unicast IP routable
>
> Le broadcast permet donc une communication "propre" même en environnement multi-serveurs.

**Question 3.** À quoi sert le **transaction ID (xid)** présent dans les
4 paquets&nbsp;? Que se passerait-il s'il était omis dans un réseau avec
plusieurs serveurs DHCP&nbsp;?

> 💬 **Votre réponse :**
>
> Le xid (transaction ID) est un nombre aléatoire 32 bits généré par le client au début de chaque échange DORA. Il permet d'associer les réponses du serveur à la bonne demande du bon client. Sans xid, dans un réseau avec plusieurs serveurs DHCP et plusieurs clients, un client ne saurait pas si un OFFER reçu répond à son DISCOVER ou à celui d'un voisin. Avec plusieurs serveurs, deux OFFER arriveraient avec des IP différentes, le xid garantit que le client traite uniquement les réponses qui lui sont destinées.

**Question 4.** Que renvoie le serveur si vous demandez explicitement une
adresse hors du pool (essayez `dhclient -v -s 172.20.1.99 eth0`)&nbsp;?
Justifiez.

> 💬 **Votre réponse :**
>
> <img width="2037" height="518" alt="image" src="https://github.com/user-attachments/assets/9a15dc0c-10d0-47c4-8c68-cdaf80d83097" />

> Le serveur répond avec un DHCPNAK (option 53 = 6). Le pool configuré est 172.20.1.100 -- 172.20.1.200 (visible dans tes logs). L'adresse 172.20.1.99 est en dehors du pool : le serveur ne peut pas la confirmer et rejette la demande par un NAK pour forcer le client à recommencer un DISCOVER depuis zéro.

**Question 5.** La directive `dhcp-authoritative` est active sur notre
serveur. Quel est son effet **comportemental** sur les NAK&nbsp;?

> 💬 **Votre réponse :**
>
> Sans cette directive, si un client arrive avec une adresse d'un réseau inconnu, le serveur ignore silencieusement le REQUEST (comportement prudent).
> Avec dhcp-authoritative, le serveur se déclare autoritaire sur son réseau : il répond immédiatement par un DHCPNAK à tout REQUEST suspect (adresse incorrecte, réseau incohérent). Cela accélère la reconvergence : le client ne patiente pas le timeout, il recommence immédiatement un nouveau DORA.

### 4. Renouvellement de bail (T1/T2)

Le bail est de 12&nbsp;h, T1 (renouvellement) à 6&nbsp;h, T2 (rebind) à 10&nbsp;h30.
En **2-3 phrases**, décrivez la différence entre un renouvellement T1 et
un rebind T2 (destinataire du paquet, comportement attendu).

> 💬 **Votre réponse :**
>
> D'après tes logs : lease = 12h, T1 = 6h, T2 = 10h30
>
> À T1 (6h) — Renouvellement unicast : le client envoie un DHCPREQUEST directement au serveur qui lui a attribué le bail (unicast vers 172.20.1.2). C'est un échange discret entre les deux. Si le serveur répond ACK, le bail est renouvelé pour 12h. Si le serveur ne répond pas (down), le client attend T2.
> 
> À T2 (10h30) — Rebind broadcast : le client n'a toujours pas réussi à renouveler. Il envoie un DHCPREQUEST en broadcast (255.255.255.255) pour tenter de se faire prendre en charge par n'importe quel serveur DHCP disponible sur le réseau. C'est le mécanisme de secours. Si aucun serveur ne répond avant la fin du bail (12h), le client perd son IP et recommence un DORA complet.
> 
> Lease = 12h | T1 = 6h (renouvellement) | T2 = 10h30 (rebind)
>
>À T1 (6h)   → DHCPREQUEST unicast vers 172.20.1.2
               Si ACK : bail renouvelé pour 12h
               Si pas de réponse : on attend T2

>À T2 (10h30)→ DHCPREQUEST broadcast vers 255.255.255.255
               N'importe quel serveur DHCP peut répondre
               Si pas de réponse : on attend l'expiry

>À 12h       → IP libérée, le client repart d'un DORA complet

# Exercice 1 — OSI à travers une vraie capture de paquets

**Durée estimée :** 45 min
**Objectif :** relier chaque couche du modèle OSI à un champ concret observable
dans une capture réseau effectuée sur le lab.

## Préparation

Configurez le client (s'il ne l'a pas déjà été par DHCP — voir exercice 2)&nbsp;:

```bash
docker exec lab_client bash -c "ip route del default 2>/dev/null; ip route add default via 172.20.1.254"
```

Vérifiez l'accès au site « public »&nbsp;:

```bash
docker exec lab_client curl -s http://172.20.0.10/whoami
```

## Manipulation

**Étape 1 — supprimez une éventuelle capture précédente** (sinon échec en
*Permission denied* car tcpdump abandonne ses privilèges vers l'utilisateur
`tcpdump` après ouverture du fichier) :

```bash
docker exec lab_client rm -f /tmp/http.pcap
```

**Étape 2 — lancez la capture côté client.** Les flags importants&nbsp;:
`-U` (écriture non bufferisée, indispensable si la capture est interrompue),
`-c 30` (s'arrête automatiquement après 30 paquets, ≈ 2 à 3 requêtes HTTP) :

```bash
docker exec lab_client tcpdump -i eth0 -U -w /tmp/http.pcap -nn -c 30 host 172.20.0.10
```

> ⚠️ Cette commande **bloque** le terminal tant que les 30 paquets ne sont
> pas capturés. Ouvrez un **second terminal** pour l'étape 3.

**Étape 3 — dans un second terminal, déclenchez du trafic** *pendant* que
tcpdump tourne&nbsp;:

```bash
docker exec lab_client curl -v http://172.20.0.10/
docker exec lab_client curl -v http://172.20.0.10/whoami
docker exec lab_client curl -s http://172.20.0.10/        # complément pour atteindre 30 paquets
```

tcpdump s'arrête seul dès le 30e paquet et affiche un résumé du type
`30 packets captured / 0 packets dropped by kernel`.

**Étape 4 — vérifiez que la capture est exploitable** :

```bash
docker exec lab_client capinfos /tmp/http.pcap
```

`Number of packets` doit être > 0. Si la capture est vide, voir la section
*Pièges fréquents* en bas de cet énoncé.

**Étape 5 — analysez la capture**. Trois vues utiles&nbsp;:

```bash
# Vue compacte : une ligne par paquet (utile pour repérer les n° de frames)
docker exec lab_client tshark -r /tmp/http.pcap
```
<img width="1461" height="750" alt="image" src="https://github.com/user-attachments/assets/8072483c-c764-45b6-bdb7-c62dff393b38" />

```bash
# Vue détaillée d'un paquet précis (ex. la requête GET = frame n°4)
docker exec lab_client tshark -r /tmp/http.pcap -V -Y 'frame.number == 4'
```
<img width="1440" height="876" alt="image" src="https://github.com/user-attachments/assets/60dba9aa-549d-44cd-84b9-a48edfbfd9c2" />
<img width="1454" height="873" alt="image" src="https://github.com/user-attachments/assets/e6a45959-2cd3-422c-99ae-47fd7524f62f" />

```bash
# Vue détaillée complète (pager : utilisez 'q' pour quitter)
docker exec -it lab_client sh -c "tshark -r /tmp/http.pcap -V | less"
```

> `-V` produit la décomposition complète couche par couche
> (Frame → Ethernet → IP → TCP → HTTP). Le filtre `-Y` cible un paquet
> par son numéro pour éviter d'avoir à scroller dans toute la trace.

## Visualisation assistée : `osi_inspect.py`

La sortie brute de `tshark -V` est dense (plusieurs centaines de lignes par
paquet). Un script Python est fourni dans ce dossier pour vous présenter,
pour chaque trame, un **tableau structuré par couche OSI** avec en plus
une **colonne d'explication pédagogique** pour chaque champ.

### Lister les trames

Depuis la racine du dépôt (l'hôte, pas l'intérieur du conteneur) :

```bash
./lab/exercises/osi_inspect.py
```

Vous obtenez la même vue compacte que `tshark` mais sans avoir à taper la
commande complète. Repérez la trame qui vous intéresse — typiquement la
trame portant `GET /` (ligne marquée `HTTP 141 GET / HTTP/1.1`).

### Détailler une trame

```bash
./lab/exercises/osi_inspect.py 4         # trame n°4 — la requête HTTP
./lab/exercises/osi_inspect.py 1         # trame n°1 — le SYN du handshake
./lab/exercises/osi_inspect.py 8         # trame n°8 — la réponse 200 OK
```
<img width="2054" height="1121" alt="image" src="https://github.com/user-attachments/assets/2cdf3417-4073-4488-a078-da295168e650" />
<img width="2033" height="824" alt="image" src="https://github.com/user-attachments/assets/c67f0615-9cea-4a6d-a0b2-bf713abe13f0" />
<img width="2033" height="995" alt="image" src="https://github.com/user-attachments/assets/774bf443-b50a-4a9d-8892-7837dad32918" />

Le script affiche, pour chaque couche OSI **présente** dans la trame, les
champs clés avec **3 informations** :

| Colonne       | Contenu                                       |
| ------------- | --------------------------------------------- |
| `Champ`       | Nom du champ tel qu'extrait par tshark        |
| `Valeur`      | Valeur réelle observée dans **votre** capture |
| `Explication` | À quoi sert ce champ, comment l'interpréter   |

C'est exactement la matière dont vous avez besoin pour remplir le tableau
*« Couche / Élément observé / Valeur exemple »* demandé dans la section
suivante.

### Réutilisation pour les exercices suivants

Le script est générique. Pour disséquer une capture DHCP (exercice 2) ou
NAT (exercice 3), pointez-le vers le bon conteneur et le bon fichier :

```bash
./lab/exercises/osi_inspect.py 1 --pcap /tmp/dhcp.pcap --container lab_dhcp_server
./lab/exercises/osi_inspect.py 3 --pcap /tmp/nat.pcap  --container lab_nat_router
```

### Travail demandé avec ce script

1. Lancez `./lab/exercises/osi_inspect.py` pour obtenir la liste des trames.
Commande utilisé : python3 osi_inspect.py
<img width="2028" height="943" alt="image" src="https://github.com/user-attachments/assets/d60d3cbf-902d-45ca-8abb-39a13f86dea9" />


3. Identifiez **une trame contenant du HTTP** (typiquement la requête `GET /`)
   et **une trame de contrôle TCP** (SYN, ACK seul, ou FIN).
Commande utilisé : python3 osi_inspect.py 4
<img width="2040" height="1079" alt="image" src="https://github.com/user-attachments/assets/2671939f-c712-4547-b7f4-6139f7c3f969" />

5. Lancez le script avec le n° de chaque trame et **copiez la sortie**
   dans le README de votre fork (bloc de code).
Commande utilisé : python3 osi_inspect.py

7. Pour chacune des deux trames, **comptez et nommez** les couches OSI
   visibles (utilisez la ligne `Pile présente : …` en en-tête). Expliquez
   en 1 phrase pourquoi la couche 7 est absente sur la trame de contrôle TCP.
Commande utilisé : python3 osi_inspect.py 1
<img width="2042" height="878" alt="image" src="https://github.com/user-attachments/assets/a59a967d-0b54-46af-a05c-912ca9134c4c" />

> 💬 **Votre réponse (sorties du script + analyse) :**
> Voir au dessus

## À rendre — répondez directement dans ce fichier

Pour **chaque couche OSI**, donnez **un exemple concret extrait de votre
capture** (champ, valeur observée). Justifiez en 1-2 phrases.

| Couche OSI         | Élément observé dans la capture | Valeur exemple |
| ------------------ | ------------------------------- | -------------- |
| 7 — Application    | _ex. méthode HTTP_              | `GET / HTTP/1.1 (trame 4)` |
| 6 — Présentation   | _ex. encodage / Content-Type_   | `text/html (trame 8)`|
| 5 — Session        | _ex. Keep-Alive, cookies_       |`Chaque curl ouvre une nouvelle connexion TCP (trames 1-12, 13-22, 23-30) - absence de keep-alive`|
| 4 — Transport      | _ex. port TCP, flags_           | `34550 → 80 [SYN] (trame 1), [FIN, ACK] (trame 10)`              |
| 3 — Réseau         | _ex. IP source / destination_   | `172.20.1.50 → 172.20.0.10`              |
| 2 — Liaison        | _ex. adresses MAC_              | `02:42:ac:...`              |
| 1 — Physique       | _non visible — pourquoi&nbsp;?_ | `—`              |

## Questions de réflexion

**Question 1.** Pourquoi l'**adresse MAC source** observée n'est-elle
**pas** celle du serveur `internet` mais celle du `nat-router`&nbsp;? Que
vous apprend cette observation sur la portée de chaque couche&nbsp;?

> 💬 **Votre réponse :**
> La couche 2 (Ethernet) ne fonctionne que sur le lien local, entre deux équipements directement connectés. Quand le client envoie un paquet vers 172.20.0.10, il le dépose sur son lien local vers le routeur : la MAC destination est donc celle du routeur NAT (la passerelle). Le routeur recrée une nouvelle trame L2 pour le segment suivant. Chaque couche a une portée différente : L3 est bout-en-bout (IP reste celle du client), L2 est saut-par-saut (MAC change à chaque routeur).

**Question 2.** Vous capturez sur `eth0` du client (côté LAN). Dans votre
trace, l'**IP source** sortante est `172.20.1.50`. Pourtant, `curl /whoami`
rapporte que le serveur perçoit `172.20.0.254`. Expliquez cette différence
et indiquez **où** il faudrait capturer pour voir l'IP réécrite.
*Astuce&nbsp;:* `docker exec lab_nat_router tcpdump -i any -nn -c 10 host 172.20.0.10`.

> 💬 **Votre réponse :**
> C'est le NAT (MASQUERADE) en action. Le routeur réécrit l'IP source du paquet quand il le fait sortir vers le réseau "internet" : 172.20.1.50 (IP privée LAN) → 172.20.0.254 (IP publique du routeur). La réécriture a lieu sur l'interface de sortie du routeur (eth0 côté internet). Pour voir l'IP réécrite, il faut capturer sur le routeur NAT :
bashdocker exec lab_nat_router tcpdump -i any -nn -c 10 host 172.20.0.10
>
> <img width="2045" height="694" alt="image" src="https://github.com/user-attachments/assets/97361177-eabe-4a8c-bf20-5a3d35a71e85" />


**Question 3.** Lancez `curl -v https://...` vers un site HTTPS public
(depuis l'hôte, pas le lab). Quelle couche change visiblement par
rapport au HTTP du lab&nbsp;? Quelles couches **disparaissent** de votre
visibilité&nbsp;?

> 💬 **Votre réponse :**
> La couche 6 (Présentation) devient visible : TLS chiffre les données entre L4 et L7. Conséquence : les couches 7 et 5 disparaissent de ta visibilité dans tshark, tu vois TLSv1.3 Application Data au lieu de HTTP GET. Le contenu applicatif est opaque.

**Question 4.** La couche 5 (Session) est très peu visible dans une
capture HTTP/1.1. Donnez **deux mécanismes applicatifs** qui jouent le
rôle de la couche session, et expliquez pourquoi ils sont implémentés
« plus haut »&nbsp;dans la pile.

> 💬 **Votre réponse :**
>  1. Connection: keep-alive maintient la connexion TCP ouverte entre plusieurs requêtes HTTP, évitant un nouveau handshake à chaque fois → rôle de maintien de session.
>  2. Les cookies HTTP permettent de reprendre une "session utilisateur" entre des requêtes distinctes (authentification, panier...) → rôle d'identification et de reprise de session.
>
> Ils sont implémentés plus haut (L7) car TCP/IP n'a pas de couche session native : la suite Internet a été conçue avec seulement 4 couches, et les concepteurs d'applications ont dû gérer eux-mêmes la continuité de session au niveau applicatif.

## Pièges fréquents

* **Capture vide (`Number of packets: 0`)** — vous avez lancé `curl` *avant*
  tcpdump, ou tcpdump a été tué avant d'écrire son buffer. Solution : utilisez
  bien `-U -c 30` (étape 2) et déclenchez le trafic *après* le message
  `listening on eth0…`.
* **`tcpdump: /tmp/http.pcap: Permission denied`** — un fichier appartenant à
  l'utilisateur `tcpdump` (créé par une capture précédente) bloque
  l'écriture. Solution : `docker exec lab_client rm -f /tmp/http.pcap`.
* **`tshark … | less` n'affiche rien** — vous êtes dans un environnement
  sans TTY (script, pipeline). Retirez `| less` ou utilisez
  `docker exec -it lab_client sh -c "… | less"`.

# Teaching-HEIGVD-SRX-2022-Laboratoire-Snort

**Ce travail de laboratoire est à faire en équipes de 2 personnes**

**ATTENTION : Commencez par créer un Fork de ce repo et travaillez sur votre fork.**

Clonez le repo sur votre machine. Vous pouvez répondre aux questions en modifiant directement votre clone du README.md ou avec un fichier pdf que vous pourrez uploader sur votre fork.

**Le rendu consiste simplement à répondre à toutes les questions clairement identifiées dans le text avec la mention "Question" et à les accompagner avec des captures. Le rendu doit se faire par une "pull request". Envoyer également le hash du dernier commit et votre username GitHub par email au professeur et à l'assistant**

## Table de matières

[Introduction](#introduction)

[Echéance](#echéance)

[Démarrage de l'environnement virtuel](#démarrage-de-lenvironnement-virtuel)

[Communication avec les conteneurs](#communication-avec-les-conteneurs)

[Configuration de la machine IDS et installation de Snort](#configuration-de-la-machine-ids-et-installation-de-snort)

[Essayer Snort](#essayer-snort)

[Utilisation comme IDS](#utilisation-comme-un-ids)

[Ecriture de règles](#ecriture-de-règles)

[Travail à effectuer](#exercises)

[Cleanup](#cleanup)


## Echéance

Ce travail devra être rendu au plus tard, **le 29 avril 2022 à 08h30.**


## Introduction

Dans ce travail de laboratoire, vous allez explorer un système de détection contre les intrusions (IDS) dont l'utilisation es très répandue grâce au fait qu'il est gratuit et open source. Il s'appelle [Snort](https://www.snort.org). Il existe des versions de Snort pour Linux et pour Windows.

### Les systèmes de détection d'intrusion

Un IDS peut "écouter" tout le traffic de la partie du réseau où il est installé. Sur la base d'une liste de règles, il déclenche des actions sur des paquets qui correspondent à la description de la règle.

Un exemple de règle pourrait être, en langage commun : "donner une alerte pour tous les paquets envoyés par le port http à un serveur web dans le réseau, qui contiennent le string 'cmd.exe'". En on peut trouver des règles très similaires dans les règles par défaut de Snort. Elles permettent de détecter, par exemple, si un attaquant essaie d'éxecuter un shell de commandes sur un serveur Web tournant sur Windows. On verra plus tard à quoi ressemblent ces règles.

Snort est un IDS très puissant. Il est gratuit pour l'utilisation personnelle et en entreprise, où il est très utilisé aussi pour la simple raison qu'il est l'un des systèmes IDS des plus efficaces.

Snort peut être exécuté comme un logiciel indépendant sur une machine ou comme un service qui tourne après chaque démarrage. Si vous voulez qu'il protège votre réseau, fonctionnant comme un IPS, il faudra l'installer "in-line" avec votre connexion Internet.

Par exemple, pour une petite entreprise avec un accès Internet avec un modem simple et un switch interconnectant une dizaine d'ordinateurs de bureau, il faudra utiliser une nouvelle machine éxecutant Snort et placée entre le modem et le switch.


## Matériel

Vous avez besoin de votre ordinateur avec Docker et docker-compose. Vous trouverez tous les fichiers nécessaires pour générer l'environnement pour virtualiser ce labo dans le projet que vous avez cloné.


## Démarrage de l'environnement virtuel

Ce laboratoire utilise docker-compose, un outil pour la gestion d'applications utilisant multiples conteneurs. Il va se charger de créer un réseaux virtuel `snortlan`, la machine IDS, un client avec un navigateur Firefox, une machine "Client" et un conteneur Wireshark directement connecté à la même interface réseau que la machine IDS. Le réseau LAN interconnecte les autres 3 machines (voir schéma ci-dessous).

![Plan d'adressage](images/docker-snort.png)

Nous allons commencer par lancer docker-compose. Il suffit de taper la commande suivante dans le répertoire racine du labo, celui qui contient le fichier [docker-compose.yml](docker-compose.yml). Optionnelement vous pouvez lancer le script [up.sh](scripts/up.sh) qui se trouve dans le répertoire [scripts](scripts), ainsi que d'autres scripts utiles pour vous :

```bash
docker-compose up --detach
```

Le téléchargement et génération des images prend peu de temps.

Les images utilisées pour les conteneurs client et la machine IDS sont basées sur l'image officielle Kali. Le fichier [Dockerfile](Dockerfile) que vous avez téléchargé contient les informations nécessaires pour la génération de l'image de base. [docker-compose.yml](docker-compose.yml) l'utilise comme un modèle pour générer ces conteneurs. Les autres deux conteneurs utilisent des images du groupe LinuxServer.io. Vous pouvez vérifier que les quatre conteneurs sont crées et qu'ils fonctionnent à l'aide de la commande suivante.

```bash
docker ps
```

## Communication avec les conteneurs

Afin de simplifier vos manipulations, les conteneurs ont été configurées avec les noms suivants :

- IDS
- Client
- wireshark
- firefox

Pour accéder au terminal de l’une des machines, il suffit de taper :

```bash
docker exec -it <nom_de_la_machine> /bin/bash
```

Par exemple, pour ouvrir un terminal sur votre IDS :

```bash
docker exec -it IDS /bin/bash
```

Optionnelement, vous pouvez utiliser les scripts [openids.sh](scripts/openids.sh), [openfirefox.sh](scripts/openfirefox.sh) et [openclient.sh](scripts/openclient.sh) pour contacter les conteneurs.

Vous pouvez bien évidemment lancer des terminaux communiquant avec toutes les machines en même temps ou même lancer plusieurs terminaux sur la même machine. ***Il est en fait conseillé pour ce laboratoire de garder au moins deux terminaux ouverts sur la machine IDS en tout moment***.


### Configuration de la machine Client et de firefox

Dans un terminal de votre machine Client et de la machine firefox, taper les commandes suivantes :

```bash
ip route del default
ip route add default via 192.168.220.2
```

Ceci configure la machine IDS comme la passerelle par défaut pour les deux autres machines.


## Configuration de la machine IDS et installation de Snort

Pour permettre à votre machine Client de contacter l'Internet à travers la machine IDS, il faut juste une petite règle NAT par intermédiaire de nftables :

```bash
nft add table nat
nft 'add chain nat postrouting { type nat hook postrouting priority 100 ; }'
nft add rule nat postrouting meta oifname "eth0" masquerade
```

Cette commande `iptables` définit une règle dans le tableau NAT qui permet la redirection de ports et donc, l'accès à l'Internet pour la machine Client.

On va maintenant installer Snort sur le conteneur IDS.

La manière la plus simple c'est d'installer Snort en ligne de commandes. Il suffit d'utiliser la commande suivante :

```
apt update && apt install snort
```

Ceci télécharge et installe la version la plus récente de Snort.

Il est possible que vers la fin de l'installation, on vous demande de fournir deux informations :

- Le nom de l'interface sur laquelle snort doit surveiller - il faudra répondre ```eth0```
- L'adresse de votre réseau HOME. Il s'agit du réseau que vous voulez protéger. Cela sert à configurer certaines variables pour Snort. Vous pouvez répondre ```192.168.220.0/24```.


## Essayer Snort

Une fois installé, vous pouvez lancer Snort comme un simple "sniffer". Pourtant, ceci capture tous les paquets, ce qui peut produire des fichiers de capture énormes si vous demandez de les journaliser. Il est beaucoup plus efficace d'utiliser des règles pour définir quel type de trafic est intéressant et laisser Snort ignorer le reste.

Snort se comporte de différentes manières en fonction des options que vous passez en ligne de commande au démarrage. Vous pouvez voir la grande liste d'options avec la commande suivante :

```
snort --help
```

On va commencer par observer tout simplement les entêtes des paquets IP utilisant la commande :

```
snort -v -i eth0
```

**ATTENTION : le choix de l'interface devient important si vous avez une machine avec plusieurs interfaces réseau. Dans notre cas, vous pouvez ignorer entièrement l'option ```-i eth0```et cela devrait quand-même fonctionner correctement.**

Snort s'éxecute donc et montre sur l'écran tous les entêtes des paquets IP qui traversent l'interface eth0. Cette interface reçoit tout le trafic en provenance de la machine "Client" puisque nous avons configuré le IDS comme la passerelle par défaut.

Pour arrêter Snort, il suffit d'utiliser `CTRL-C` (**attention** : en ligne générale, ceci fonctionne si vous patientez un moment... Snort est occupé en train de gérer le contenu du tampon de communication et cela qui peut durer quelques secondes. Cependant, il peut arriver de temps à autres que Snort ne réponde plus correctement au signal d'arrêt. Dans ce cas-là, il faudra utiliser `kill` depuis un deuxième terminal pour arrêter le process).


## Utilisation comme un IDS

Pour enregistrer seulement les alertes et pas tout le trafic, on execute Snort en mode IDS. Il faudra donc spécifier un fichier contenant des règles.

Il faut noter que `/etc/snort/snort.config` contient déjà des références aux fichiers de règles disponibles avec l'installation par défaut. Si on veut tester Snort avec des règles simples, on peut créer un fichier de config personnalisé (par exemple `mysnort.conf`) et importer un seul fichier de règles utilisant la directive "include".

Les fichiers de règles sont normalement stockes dans le répertoire `/etc/snort/rules/`, mais en fait un fichier de config et les fichiers de règles peuvent se trouver dans n'importe quel répertoire de la machine.

Par exemple, créez un fichier de config `mysnort.conf` dans le repertoire `/etc/snort` avec le contenu suivant :

```
include /etc/snort/rules/icmp2.rules
```

Ensuite, créez le fichier de règles `icmp2.rules` dans le repertoire `/etc/snort/rules/` et rajoutez dans ce fichier le contenu suivant :

`alert icmp any any -> any any (msg:"ICMP Packet"; sid:4000001; rev:3;)`

On peut maintenant éxecuter la commande :

```
snort -c /etc/snort/mysnort.conf
```

Vous pouvez maintenant faire quelques pings depuis votre "Client" et regarder les résultas dans le fichier d'alertes contenu dans le repertoire `/var/log/snort/`.


## Ecriture de règles

Snort permet l'écriture de règles qui décrivent des tentatives de exploitation de vulnérabilités bien connues. Les règles Snort prennent en charge à la fois, l'analyse de protocoles et la recherche et identification de contenu.

Il y a deux principes de base à respecter :

* Une règle doit être entièrement contenue dans une seule ligne
* Les règles sont divisées en deux sections logiques : (1) l'entête et (2) les options.

L'entête de la règle contient l'action de la règle, le protocole, les adresses source et destination, et les ports source et destination.

L'option contient des messages d'alerte et de l'information concernant les parties du paquet dont le contenu doit être analysé. Par exemple:

```
alert tcp any any -> 192.168.220.0/24 111 (content:"|00 01 86 a5|"; msg: "mountd access";)
```

Cette règle décrit une alerte générée quand Snort trouve un paquet avec tous les attributs suivants :

* C'est un paquet TCP
* Emis depuis n'importe quelle adresse et depuis n'importe quel port
* A destination du réseau identifié par l'adresse 192.168.220.0/24 sur le port 111

Le text jusqu'au premier parenthèse est l'entête de la règle.

```
alert tcp any any -> 192.168.220.0/24 111
```

Les parties entre parenthèses sont les options de la règle:

```
(content:"|00 01 86 a5|"; msg: "mountd access";)
```

Les options peuvent apparaître une ou plusieurs fois. Par exemple :

```
alert tcp any any -> any 21 (content:"site exec"; content:"%"; msg:"site
exec buffer overflow attempt";)
```

La clé "content" apparait deux fois parce que les deux strings qui doivent être détectés n'apparaissent pas concaténés dans le paquet mais a des endroits différents. Pour que la règle soit déclenchée, il faut que le paquet contienne **les deux strings** "site exec" et "%".

Les éléments dans les options d'une règle sont traités comme un AND logique. La liste complète de règles sont traitées comme une succession de OR.

## Informations de base pour le règles

### Actions :

```
alert tcp any any -> any any (msg:"My Name!"; content:"Skon"; sid:1000001; rev:1;)
```

L'entête contient l'information qui décrit le "qui", le "où" et le "quoi" du paquet. Ça décrit aussi ce qui doit arriver quand un paquet correspond à tous les contenus dans la règle.

Le premier champ dans le règle c'est l'action. L'action dit à Snort ce qui doit être fait quand il trouve un paquet qui correspond à la règle. Il y a six actions :

* alert - générer une alerte et écrire le paquet dans le journal
* log - écrire le paquet dans le journal
* pass - ignorer le paquet
* drop - bloquer le paquet et l'ajouter au journal
* reject - bloquer le paquet, l'ajouter au journal et envoyer un `TCP reset` si le protocole est TCP ou un `ICMP port unreachable` si le protocole est UDP
* sdrop - bloquer le paquet sans écriture dans le journal

### Protocoles :

Le champ suivant c'est le protocole. Il y a trois protocoles IP qui peuvent être analysés par Snort : TCP, UDP et ICMP.


### Adresses IP :

La section suivante traite les adresses IP et les numéros de port. Le mot `any` peut être utilisé pour définir "n'import quelle adresse". On peut utiliser l'adresse d'une seule machine ou un block avec la notation CIDR.

Un opérateur de négation peut être appliqué aux adresses IP. Cet opérateur indique à Snort d'identifier toutes les adresses IP sauf celle indiquée. L'opérateur de négation est le `!`.

Par exemple, la règle du premier exemple peut être modifiée pour alerter pour le trafic dont l'origine est à l'extérieur du réseau :

```
alert tcp !192.168.220.0/24 any -> 192.168.220.0/24 111
(content: "|00 01 86 a5|"; msg: "external mountd access";)
```

### Numéros de Port :

Les ports peuvent être spécifiés de différentes manières, y-compris `any`, une définition numérique unique, une plage de ports ou une négation.

Les plages de ports utilisent l'opérateur `:`, qui peut être utilisé de différentes manières aussi :

```
log udp any any -> 192.168.220.0/24 1:1024
```

Journaliser le traffic UDP venant d'un port compris entre 1 et 1024.

--

```
log tcp any any -> 192.168.220.0/24 :6000
```

Journaliser le traffic TCP venant d'un port plus bas ou égal à 6000.

--

```
log tcp any :1024 -> 192.168.220.0/24 500:
```

Journaliser le traffic TCP venant d'un port privilégié (bien connu) plus grand ou égal à 500 mais jusqu'au port 1024.


### Opérateur de direction

L'opérateur de direction `->`indique l'orientation ou la "direction" du trafique.

Il y a aussi un opérateur bidirectionnel, indiqué avec le symbole `<>`, utile pour analyser les deux côtés de la conversation. Par exemple un échange telnet :

```
log 192.168.220.0/24 any <> 192.168.220.0/24 23
```

## Alertes et logs Snort

Si Snort détecte un paquet qui correspond à une règle, il envoie un message d'alerte ou il journalise le message. Les alertes peuvent être envoyées au syslog, journalisées dans un fichier text d'alertes ou affichées directement à l'écran.

Le système envoie **les alertes vers le syslog** et il peut en option envoyer **les paquets "offensifs" vers une structure de repertoires**.

Les alertes sont journalisées via syslog dans le fichier `/var/log/snort/alerts`. Toute alerte se trouvant dans ce fichier aura son paquet correspondant dans le même repertoire, mais sous le fichier `snort.log.xxxxxxxxxx` où `xxxxxxxxxx` est l'heure Unix du commencement du journal.

Avec la règle suivante :

```
alert tcp any any -> 192.168.220.0/24 111
(content:"|00 01 86 a5|"; msg: "mountd access";)
```

un message d'alerte est envoyé à syslog avec l'information "mountd access". Ce message est enregistré dans `/var/log/snort/alerts` et le vrai paquet responsable de l'alerte se trouvera dans un fichier dont le nom sera `/var/log/snort/snort.log.xxxxxxxxxx`.

Les fichiers log sont des fichiers binaires enregistrés en format pcap. Vous pouvez les ouvrir avec Wireshark ou les diriger directement sur la console avec la commande suivante :

```
tcpdump -r /var/log/snort/snort.log.xxxxxxxxxx
```

Vous pouvez aussi utiliser des captures Wireshark ou des fichiers snort.log.xxxxxxxxx comme source d'analyse por Snort.

## Exercises

**Réaliser des captures d'écran des exercices suivants et les ajouter à vos réponses.**

### Essayer de répondre à ces questions en quelques mots, en réalisant des recherches sur Internet quand nécessaire :

**Question 1: Qu'est ce que signifie les "preprocesseurs" dans le contexte de Snort ?**

---

**Réponse :** 

Selon la [documentation officielle](https://www.snort.org/documents/snort-users-manual), les préprocesseurs permettent d'étendre les fonctionnalités de Snort en permettant aux utilisateurs et aux programmeurs de déposer assez facilement des plugins modulaires dans Snort. Le code du préprocesseur est exécuté avant que le moteur de détection ne soit appelé, mais après le traitement du paquet.

---

**Question 2: Pourquoi êtes vous confronté au WARNING suivant `"No preprocessors configured for policy 0"` lorsque vous exécutez la commande `snort` avec un fichier de règles ou de configuration "fait-maison" ?**

---

**Réponse :**  

Parce que le fichier de configuration "fait-maison" (par ex. `mysnort.conf`) ne contient pas de directives `preprocessor <name>: <options>`.

---

--

### Trouver du contenu :

Considérer la règle simple suivante:

alert tcp any any -> any any (msg:"Mon nom!"; content:"Rubinstein"; sid:4000015; rev:1;)

**Question 3: Qu'est-ce qu'elle fait la règle et comment ça fonctionne ?**

---

**Réponse :**  

Si un paquet TCP ayant n'importe quel port et n'importe quelle adresse ip comme source ou destination contient "Rubinstein" dans la payload, un message d'alerte avec comme information "Mon nom!" est généré et le paquet est écrit dans le journal.

---

Utiliser nano pour créer un fichier `myrules.rules` sur votre répertoire home (```/root```). Rajouter une règle comme celle montrée avant mais avec votre text, phrase ou mot clé que vous aimeriez détecter. Lancer Snort avec la commande suivante :

```
snort -c myrules.rules -i eth0 -k none
```

**Question 4: Que voyez-vous quand le logiciel est lancé ? Qu'est-ce que tous ces messages affichés veulent dire ?**

---

**Réponse :**  

```
Running in IDS mode

        --== Initializing Snort ==--
Initializing Output Plugins!
Initializing Preprocessors!
Initializing Plug-ins!
Parsing Rules file "myrules.rules"
Tagged Packet Limit: 256
Log directory = /var/log/snort

+++++++++++++++++++++++++++++++++++++++++++++++++++
Initializing rule chains...
1 Snort rules read
    1 detection rules
    0 decoder rules
    0 preprocessor rules
1 Option Chains linked into 1 Chain Headers
+++++++++++++++++++++++++++++++++++++++++++++++++++

+-------------------[Rule Port Counts]---------------------------------------
|             tcp     udp    icmp      ip
|     src       0       0       0       0
|     dst       0       0       0       0
|     any       1       0       0       0
|      nc       0       0       0       0
|     s+d       0       0       0       0
+----------------------------------------------------------------------------

+-----------------------[detection-filter-config]------------------------------
| memory-cap : 1048576 bytes
+-----------------------[detection-filter-rules]-------------------------------
| none
-------------------------------------------------------------------------------

+-----------------------[rate-filter-config]-----------------------------------
| memory-cap : 1048576 bytes
+-----------------------[rate-filter-rules]------------------------------------
| none
-------------------------------------------------------------------------------

+-----------------------[event-filter-config]----------------------------------
| memory-cap : 1048576 bytes
+-----------------------[event-filter-global]----------------------------------
+-----------------------[event-filter-local]-----------------------------------
| none
+-----------------------[suppression]------------------------------------------
| none
-------------------------------------------------------------------------------
Rule application order: pass->drop->sdrop->reject->alert->log
Verifying Preprocessor Configurations!

[ Port Based Pattern Matching Memory ]
+-[AC-BNFA Search Info Summary]------------------------------
| Instances        : 1
| Patterns         : 1
| Pattern Chars    : 8
| Num States       : 8
| Num Match States : 1
| Memory           :   1.62Kbytes
|   Patterns       :   0.05K
|   Match Lists    :   0.09K
|   Transitions    :   1.09K
+-------------------------------------------------
pcap DAQ configured to passive.
Acquiring network traffic from "eth0".
Reload thread starting...
Reload thread started, thread 0x7f6a2849c640 (20)
Decoding Ethernet

        --== Initialization Complete ==--

   ,,_     -*> Snort! <*-
  o"  )~   Version 2.9.15.1 GRE (Build 15125) 
   ''''    By Martin Roesch & The Snort Team: http://www.snort.org/contact#team
           Copyright (C) 2014-2019 Cisco and/or its affiliates. All rights reserved.
           Copyright (C) 1998-2013 Sourcefire, Inc., et al.
           Using libpcap version 1.10.1 (with TPACKET_V3)
           Using PCRE version: 8.39 2016-06-14
           Using ZLIB version: 1.2.11

Commencing packet processing (pid=19)
```

On voit entre autres, dans l'ordre :
- Des informations sur l'initialisation de Snort, notamment le fichier d'origine des règles ainsi que le répertoire de log
- Le nombre de règles lues, en l'occurrence 1 règle de détection
- Les catégories de règles, en l'occurrence il y a une règle TCP
- L'interface depuis laquelle Snort écoute
- La version de Snort et d'autres logiciels utilisés
- Le PID de Snort

---

Aller à un site web contenant dans son text la phrase ou le mot clé que vous avez choisi (il faudra chercher un peu pour trouver un site en http... Si vous n'y arrivez pas, vous pouvez utiliser [http://neverssl.com](http://neverssl.com) et modifier votre votre règle pour détecter un morceau de text contenu dans le site).

Pour accéder à Firefox dans son conteneur, ouvrez votre navigateur web sur votre machine hôte et dirigez-le vers [http://localhost:4000](http://localhost:4000). Optionnellement, vous pouvez utiliser wget sur la machine client pour lancer la requête http ou le navigateur Web lynx - il suffit de taper ```lynx neverssl.com```. Le navigateur lynx est un navigateur basé sur text, sans interface graphique.

**Question 5: Que voyez-vous sur votre terminal quand vous chargez le site depuis Firefox ou la machine Client ?**

---

**Réponse :**  

Absolument rien (sauf des `No preprocessors configured for policy 0`.
Cependant, l'alerte est loggée dans le fichier `/var/log/snort/alert`.

---

Arrêter Snort avec `CTRL-C`.

**Question 6: Que voyez-vous quand vous arrêtez snort ? Décrivez en détail toutes les informations qu'il vous fournit.**

---

**Réponse :**  

Lorsqu'on arrête Snort, on voit des statistiques sur ce qui a été analysé, plus précisément :

- le nombre total de paquets traités, ainsi que la moyenne du nombre de paquets reçu par minutes et par secondes
- un résumé sur l'utilisation de la RAM
- le nombre total de paquets reçus, analysés, droppés, filtrés, en suspend, et injectés
- la répartition des protocoles décodés par Snort
- un résumé sur :
  - ce que Snort a effectué comme action (alerter, logger, laisser passer) sur les paquets.
  - ce qui n'a pas pu être traité dû aux limitations du système (mémoire RAM libre, temps de calcul, ...)
  - le verdict des paquets (accepté, bloqué, remplacé, ignoré, ...)

---


Aller au répertoire /var/log/snort. Ouvrir le fichier `alert`. Vérifier qu'il y ait des alertes pour votre text choisi.

**Question 7: A quoi ressemble l'alerte ? Qu'est-ce que chaque élément de l'alerte veut dire ? Décrivez-la en détail !**

---

**Réponse :**  

L'alerte ressemble à ceci :
```
[**] [1:4000001:1] Concerne l'HEIG-VD [**]
[Priority: 0] 
04/25-09:41:35.419896 193.134.221.132:80 -> 192.168.220.4:38724
TCP TTL:47 TOS:0x0 ID:37118 IpLen:20 DgmLen:1308 DF
***AP*** Seq: 0xB3450495  Ack: 0x1FD04902  Win: 0x1C5  TcpLen: 32
TCP Options (3) => NOP NOP TS: 4066533714 2833739280
```

On voit le SID et la REV (`[1:4000001:1]`) de la règle, le message (`Concerne l'HEIG-VD`), la priorité (`[Priority: 0]`), la date et l'heure de la réception du paquet (`04/25-09:41:35.419896`), l'ip/port source et destination (`193.134.221.132:80 -> 192.168.220.4:38724`), puis tous les détails (séquence, acknowledge, ...) du header du paquet TCP concerné.

---


--

### Detecter une visite à Wikipedia

Ecrire deux règles qui journalisent (sans alerter) chacune un message à chaque fois que Wikipedia est visité **SPECIFIQUEMENT DEPUIS VOTRE MACHINE CLIENT OU DEPUIS FIREFOX**. Chaque règle doit identifier quelle machine à réalisé la visite. Ne pas utiliser une règle qui détecte un string ou du contenu. Il faudra se baser sur d'autres paramètres.

**Question 8: Quelle est votre règle ? Où le message a-t'il été journalisé ? Qu'est-ce qui a été journalisé ?**

---

**Réponse :**  

Notre règle est la suivante :
```
portvar HTTP [80,443]
ipvar CLIENT 192.168.220.3
ipvar FIREFOX 192.168.220.4

ipvar WIKIPEDIA 91.198.174.192
log tcp $CLIENT any -> $WIKIPEDIA $HTTP (msg:"Wikipedia visited"; sid:40000002; rev:1;)
log tcp $FIREFOX any -> $WIKIPEDIA $HTTP (msg:"Wikipedia visited"; sid:40000003; rev:1;)
```

Tout le contenu qui répondait à la règle écrite ci-dessus a été journalisé :
`tcpdump -r snort.log.1651128924 ` :

```
reading from file snort.log.1651128924, link-type EN10MB (Ethernet), snapshot length 1514
06:55:29.586884 IP firefox.snortlan.32782 > text-lb.esams.wikimedia.org.https: Flags [P.], seq 2504514308:2504514523, ack 913394832, win 501, options [nop,nop,TS val 1175129956 ecr 2832372735], length 215
06:55:29.587502 IP firefox.snortlan.32782 > text-lb.esams.wikimedia.org.https: Flags [P.], seq 215:250, ack 1, win 501, options [nop,nop,TS val 1175129957 ecr 2832372735], length 35
06:55:29.589227 IP firefox.snortlan.32782 > text-lb.esams.wikimedia.org.https: Flags [P.], seq 250:333, ack 1, win 501, options [nop,nop,TS val 1175129958 ecr 2832372735], length 83
06:55:29.612648 IP firefox.snortlan.32782 > text-lb.esams.wikimedia.org.https: Flags [.], ack 1371, win 493, options [nop,nop,TS val 1175129982 ecr 2832388835], length 0
06:55:29.614543 IP firefox.snortlan.32782 > text-lb.esams.wikimedia.org.https: Flags [.], ack 2819, win 493, options [nop,nop,TS val 1175129984 ecr 2832388838], length 0
06:55:29.614825 IP firefox.snortlan.32782 > text-lb.esams.wikimedia.org.https: Flags [.], ack 4267, win 493, options [nop,nop,TS val 1175129984 ecr 2832388838], length 0
06:55:29.614947 IP firefox.snortlan.32782 > text-lb.esams.wikimedia.org.https: Flags [.], ack 5715, win 493, options [nop,nop,TS val 1175129984 ecr 2832388838], length 0
06:55:29.615698 IP firefox.snortlan.32782 > text-lb.esams.wikimedia.org.https: Flags [.], ack 8611, win 477, options [nop,nop,TS val 1175129985 ecr 2832388838], length 0
06:55:29.616416 IP firefox.snortlan.32782 > text-lb.esams.wikimedia.org.https: Flags [.], ack 15851, win 429, options [nop,nop,TS val 1175129986 ecr 2832388838], length 0
06:55:29.618242 IP firefox.snortlan.32782 > text-lb.esams.wikimedia.org.https: Flags [.], ack 17299, win 493, options [nop,nop,TS val 1175129987 ecr 2832388838], length 0
06:55:29.618364 IP firefox.snortlan.32782 > text-lb.esams.wikimedia.org.https: Flags [.], ack 18747, win 493, options [nop,nop,TS val 1175129988 ecr 2832388838], length 0
06:55:29.619873 IP firefox.snortlan.32782 > text-lb.esams.wikimedia.org.https: Flags [.], ack 20195, win 493, options [nop,nop,TS val 1175129989 ecr 2832388838], length 0
06:55:29.620474 IP firefox.snortlan.32782 > text-lb.esams.wikimedia.org.https: Flags [.], ack 21387, win 493, options [nop,nop,TS val 1175129990 ecr 2832388838], length 0
06:55:29.641705 IP firefox.snortlan.32782 > text-lb.esams.wikimedia.org.https: Flags [P.], seq 333:475, ack 21387, win 501, options [nop,nop,TS val 1175130011 ecr 2832388838], length 142
06:55:29.642003 IP firefox.snortlan.32782 > text-lb.esams.wikimedia.org.https: Flags [P.], seq 475:578, ack 21387, win 501, options [nop,nop,TS val 1175130011 ecr 2832388838], length 103
06:55:29.642066 IP firefox.snortlan.32782 > text-lb.esams.wikimedia.org.https: Flags [P.], seq 578:613, ack 21387, win 501, options [nop,nop,TS val 1175130011 ecr 2832388838], length 35
06:55:29.642176 IP firefox.snortlan.32782 > text-lb.esams.wikimedia.org.https: Flags [P.], seq 613:648, ack 21387, win 501, options [nop,nop,TS val 1175130011 ecr 2832388838], length 35
06:55:29.704932 IP firefox.snortlan.32782 > text-lb.esams.wikimedia.org.https: Flags [P.], seq 648:764, ack 21387, win 501, options [nop,nop,TS val 1175130074 ecr 2832388896], length 116
06:55:29.705060 IP firefox.snortlan.32782 > text-lb.esams.wikimedia.org.https: Flags [P.], seq 764:850, ack 21387, win 501, options [nop,nop,TS val 1175130074 ecr 2832388896], length 86
06:55:29.705088 IP firefox.snortlan.32782 > text-lb.esams.wikimedia.org.https: Flags [P.], seq 850:885, ack 21387, win 501, options [nop,nop,TS val 1175130074 ecr 2832388896], length 35
06:55:29.705534 IP firefox.snortlan.32782 > text-lb.esams.wikimedia.org.https: Flags [P.], seq 885:920, ack 21387, win 501, options [nop,nop,TS val 1175130075 ecr 2832388896], length 35
```
Nous pouvons remarquer que Wikipédia est hébergé chez Wikimedia (l'organisation mère).

---


### Détecter un ping d'un autre système

Ecrire une règle qui alerte à chaque fois que votre machine IDS **reçoit** un ping depuis une autre machine (n'import laquelle des autres machines de votre réseau). Assurez-vous que **ça n'alerte pas** quand c'est vous qui **envoyez** le ping depuis l'IDS vers un autre système !

**Question 9: Quelle est votre règle ?**

---

**Réponse :**  

```
ipvar IDS 192.168.220.2
ipvar LOCAL_NETWORK 192.168.220.0/24
alert icmp $LOCAL_NETWORK any -> $IDS any (msg:"PING ALERT !"; itype:8; sid:40000004; rev:1;)
```

---


**Question 10: Comment avez-vous fait pour que ça identifie seulement les pings entrants ?**

---

**Réponse :**  

Nous avons utilisé le mot clé `itype:8` (ICMP type 8 = requête echo) pour éviter de détecter les réponses à des pings.

---


**Question 11: Où le message a-t-il été journalisé ?**

---

**Réponse :**  


Dans `var/log/snort/snort.log.xxxxxxxxxx`

(une alerte a également été ajoutée dans `/var/log/snort/alert`)

---

Les journaux sont générés en format pcap. Vous pouvez donc les lire avec Wireshark. Vous pouvez utiliser le conteneur wireshark en dirigeant le navigateur Web de votre hôte sur vers [http://localhost:3000](http://localhost:3000). Optionnellement, vous pouvez lire les fichiers log utilisant la commande `tshark -r nom_fichier_log` depuis votre IDS.

**Question 12: Qu'est-ce qui a été journalisé ?**

---

**Réponse :**  

Les paquets entiers qui correspondent à une ou plusieurs règle(s) ont été journalisés.
On peut donc relire les headers et les payloads.

`tcpdump -r snort.log.1651133359`:
```
reading from file snort.log.1651133359, link-type EN10MB (Ethernet), snapshot length 1514
08:09:21.497813 IP firefox.snortlan > IDS: ICMP echo request, id 5903, seq 0, length 64
08:09:22.498073 IP firefox.snortlan > IDS: ICMP echo request, id 5903, seq 1, length 64
08:09:23.498305 IP firefox.snortlan > IDS: ICMP echo request, id 5903, seq 2, length 64
```

`tshark -r snort.log.1651133359`:
```
Running as user "root" and group "root". This could be dangerous.
    1   0.000000 192.168.220.4 ? 192.168.220.2 ICMP 98 Echo (ping) request  id=0x170f, seq=0/0, ttl=64
    2   1.000260 192.168.220.4 ? 192.168.220.2 ICMP 98 Echo (ping) request  id=0x170f, seq=1/256, ttl=64
    3   2.000492 192.168.220.4 ? 192.168.220.2 ICMP 98 Echo (ping) request  id=0x170f, seq=2/512, ttl=64
```

---

--

### Detecter les ping dans les deux sens

Faites le nécessaire pour que les pings soient détectés dans les deux sens.

**Question 13: Qu'est-ce que vous avez fait pour détecter maintenant le trafic dans les deux sens ?**

---

**Réponse :**  

(Nous partons du principe que, comme pour la question 9, on veut détecter seulement des pings de/vers des machines de notre réseau.)

Il suffit de changer `->` par `<>`:

```
ipvar IDS 192.168.220.2
ipvar LOCAL_NETWORK 192.168.220.0/24
alert icmp $LOCAL_NETWORK any <> $IDS any (msg:"PING ALERT !"; itype:8; sid:40000004; rev:1;)
```

---


--

### Detecter une tentative de login SSH

Essayer d'écrire une règle qui Alerte qu'une tentative de session SSH a été faite depuis la machine Client sur l'IDS.

**Question 14: Quelle est votre règle ? Montrer la règle et expliquer en détail comment elle fonctionne.**

---

**Réponse :**  

Notre règle est la suivante :
```
ipvar CLIENT 192.168.220.3
ipvar IDS 192.168.220.2
portvar SSH 22
alert tcp $CLIENT any -> $IDS $SSH (msg:"SSH ALERT !"; sid:40000005; rev:1;)
```
Elle nous alerte si un paquet provient de la machine `Client` et est à destination du port 22 (SSH) de notre IDS.

---


**Question 15: Montrer le message enregistré dans le fichier d'alertes.**

---

**Réponse :**  

```
[**] [1:40000005:1] SSH ALERT ! [**]
[Priority: 0] 
04/28-08:26:30.555510 192.168.220.3:35946 -> 192.168.220.2:22
TCP TTL:64 TOS:0x10 ID:3943 IpLen:20 DgmLen:60 DF
******S* Seq: 0x3BF2EA7E  Ack: 0x0  Win: 0xFAF0  TcpLen: 40
TCP Options (5) => MSS: 1460 SackOK TS: 2609854087 0 NOP WS: 7
```

---

--

### Analyse de logs

Depuis l'IDS, servez-vous de l'outil ```tshark```pour capturer du trafic dans un fichier. ```tshark``` est une version en ligne de commandes de ```Wireshark```, sans interface graphique.

Pour lancer une capture dans un fichier, utiliser la commande suivante :

```
tshark -w nom_fichier.pcap
```

Générez du trafic depuis le deuxième terminal qui corresponde à l'une des règles que vous avez ajoutées à votre fichier de configuration personnel. Arrêtez la capture avec ```Ctrl-C```.

**Question 16: Quelle est l'option de Snort qui permet d'analyser un fichier pcap ou un fichier log ?**

---

**Réponse :**  

L'option `-r`.
Utilisation : `snort -r nom_fichier.{pcap|log}`

---

Utiliser l'option correcte de Snort pour analyser le fichier de capture Wireshark que vous venez de générer.

**Question 17: Quelle est le comportement de Snort avec un fichier de capture ? Y-a-t'il une différence par rapport à l'analyse en temps réel ?**

---

**Réponse :**  

Si Snort est lancé en mode IDS sur un fichier de capture (`snort -c rules -r capture.pcap`) :
- Snort a le même comportement que l'analyse en temps réel, sauf que les paquets sont lus depuis la capture
- Des alertes et journalisations sont générées comme si les paquets venaient de l'analyse en temps réel
- Après avoir lu la capture, Snort se termine automatiquement

Si Snort est lancé en mode sniffer (uniquement l'option `-r`) sur un fichier de capture (`snort -r capture.pcap`),
Snort affiche dans la console le détail de chaque paquet
séparé par `=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+` (voir exemple ci-dessous) et il ne génère aucun log ni
aucune alerte, contrairement à l'analyse en temps réelle.
Exemple :

```
WARNING: No preprocessors configured for policy 0.
04/28-08:38:31.856358 192.168.220.4 -> 192.168.220.2
ICMP TTL:64 TOS:0x0 ID:54502 IpLen:20 DgmLen:84 DF
Type:8  Code:0  ID:7438   Seq:0  ECHO
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/28-08:38:31.856413 192.168.220.2 -> 192.168.220.4
ICMP TTL:64 TOS:0x0 ID:40635 IpLen:20 DgmLen:84
Type:0  Code:0  ID:7438  Seq:0  ECHO REPLY
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/28-08:38:32.856574 192.168.220.4 -> 192.168.220.2
ICMP TTL:64 TOS:0x0 ID:54549 IpLen:20 DgmLen:84 DF
Type:8  Code:0  ID:7438   Seq:1  ECHO
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+

WARNING: No preprocessors configured for policy 0.
04/28-08:38:32.856608 192.168.220.2 -> 192.168.220.4
ICMP TTL:64 TOS:0x0 ID:40646 IpLen:20 DgmLen:84
Type:0  Code:0  ID:7438  Seq:1  ECHO REPLY
=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
```


---

**Question 18: Est-ce que des alertes sont aussi enregistrées dans le fichier d'alertes?**

---

**Réponse :**  

Non, sauf si Snort est lancé en mode IDS avec `-c`.

---

--

### Contournement de la détection

Faire des recherches à propos des outils `fragroute` et `fragrouter`.

**Question 19: A quoi servent ces deux outils ?**

---

**Réponse :**  

`fragroute` est un outil qui intercepte, modifie, et re-écrit le trafic de sortie de notre machine vers une cible, ce qui permet d'effectuer certaines attaques. Il permet notamment de :
- Retarder l'envoi de paquets
- Droper des paquets
- Dupliquer des paquets
- Fragmenter les paquets au niveau du protocole IP
- Fragmenter les paquets aun niveau du protocole TCP

`fragrouter` est un programme avec des fonctionnalités similaires, qui intercepte le traffic sortant et le modifie de maninère à essayer d'éviter qu'il soit détecté par un NIDS. Il peut notamment fragmenter les paquets au niveau IP et TCP, changer leur ordre, etc.

---


**Question 20: Quel est le principe de fonctionnement ?**

---

**Réponse :**  

Le but est de jouer avec les spécifications des protocoles réseaux pour essayer d'envoyer des messages qui seront correctement interprétés pas la cible, mais qui ne seront pas correctement analysés par l'IDS.

Notamment, les protocoles TCP et IP autorisent de fragmenter une payload en plusieurs paquets qui seront reconstitués par le destinataire. Si l'IDS ne fait pas correctement ce travail de reconstitution de paquets fragmentés, cela peut permettre à un attaquant d'envoyer du contenu qui ne sera pas détecté par l'IDS.

---


**Question 21: Qu'est-ce que le `Frag3 Preprocessor` ? A quoi ça sert et comment ça fonctionne ?**

---

**Réponse :**  

Le préprocesseur `Frag3` permet de détecter les paquets qui ont étés fragmentés.
Il fonctionne en récupérant tous les fragments de paquets pour ensuite reconstituer le paquet original.
Par la suite, le paquet original est analysé par Snort.

---


L'utilisation des outils ```Fragroute``` et ```Fragrouter``` nécessite une infrastructure un peu plus complexe. On va donc utiliser autre chose pour essayer de contourner la détection.

L'outil nmap propose une option qui fragmente les messages afin d'essayer de contourner la détection des IDS. Générez une règle qui détecte un SYN scan sur le port 22 de votre IDS.


**Question 22: A quoi ressemble la règle que vous avez configurée ?**

---

**Réponse :**  

```
ipvar IDS 192.168.220.2
portvar SSH 22
alert tcp any any -> $IDS $SSH (msg:"SYN packet on SSH port"; flags:S; sid:40000006; rev:1;)
```

---


Ensuite, servez-vous du logiciel nmap pour lancer un SYN scan sur le port 22 depuis la machine Client :

```
nmap -sS -p 22 192.168.220.2
```
Vérifiez que votre règle fonctionne correctement pour détecter cette tentative.

Ensuite, modifiez votre commande nmap pour fragmenter l'attaque :

```
nmap -sS -f -p 22 --send-eth 192.168.220.2
```

**Question 23: Quel est le résultat de votre tentative ?**

---

**Réponse :**  

Le paquet SYN est détecté si et seulement si le paquet n'est pas fragmenté.
```
[**] [1:40000006:1] SYN packet on SSH port [**]
[Priority: 0] 
04/28-11:31:38.142265 192.168.220.3:48255 -> 192.168.220.2:22
TCP TTL:43 TOS:0x0 ID:11034 IpLen:20 DgmLen:44
******S* Seq: 0xE936B454  Ack: 0x0  Win: 0x400  TcpLen: 24
TCP Options (1) => MSS: 1460
```

---


Modifier le fichier `myrules.rules` pour que snort utiliser le `Frag3 Preprocessor` et refaire la tentative.


**Question 24: Quel est le résultat ?**

---

**Réponse :**  

Le paquet fragmenté est détecté:
```
[**] [1:40000006:1] SYN packet on SSH port [**]
[Priority: 0] 
04/28-11:42:21.322002 192.168.220.3:47096 -> 192.168.220.2:22
TCP TTL:45 TOS:0x0 ID:4639 IpLen:20 DgmLen:44
******S* Seq: 0x7885591A  Ack: 0x0  Win: 0x400  TcpLen: 24
TCP Options (1) => MSS: 1460
```

---


**Question 25: A quoi sert le `SSL/TLS Preprocessor` ?**

---

**Réponse :**  

Ce préprocesseur sert à détecter le trafic chiffré avec TLS/SSL pour que Snort arrête de l'inspecter.

C'est une bonne chose qu'il exsite, car le trafic chiffré est inutile pour l'IDs, car il risque de déclencher
des faux positifs et de gaspiller du temps de calcul.
---


**Question 26: A quoi sert le `Sensitive Data Preprocessor` ?**

---

**Réponse :**  

Il permet de détecter lorsque des informations personnelles identifiables comme des numéros de carte de crédit, numéro de sécurité social, etc. sont transmises sur le réseau. Il est aussi possible de masquer ces informations en les remplaçant partiellement pas des "X".

---

### Conclusions


**Question 27: Donnez-nous vos conclusions et votre opinion à propos de snort**

---

**Réponse :**  

Snort est un outil très puissant, mais la documentation est assez pauvre et, par conséquent, il est difficile de le configurer avec des règles très précises.

---

### Cleanup

Pour nettoyer votre système et effacer les fichiers générés par Docker, vous pouvez exécuter le script [cleanup.sh](scripts/cleanup.sh). **ATTENTION : l'effet de cette commande est irréversible***.


<sub>This guide draws heavily on http://cs.mvnu.edu/twiki/bin/view/Main/CisLab82014</sub>

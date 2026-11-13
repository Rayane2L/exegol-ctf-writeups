# Cookie Monster — user.txt

## 1. Flag

```text
EPI{********}
```

## 2. Objectif

L'objectif du challenge est d'exploiter l'application web afin d'obtenir un accès au système avec l'utilisateur `cookie-monster` et de récupérer le fichier `user.txt`.

L'exploitation repose principalement sur une **SQL Injection UNION-based**, qui permet ensuite de transformer l'accès à la base de données en lecture de fichiers puis en exécution de code sur le serveur.

---

## 3. Techniques utilisées

* SQL Injection
* UNION-based SQL Injection
* Bypass de WAF
* Énumération de bases de données
* Énumération de tables
* Lecture arbitraire de fichiers
* Exploitation du privilège MySQL `FILE`
* `LOAD_FILE`
* `INTO OUTFILE`
* Webshell PHP
* Remote Code Execution (RCE)
* Énumération de fichiers locaux
* Analyse de logs SSH
* Récupération de credentials
* `su`
* Élévation vers un compte utilisateur

---

## 4. Outils utilisés

* Navigateur web
* `curl`
* SQL
* MySQL
* PHP
* Webshell
* Shell Linux
* `su`

---

## 5. SQL Injection sur le cookie `id`

L'application utilise un cookie nommé :

```text
id
```

L'analyse du comportement de l'application permet d'identifier une injection SQL.

Le nombre de colonnes nécessaire pour une injection `UNION` est déterminé à **2 colonnes**.

La vulnérabilité permet donc d'injecter des requêtes SQL supplémentaires dans le traitement du cookie.

---

## 6. Bypass du WAF

Un mécanisme de filtrage bloque certains motifs SQL.

Le WAF détecte notamment les mots-clés lorsqu'ils sont écrits avec une casse précise.

Le contournement consiste à modifier la casse des mots-clés SQL afin que les motifs recherchés par le WAF ne correspondent plus.

Exemples utilisés :

```text
SeLeCT
GrOuP_CoNcAt
```

Cette technique permet de conserver une requête SQL valide tout en contournant le filtrage mis en place.

---

## 7. Énumération de la base de données

L'injection `UNION-based` permet ensuite d'interroger les informations internes de MySQL.

La base de données intéressante est :

```text
cookies
```

L'énumération des tables révèle une seule table :

```text
queue
```

Cette table s'avère être un leurre destiné à donner l'impression qu'une véritable donnée intéressante se trouve dans la base.

Elle correspond en réalité à une table utilisée comme mécanisme anti-bruteforce.

---

## 8. Lecture arbitraire de fichiers avec `LOAD_FILE`

L'analyse des privilèges de la base révèle que le privilège :

```text
FILE
```

est activé.

La configuration :

```text
secure_file_priv
```

est également vide.

Cette configuration permet à MySQL d'accéder à des fichiers du système de fichiers sans restriction de répertoire imposée par `secure_file_priv`.

La fonction :

```sql
LOAD_FILE()
```

peut donc être utilisée pour lire des fichiers locaux.

---

## 9. Lecture de `/etc/passwd`

La lecture de `/etc/passwd` permet de confirmer la possibilité de lire arbitrairement des fichiers :

```text
/etc/passwd
```

Cette lecture révèle notamment l'existence de l'utilisateur :

```text
cookie-monster
```

L'objectif devient alors d'obtenir un accès à ce compte.

---

## 10. RCE avec `INTO OUTFILE`

Le privilège `FILE` ne permet pas seulement de lire des fichiers.

Il permet également d'écrire des fichiers avec :

```sql
INTO OUTFILE
```

Cette fonctionnalité peut être utilisée pour déposer un fichier PHP dans le répertoire servi par Apache.

Le fichier choisi est :

```text
/var/www/html/pwn123.php
```

Le nom `shell.php` n'est pas utilisé car un fichier portant déjà ce nom est présent sur le serveur et correspond à un leurre volontairement cassé.

Le nouveau fichier PHP sert de **webshell**.

---

## 11. Obtention d'une RCE

Une fois le fichier PHP écrit dans :

```text
/var/www/html/pwn123.php
```

il devient accessible via le serveur web.

L'exécution du webshell permet d'obtenir des commandes système avec les privilèges du processus Apache :

```text
www-data
```

L'exécution est donc confirmée avec l'utilisateur :

```text
www-data
```

La SQL Injection a ainsi été transformée en **Remote Code Execution** sur le serveur.

La chaîne est maintenant :

```text
SQL Injection
    ↓
UNION-based SQLi
    ↓
FILE privilege
    ↓
INTO OUTFILE
    ↓
Webshell PHP
    ↓
RCE
    ↓
www-data
```

---

## 12. Recherche d'informations sur `cookie-monster`

Une fois le shell obtenu avec `www-data`, les fichiers accessibles au système sont énumérés.

Le fichier intéressant est découvert dans :

```text
/home/cookie-monster/
```

Il s'agit de :

```text
cookie_analysis
```

Le fichier est lisible par `www-data`.

Son contenu correspond à un log lié à des tentatives de connexion SSH.

---

## 13. Récupération du mot de passe

L'analyse du fichier `cookie_analysis` révèle une tentative de connexion dans laquelle le mot de passe a été saisi par erreur dans le champ destiné au nom d'utilisateur.

La valeur observée est :

```text
cookie-monsterGiv3_Me_M04r_Co0Ki3s
```

Cette information permet de séparer le nom d'utilisateur et le mot de passe :

```text
Utilisateur :
cookie-monster

Mot de passe :
Giv3_Me_M04r_Co0Ki3s
```

Le mot de passe peut donc être réutilisé pour obtenir un shell avec le compte `cookie-monster`.

---

## 14. Passage à `cookie-monster`

Depuis le shell `www-data`, la commande suivante est utilisée :

```bash
su cookie-monster
```

Avec le mot de passe récupéré :

```text
Giv3_Me_M04r_Co0Ki3s
```

L'authentification fonctionne et permet de passer de :

```text
www-data
```

à :

```text
cookie-monster
```

---

## 15. Récupération du flag

Une fois connecté en tant que `cookie-monster`, le fichier `user.txt` peut être lu.

Le flag récupéré est :

```text
EPI{********}
```

---

## 16. Vulnérabilités identifiées

### 16.1 SQL Injection

Le cookie `id` est vulnérable à une injection SQL permettant notamment l'utilisation de requêtes `UNION`.

### 16.2 WAF contournable

Le filtrage repose sur des motifs sensibles à la casse.

La variation de casse des mots-clés SQL permet de contourner le WAF :

```text
SeLeCT
GrOuP_CoNcAt
```

### 16.3 Privilège MySQL `FILE`

Le compte utilisé par l'application possède le privilège :

```text
FILE
```

Ce privilège permet notamment la lecture et l'écriture de fichiers selon les fonctionnalités utilisées.

### 16.4 `secure_file_priv` mal configuré

La valeur vide de :

```text
secure_file_priv
```

supprime la restriction normalement imposée aux opérations de fichiers de MySQL.

La combinaison avec `FILE` permet notamment d'utiliser `LOAD_FILE` et `INTO OUTFILE`.

### 16.5 RCE via `INTO OUTFILE`

L'écriture d'un fichier PHP dans :

```text
/var/www/html/
```

permet de transformer la capacité d'écriture MySQL en exécution de code côté serveur.

### 16.6 Divulgation de credentials

Le fichier :

```text
/home/cookie-monster/cookie_analysis
```

est lisible par `www-data` alors qu'il contient des informations permettant de retrouver le mot de passe du compte `cookie-monster`.

---

## 17. Chaîne d'exploitation

```text
Cookie id
    ↓
SQL Injection
    ↓
UNION-based SQLi
    ↓
Bypass du WAF avec variation de casse
    ↓
Base : cookies
    ↓
Table : queue
    ↓
Privilège FILE
    ↓
secure_file_priv vide
    ↓
LOAD_FILE()
    ↓
Lecture de /etc/passwd
    ↓
Découverte de cookie-monster
    ↓
INTO OUTFILE
    ↓
/var/www/html/pwn123.php
    ↓
Webshell PHP
    ↓
RCE
    ↓
www-data
    ↓
/home/cookie-monster/cookie_analysis
    ↓
Récupération du mot de passe
    ↓
su cookie-monster
    ↓
user.txt
    ↓
EPI{********}
```

---

## 18. Résumé

Le challenge commence par une **SQL Injection UNION-based** présente dans le cookie `id`.

Le WAF est contourné en modifiant la casse des mots-clés SQL. L'injection permet ensuite d'énumérer la base `cookies` et sa table `queue`.

Le compte MySQL possède le privilège `FILE` et `secure_file_priv` est vide. Il est donc possible d'utiliser `LOAD_FILE` pour lire `/etc/passwd`, ce qui permet notamment d'identifier l'utilisateur `cookie-monster`.

Le privilège `FILE` permet ensuite d'utiliser `INTO OUTFILE` afin d'écrire un webshell PHP dans `/var/www/html/pwn123.php`. L'exécution de ce webshell donne une RCE avec les privilèges de `www-data`.

Enfin, le fichier `/home/cookie-monster/cookie_analysis`, accessible depuis `www-data`, contient une tentative de connexion SSH révélant accidentellement le mot de passe de `cookie-monster`.

L'utilisation de `su` permet alors de passer sur le compte `cookie-monster` et de lire `user.txt`.

```text
EPI{********}
```

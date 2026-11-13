# CTF — jormungandr

## Flag

```text
EPI{********}
```

---

## 1. Périmètre et objectif

L'objectif du challenge était de récupérer le fichier :

```text
/home/jormungandr/user.txt
```

La question du challenge demandait uniquement le flag `user.txt`. L'objectif a donc été atteint après la récupération du flag utilisateur.

La méthodologie suivie repose sur plusieurs étapes :

* énumération des services réseau ;
* accès à un serveur FTP anonyme ;
* recherche de fichiers cachés ;
* analyse d'un fichier contenant des données sérialisées Python ;
* récupération d'identifiants SSH ;
* mouvement latéral vers un autre compte ;
* récupération d'un bytecode Python ;
* décompilation du programme ;
* analyse du code source ;
* découverte d'identifiants supplémentaires ;
* connexion à un service TCP custom ;
* exploitation d'une RCE ;
* lecture du fichier `user.txt`.

---

## 2. Techniques utilisées

* Network enumeration
* FTP enumeration
* Anonymous FTP access
* Hidden file discovery
* Python pickle analysis
* Data decoding
* Credential discovery
* SSH authentication
* Lateral movement
* Python bytecode analysis
* Python bytecode decompilation
* Source code analysis
* Hardcoded credential discovery
* Custom TCP service enumeration
* Remote Code Execution (RCE)
* Arbitrary command execution
* Local file disclosure

---

## 3. Outils utilisés

* `nmap`
* `ftp`
* `ls`
* `file`
* Python
* `pycdc`
* `ssh`
* `nc` / client TCP
* `cat`

---

# 4. Énumération réseau

La première étape consiste à identifier les services accessibles sur la machine cible.

```bash
nmap -sC -sV -p- <TARGET_IP>
```

L'énumération révèle notamment un service FTP accessible sur le port `21`.

Un service TCP custom est également découvert sur le port :

```text
7432
```

Le service présent sur ce port devient particulièrement intéressant après la découverte de son implémentation.

---

# 5. Accès FTP anonyme

Le service FTP permet une connexion anonyme.

```bash
ftp <TARGET_IP>
```

Une fois connecté, le contenu accessible est énuméré :

```bash
ls -la
```

Un fichier caché particulièrement intéressant est découvert :

```text
.yggdrasil.cache
```

Ce fichier ne correspond pas à un fichier texte classique et nécessite une analyse plus approfondie.

---

# 6. Analyse de `.yggdrasil.cache`

Le fichier `.yggdrasil.cache` est récupéré depuis le serveur FTP.

Son type est ensuite vérifié :

```bash
file .yggdrasil.cache
```

L'analyse du fichier révèle qu'il contient des données sérialisées avec **Python pickle**, sous une forme binaire représentée en ASCII.

Le contenu est décodé afin de récupérer les informations qu'il contient.

L'analyse permet de découvrir des identifiants SSH :

```text
Username: fenrir
Password: ********
```

Ces identifiants constituent le premier moyen d'accès à la machine.

---

# 7. Connexion SSH avec `fenrir`

Les identifiants récupérés sont utilisés pour se connecter au service SSH :

```bash
ssh fenrir@<TARGET_IP>
```

Une fois connecté, l'utilisateur courant est vérifié :

```bash
id
```

L'accès au compte `fenrir` permet ensuite d'énumérer son répertoire personnel à la recherche de fichiers intéressants.

---

# 8. Découverte de `mjolnir.pyc`

Dans le home de `fenrir`, un fichier particulièrement intéressant est découvert :

```text
mjolnir.pyc
```

Le fichier correspond à du **bytecode Python 3.8**.

Le fichier est récupéré localement afin d'être analysé et décompilé.

Le but est de comprendre le fonctionnement du service TCP découvert précédemment sur le port `7432`.

---

# 9. Décompilation du bytecode Python

Le bytecode Python est analysé avec `pycdc`.

```bash
pycdc mjolnir.pyc
```

La décompilation permet de retrouver le fonctionnement du programme sous une forme proche du code source original.

L'analyse du code révèle que le programme implémente un service réseau personnalisé accessible sur :

```text
TCP/7432
```

Plus important encore, le code contient directement des identifiants :

```text
Username: jormungandr
Password: ********
```

Ces identifiants permettent d'accéder au service avec les privilèges du compte `jormungandr`.

---

# 10. Connexion au service TCP

Le service personnalisé écoute sur le port `7432`.

Une connexion est établie afin d'interagir avec le programme :

```bash
nc <TARGET_IP> 7432
```

Les identifiants récupérés dans le code source sont utilisés :

```text
jormungandr
Jag_ar_Jormungandr_midgardsormen
```

L'authentification réussit.

---

# 11. Remote Code Execution

L'analyse du code de `mjolnir.pyc` révèle que le service exécute directement les commandes fournies par l'utilisateur.

Le service constitue donc une **Remote Code Execution (RCE)**.

Les commandes sont exécutées avec les privilèges du compte :

```text
jormungandr
```

Une vérification des privilèges permet de confirmer l'utilisateur utilisé pour l'exécution :

```bash
id
```

Résultat :

```text
uid=1001(jormungandr)
```

Le service permet donc l'exécution de commandes arbitraires à distance avec les privilèges de `jormungandr`.

---

# 12. Lecture de `user.txt`

Le fichier recherché se trouve dans le home de l'utilisateur :

```text
/home/jormungandr/user.txt
```

La RCE permet de lire directement le fichier :

```bash
cat /home/jormungandr/user.txt
```

Le contenu récupéré est :

```text
EPI{********}
```

Le flag utilisateur est donc :

```text
EPI{********}
```

---

# 13. Vulnérabilités et faiblesses identifiées

## 13.1 FTP anonyme

Le serveur FTP autorise l'accès anonyme à des fichiers internes.

Un fichier caché contenant des informations sensibles est ainsi accessible :

```text
.yggdrasil.cache
```

**Impact :** fuite d'informations et récupération d'identifiants.

---

## 13.2 Stockage non sécurisé d'identifiants

Le fichier `.yggdrasil.cache` contient des identifiants permettant une connexion SSH :

```text
fenrir / I_will_kill_odin_during_ragnarok
```

**Impact :** compromission du compte `fenrir`.

---

## 13.3 Bytecode Python accessible

Le fichier :

```text
mjolnir.pyc
```

est accessible depuis le compte `fenrir`.

Sa décompilation permet de récupérer la logique interne du service ainsi que des identifiants présents en dur dans le code.

**Impact :** divulgation du fonctionnement interne du service et des identifiants du compte `jormungandr`.

---

## 13.4 Identifiants hardcodés

Les identifiants suivants sont présents directement dans le programme :

```text
jormungandr
Jag_ar_Jormungandr_midgardsormen
```

**Impact :** accès au service avec les privilèges de `jormungandr`.

---

## 13.5 Remote Code Execution

Le service TCP sur le port `7432` exécute les commandes fournies par l'utilisateur sans filtrage suffisant.

Cela permet une exécution de commandes arbitraires avec les privilèges de `jormungandr`.

**Impact :** compromission du compte `jormungandr` et accès aux fichiers auxquels ce compte peut accéder.

---

# 14. Chaîne d'exploitation complète

```text
Nmap
  │
  ▼
FTP / TCP 7432
  │
  ▼
FTP anonyme
  │
  ▼
.yggdrasil.cache
  │
  ▼
Analyse / décodage Python pickle
  │
  ▼
Identifiants fenrir
  │
  ▼
SSH → fenrir
  │
  ▼
mjolnir.pyc
  │
  ▼
Décompilation avec pycdc
  │
  ▼
Code source du service TCP 7432
  │
  ▼
Identifiants jormungandr
  │
  ▼
Connexion au port 7432
  │
  ▼
RCE
  │
  ▼
jormungandr (uid=1001)
  │
  ▼
cat /home/jormungandr/user.txt
  │
  ▼
EPI{********}
```

---

# 15. Résumé

L'exploitation commence par l'énumération des services et la découverte d'un serveur FTP autorisant les connexions anonymes.

L'accès FTP permet de récupérer le fichier caché `.yggdrasil.cache`, qui contient des données sérialisées Python. Leur analyse permet de récupérer les identifiants du compte `fenrir`.

Après connexion en SSH avec `fenrir`, un fichier `mjolnir.pyc` est découvert. La décompilation de ce bytecode avec `pycdc` révèle le fonctionnement d'un service TCP personnalisé écoutant sur le port `7432`.

Le code contient également les identifiants du compte `jormungandr`.

Après authentification auprès du service, il est possible d'exécuter des commandes arbitraires à distance. Cette RCE est utilisée pour lire :

```text
/home/jormungandr/user.txt
```

Le flag obtenu est :

```text
EPI{********}
```

L'objectif du challenge étant uniquement de récupérer `user.txt`, l'exploitation est considérée comme terminée à cette étape.

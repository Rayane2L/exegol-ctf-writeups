# Jormungandr

## Flag

```text
EPI{********}
```

---

## 1. Périmètre et objectif

L'objectif de ce challenge était de récupérer le fichier `root.txt`.

L'accès initial utilisé est le compte `jormungandr`, obtenu lors du challenge précédent grâce à la RCE disponible sur le service TCP `7432`.

La méthodologie suivie repose sur :

* la stabilisation de l'accès SSH ;
* l'énumération des permissions `sudo` ;
* l'analyse d'un binaire exécuté avec les privilèges `root` ;
* l'identification d'un mécanisme de désérialisation Python `pickle` ;
* l'analyse des indices fournis par le programme ;
* la construction d'un payload `pickle` ;
* l'encodage du payload en Base64 ;
* l'exploitation d'une désérialisation non sécurisée ;
* l'exécution de commandes arbitraires avec les privilèges `root` ;
* la recherche puis la lecture du fichier `root.txt`.

---

## 2. Techniques utilisées

* Remote Code Execution
* SSH key injection
* Linux privilege enumeration
* `sudo` enumeration
* Binary analysis
* PyInstaller analysis
* Python `pickle` exploitation
* Insecure deserialization
* Base64 encoding
* Python `__reduce__` exploitation
* Command execution as root
* File enumeration
* Privileged file disclosure

---

## 3. Outils utilisés

* `ssh`
* `sudo`
* `base64`
* Python
* `pickle`
* `os.system`
* `cat`
* `ls`

---

# 4. Reprise de l'accès `jormungandr`

L'accès initial au compte `jormungandr` avait été obtenu lors du challenge précédent grâce à la RCE du service custom disponible sur le port `7432`.

Afin d'obtenir une session SSH plus stable, une clé publique SSH est injectée dans :

```text
/home/jormungandr/.ssh/authorized_keys
```

La clé est encodée en Base64 avant son transfert afin d'éviter les problèmes d'échappement lors de son injection.

Une connexion SSH stable peut ensuite être établie avec :

```bash
ssh jormungandr@<TARGET_IP>
```

Cette session est utilisée pour poursuivre l'énumération locale.

---

# 5. Énumération des privilèges sudo

La première étape consiste à vérifier les commandes que `jormungandr` peut exécuter avec `sudo` :

```bash
sudo -l
```

Une règle particulièrement intéressante est découverte :

```text
(root) NOPASSWD: /opt/ragnarok/ragnarok
```

Cela signifie que l'utilisateur `jormungandr` peut exécuter le programme :

```text
/opt/ragnarok/ragnarok
```

avec les privilèges `root`, sans fournir de mot de passe.

Le binaire devient donc le principal vecteur potentiel d'élévation de privilèges.

---

# 6. Analyse du binaire `ragnarok`

Le fichier suivant est étudié :

```text
/opt/ragnarok/ragnarok
```

L'analyse montre qu'il s'agit d'un exécutable **PyInstaller** contenant une application Python.

La version de Python utilisée est :

```text
Python 3.5
```

Lors de son exécution avec `sudo`, le programme demande un élément appelé **"meal"**.

```bash
sudo /opt/ragnarok/ragnarok
```

Le comportement du programme fournit plusieurs indices intéressants.

Une première erreur indique :

```text
include at least a pickle in it!
```

Cette erreur indique que le programme attend la présence d'un objet `pickle`.

Un autre message retourné par le programme est :

```text
64 times worst than I thought
```

Ce message constitue un indice supplémentaire vers un encodage en **Base64**.

Les deux indices permettent de déduire que le programme attend probablement :

```text
pickle → Base64
```

---

# 7. Analyse du mécanisme de désérialisation

Le fonctionnement du programme est analysé afin de déterminer la manière dont le "meal" est traité.

Le programme récupère les données fournies sur son entrée standard, les décode puis les désérialise avec Python `pickle`.

Le mécanisme utilisé revient à effectuer une opération équivalente à :

```python
pickle.loads(...)
```

Le problème est que `pickle` n'est pas conçu pour désérialiser des données provenant d'une source non fiable.

Lors de la désérialisation, certaines classes peuvent provoquer l'exécution de fonctions arbitraires.

Le vecteur d'exploitation repose donc sur une **insecure deserialization**.

---

# 8. Construction du payload Pickle

Pour déclencher l'exécution d'une commande système, un objet Python utilisant `__reduce__` est construit.

Le principe utilisé est équivalent à :

```python
import os

class Exploit:
    def __reduce__(self):
        return (os.system, ("<COMMAND>",))
```

Lors de la désérialisation, Python reconstruit l'objet et appelle :

```text
os.system("<COMMAND>")
```

Comme le programme `ragnarok` est lui-même lancé avec :

```text
sudo
```

et les privilèges :

```text
root
```

la commande exécutée par `os.system()` bénéficie également des privilèges `root`.

---

# 9. Compatibilité avec Python 3.5

Le binaire utilise Python 3.5.

Le payload `pickle` est donc sérialisé avec le **protocole 2**, afin d'assurer sa compatibilité avec l'environnement Python utilisé par le programme.

Le payload est ensuite encodé en Base64.

La chaîne d'exploitation devient :

```text
Objet Python
    ↓
pickle.dumps(..., protocol=2)
    ↓
Base64
    ↓
stdin de ragnarok
    ↓
pickle.loads(...)
    ↓
os.system(...)
    ↓
Commande exécutée en tant que root
```

---

# 10. Exécution de commandes en tant que root

Le payload est envoyé au programme exécuté avec `sudo` :

```bash
sudo /opt/ragnarok/ragnarok
```

Le programme désérialise le payload et déclenche l'appel :

```text
os.system(cmd)
```

La commande est donc exécutée avec les privilèges `root`.

Cela fournit une **RCE root**.

Une première utilisation consiste à énumérer le contenu du répertoire `/root/`.

```bash
ls -la /root/
```

Cette énumération révèle un détail important : le fichier recherché possède un **espace au début de son nom**.

Le fichier est donc différent du chemin habituel :

```text
/root/root.txt
```

et nécessite une recherche adaptée.

---

# 11. Recherche du fichier root.txt

Pour éviter les problèmes liés à l'espace présent au début du nom, une recherche avec un wildcard est utilisée :

```bash
cat /root/*root.txt*
```

Cette commande permet de cibler le fichier malgré son nom inhabituel.

Le contenu retourné est :

```text
EPI{********}
```

Le flag `root.txt` est donc récupéré.

---

# 12. Analyse de la vulnérabilité

La vulnérabilité principale repose sur la combinaison de deux mauvaises configurations.

### 12.1 Règle sudo trop permissive

La configuration autorise :

```text
jormungandr ALL=(root) NOPASSWD: /opt/ragnarok/ragnarok
```

L'utilisateur peut donc exécuter le programme avec les privilèges `root`.

### 12.2 Désérialisation non sécurisée

Le programme traite des données contrôlées par l'utilisateur avec `pickle.loads()`.

Or, un objet `pickle` peut déclencher l'exécution de fonctions lors de sa désérialisation.

La combinaison des deux vulnérabilités permet :

```text
jormungandr
    │
    ▼
sudo NOPASSWD
    │
    ▼
/opt/ragnarok/ragnarok
    │
    ▼
pickle.loads(user_input)
    │
    ▼
__reduce__()
    │
    ▼
os.system(command)
    │
    ▼
ROOT
```

Cette vulnérabilité peut être décrite comme une **insecure deserialization leading to arbitrary command execution**, aggravée par une règle `sudo` permettant l'exécution du programme vulnérable avec les privilèges `root`.

---

# 13. Chaîne d'exploitation complète

```text
Accès jormungandr
       │
       ▼
Injection de clé SSH
       │
       ▼
Session SSH stable
       │
       ▼
sudo -l
       │
       ▼
(root) NOPASSWD: /opt/ragnarok/ragnarok
       │
       ▼
Analyse du binaire
       │
       ▼
PyInstaller / Python 3.5
       │
       ▼
"include at least a pickle in it!"
       │
       ▼
"64 times worst than I thought"
       │
       ▼
Pickle + Base64
       │
       ▼
Payload __reduce__()
       │
       ▼
os.system(command)
       │
       ▼
RCE ROOT
       │
       ▼
ls -la /root/
       │
       ▼
Nom de fichier avec espace de tête
       │
       ▼
cat /root/*root.txt*
       │
       ▼
EPI{********}
```

---

# 14. Flag final

```text
EPI{********}
```

---

# 15. Conclusion

L'exploitation commence avec l'accès `jormungandr` obtenu lors du challenge précédent.

L'énumération `sudo` révèle que ce compte peut exécuter `/opt/ragnarok/ragnarok` avec les privilèges `root` sans mot de passe.

L'analyse du programme révèle qu'il s'agit d'une application Python packagée avec PyInstaller. Les messages d'erreur du programme fournissent des indices vers l'utilisation de `pickle` et de Base64.

La désérialisation de données contrôlées par l'utilisateur avec `pickle.loads()` permet de construire un objet utilisant `__reduce__` afin de déclencher `os.system()`.

Le programme étant exécuté par `sudo` avec les privilèges `root`, la commande arbitraire est elle-même exécutée en tant que `root`.

Cette RCE root permet finalement d'énumérer `/root/`, de découvrir le nom inhabituel du fichier contenant le flag, puis de le lire.

Le flag obtenu est :

```text
EPI{********}
```

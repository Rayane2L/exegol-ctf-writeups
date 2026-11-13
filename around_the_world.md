# Around The World

## Flag

```text id="n3j4vp"
EPI{********}
```

---

## 1. Périmètre et objectif

L'objectif du challenge était de récupérer le fichier :

```text id="f4d2xk"
/home/gm08/user.txt
```

L'application cible est une application web **Node.js / Express** appelée **Around The World**, accessible sur le port `5555`.

La méthodologie suivie repose sur :

* l'énumération de l'application web ;
* la création d'un compte ;
* l'analyse du système de monnaie virtuelle ;
* l'exploitation d'une race condition ;
* l'accumulation de `gold` ;
* l'achat du statut Premium ;
* l'exploitation de la calculatrice Premium ;
* l'identification d'une exécution de JavaScript côté serveur ;
* l'utilisation des modules Node.js `os` et `fs` ;
* la lecture du fichier `user.txt`.

---

## 2. Techniques utilisées

* Web enumeration
* Account registration
* Authentication
* Business logic analysis
* Race condition
* Concurrent HTTP requests
* Integer/resource manipulation
* Privilege/status escalation
* Server-Side JavaScript Injection
* Remote Code Execution
* Node.js API abuse
* Local file disclosure

---

## 3. Outils utilisés

* Navigateur web
* `curl`
* Requêtes HTTP concurrentes
* JavaScript
* Node.js
* API `os`
* API `fs`

---

# 4. Énumération de l'application

L'application **Around The World** est découverte sur le port `5555`.

L'application est basée sur :

```text
Node.js / Express
```

Le thème graphique et certains éléments de l'application font référence à **Daft Punk**.

L'analyse des fonctionnalités et des requêtes permet notamment d'identifier plusieurs endpoints :

```text
/api/create
/api/login
/api/givegold
/api/buypremium
/api/calculate
```

Les endpoints `/api/create` et `/api/login` permettent respectivement de créer un compte et de s'authentifier.

---

# 5. Création d'un compte

Un compte utilisateur est créé via l'API :

```text
POST /api/create
```

Puis une connexion est effectuée avec :

```text
POST /api/login
```

L'application attribue un solde de `gold` au compte.

Une fonctionnalité Premium est disponible, mais son coût est de :

```text
10000 gold
```

Le solde initial ne permet normalement pas d'acheter ce statut.

L'objectif devient donc de rechercher une faiblesse dans le système de transfert de `gold`.

---

# 6. Analyse du transfert de Gold

L'endpoint suivant est identifié :

```text
/api/givegold
```

Cette fonctionnalité permet de transférer une quantité de `gold` entre utilisateurs.

Lors de l'analyse du code HTML de l'application, un commentaire particulièrement intéressant est découvert :

```text
please fix the gold spamming issues
```

Ce commentaire suggère l'existence d'un problème lié à l'envoi répété de requêtes de transfert.

Le comportement de l'API est donc étudié sous l'angle d'une éventuelle **race condition**.

---

# 7. Exploitation de la race condition

Le problème vient du fait que la vérification du solde et la déduction du montant ne sont pas réalisées de manière atomique.

Le fonctionnement vulnérable peut être résumé ainsi :

```text
Check balance
     │
     ▼
Sufficient funds?
     │
     ▼
Add gold to recipient
     │
     ▼
Deduct gold from sender
```

Lorsque plusieurs requêtes sont envoyées simultanément, plusieurs requêtes peuvent effectuer leur vérification avant que la modification du solde soit correctement prise en compte.

Chaque requête concurrente peut donc créditer intégralement le destinataire sans que les déductions correspondantes soient correctement appliquées.

Cela permet de générer beaucoup plus de `gold` que le solde initial ne devrait le permettre.

---

# 8. Amplification du solde

Des requêtes concurrentes sont envoyées sur `/api/givegold`.

L'exploitation est effectuée en alternant les comptes émetteur et destinataire sur plusieurs rounds.

Les soldes obtenus évoluent notamment de la manière suivante :

```text
Round 1
10 gold
   │
   ▼
90 gold

Round 2
90 gold
   │
   ▼
2070 gold

Round 3
2070 gold
   │
   ▼
43470 gold
```

Le mécanisme permet donc d'obtenir un solde largement supérieur au montant initial.

Le seuil nécessaire pour accéder au Premium est :

```text
10000 gold
```

Le solde obtenu est suffisamment important pour acheter le statut Premium.

---

# 9. Achat du statut Premium

Une fois suffisamment de `gold` accumulé, l'endpoint :

```text
/api/buypremium
```

est utilisé.

Le compte obtient alors le statut Premium.

Cette fonctionnalité débloque notamment une calculatrice accessible via :

```text
/api/calculate
```

---

# 10. Analyse de la calculatrice Premium

La calculatrice accepte un paramètre nommé :

```text
calculation
```

Le comportement de l'application est étudié en envoyant différentes expressions JavaScript.

L'analyse révèle que la valeur fournie par l'utilisateur est évaluée directement côté serveur avec :

```javascript
eval()
```

L'utilisation de `eval()` sur une donnée contrôlée par l'utilisateur permet d'exécuter du JavaScript arbitraire dans le contexte du serveur Node.js.

Il s'agit donc d'une **Server-Side JavaScript Injection**, permettant une RCE.

---

# 11. Confirmation de l'exécution côté serveur

Afin de confirmer que le JavaScript est bien exécuté dans le processus Node.js, la propriété suivante est utilisée :

```javascript
process.version
```

L'application retourne la version de Node.js utilisée par le serveur.

Cela confirme que l'entrée `calculation` n'est pas simplement évaluée par un moteur de calcul côté client mais exécutée dans le contexte Node.js du serveur.

La chaîne d'exploitation devient alors :

```text
User input
    │
    ▼
calculation
    │
    ▼
eval()
    │
    ▼
Node.js
    │
    ▼
JavaScript arbitraire
```

---

# 12. Énumération du système via Node.js

Une première approche consistait à utiliser des fonctions permettant d'exécuter des commandes système.

Cependant, une tentative utilisant `child_process.execSync` a provoqué un crash du serveur.

Afin d'éviter de perturber davantage le service, l'énumération est poursuivie avec les modules Node.js intégrés `os` et `fs`.

Le module `os` permet notamment d'obtenir les informations sur l'utilisateur exécutant le processus.

La fonction utilisée est :

```javascript
os.userInfo()
```

Le résultat permet d'identifier l'utilisateur sous lequel l'application Node.js s'exécute :

```text
gm08
```

---

# 13. Recherche de `user.txt`

Une fois l'utilisateur identifié, le contenu de son répertoire personnel est énuméré avec le module `fs`.

La fonction utilisée est :

```javascript
fs.readdirSync('/home/gm08')
```

Le résultat révèle notamment la présence du fichier :

```text
user.txt
```

Le chemin cible est donc :

```text
/home/gm08/user.txt
```

---

# 14. Lecture du flag

La fonction Node.js suivante est utilisée pour lire le contenu du fichier :

```javascript
fs.readFileSync('/home/gm08/user.txt')
```

Le contenu retourné est :

```text
EPI{********}
```

Le flag utilisateur est donc :

```text
EPI{********}
```

---

# 15. Vulnérabilités et faiblesses identifiées

## 15.1 Race condition sur le transfert de Gold

L'endpoint `/api/givegold` ne protège pas correctement les opérations de vérification et de modification du solde.

La vérification du solde et la déduction ne sont pas atomiques.

**Impact :** manipulation du solde et génération artificielle de `gold`.

---

## 15.2 Utilisation dangereuse de `eval()`

L'endpoint Premium `/api/calculate` évalue directement une entrée contrôlée par l'utilisateur avec `eval()`.

**Impact :** exécution de JavaScript arbitraire côté serveur.

---

## 15.3 Accès aux API Node.js depuis le contexte d'exécution

L'exécution de JavaScript dans le contexte Node.js permet d'accéder aux fonctionnalités internes du runtime, notamment :

```text
os.userInfo()
fs.readdirSync()
fs.readFileSync()
```

Cela permet l'énumération du système et la lecture de fichiers accessibles au processus.

---

# 16. Chaîne d'exploitation complète

```text
Application Node.js / Express
          │
          ▼
     Port 5555
          │
          ▼
   Création de compte
          │
          ▼
   Analyse /api/givegold
          │
          ▼
   Race Condition
          │
          ▼
10 → 90 → 2070 → 43470 gold
          │
          ▼
   /api/buypremium
          │
          ▼
      Premium
          │
          ▼
   /api/calculate
          │
          ▼
       eval()
          │
          ▼
 JavaScript côté serveur
          │
          ▼
      os.userInfo()
          │
          ▼
        gm08
          │
          ▼
fs.readdirSync('/home/gm08')
          │
          ▼
      user.txt
          │
          ▼
fs.readFileSync('/home/gm08/user.txt')
          │
          ▼
EPI{********}
```

---

# 17. Résumé

L'exploitation commence par l'analyse de l'application **Around The World**, accessible sur le port `5555`.

Le statut Premium coûte `10000 gold`, mais une race condition est découverte dans l'endpoint `/api/givegold`. Le contrôle du solde et sa modification n'étant pas atomiques, l'envoi de requêtes concurrentes permet de multiplier artificiellement le nombre de `gold` disponibles.

Après avoir accumulé suffisamment de monnaie virtuelle, le statut Premium est acheté via `/api/buypremium`.

La fonctionnalité Premium `/api/calculate` utilise ensuite `eval()` sur le paramètre `calculation`. Cela permet d'exécuter du JavaScript arbitraire dans le contexte du serveur Node.js.

L'utilisateur du processus est identifié avec `os.userInfo()` comme étant `gm08`. L'énumération de `/home/gm08` avec `fs.readdirSync()` révèle ensuite `user.txt`.

Enfin, `fs.readFileSync()` permet de lire directement le fichier :

```text
/home/gm08/user.txt
```

Le flag obtenu est :

```text
EPI{********}
```

L'objectif du challenge étant la récupération de `user.txt`, l'exploitation est terminée à cette étape.

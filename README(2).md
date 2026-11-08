# Guide développeur — Créer une app qui utilise PoutAPI

Bienvenue ! Ce guide t'explique comment brancher ta propre app sur
l'écosystème PoutCoin (comptes, virements, données partagées) via
**PoutAPI**, l'API web commune à toutes les apps du hameau.

Tu n'as besoin de rien connaître du fonctionnement interne de l'API pour
suivre ce guide — juste savoir faire des requêtes HTTP dans ton langage de
prédilection.

---

## 1. Vue d'ensemble

PoutAPI est une API HTTP en ligne, hébergée à une adresse du type
`https://poutapi-server.onrender.com`. Toutes les apps du hameau (app
bancaire, chat, marketplace, etc.) parlent à cette même API, ce qui permet
par exemple à un utilisateur de transférer des PoutCoins d'une app à
l'autre de façon transparente.

**Deux niveaux d'identité à bien distinguer :**

1. **Ton app** — chaque app enregistrée a sa propre identité
   cryptographique (une paire de clés). Ça sert à identifier *quelle app*
   fait une requête, et permet à l'admin de révoquer une app précise en
   cas de problème.
2. **Le compte PoutCoin de l'utilisateur** — la personne qui utilise ton
   app se connecte avec son propre identifiant/mot de passe PoutCoin (le
   même que dans toutes les autres apps). C'est ce compte qui détermine ce
   que la personne a le droit de faire, jamais ton app.

Ces deux niveaux sont indépendants : ton app peut être open source sans
rien exposer de sensible, puisque les droits (notamment les droits admin)
suivent uniquement le compte connecté.

---

## 2. Étape 1 — Obtenir une clé pour ton app

Avant de coder quoi que ce soit, il te faut une identité pour ton app.

1. Récupère l'outil **"Demande de clé"** (fourni séparément par
   l'administrateur du hameau).
2. Lance-le, connecte-toi avec **ton propre compte PoutCoin** (pas besoin
   de droits particuliers).
3. Onglet **"Nouvelle demande"** : donne un nom et une description à ton
   app, valide. L'outil génère automatiquement une paire de clés sur ta
   machine — tu n'as rien à comprendre en cryptographie pour ça.
4. Préviens l'administrateur qu'une demande attend une validation.
5. Une fois approuvée : onglet **"Mes demandes"** → Actualiser →
   sélectionne ta demande → **"Finaliser"** → enregistre le fichier généré
   sous le nom `poutapi_secrets.json`, **à côté du code de ton app**.

Ce fichier contient tout ce qu'il te faut :

```json
{
  "app_id": "ton_app",
  "public_key": "...",
  "private_key": "...",
  "api_base_url": "https://poutapi-server.onrender.com"
}
```

### ⚠️ À savoir sur ce fichier

- **Ne le mets jamais dans un dépôt Git public** (ajoute-le à ton
  `.gitignore`). Ton code source, lui, peut être open source sans
  problème — c'est uniquement ce fichier qui doit rester privé.
- Si ton app est distribuée à plusieurs personnes (installateur, etc.), ce
  fichier partira avec chaque installation. C'est un choix de conception
  assumé : cette clé identifie ton app et permet de la révoquer en cas de
  souci, ce n'est **pas** un coffre-fort inviolable. La vraie protection
  des utilisateurs vient de leur mot de passe de compte, jamais de cette
  clé d'app. Voir section 7 pour le détail.
- Si la clé fuite ou que ton app est compromise, préviens l'admin : il
  peut la révoquer en un clic, et tu pourras en redemander une nouvelle.

---

## 3. Étape 2 — Comprendre l'authentification

Chaque requête vers l'API doit être **signée** avec la clé privée de ton
app, et inclure trois en-têtes HTTP :

| Header | Contenu |
|---|---|
| `X-App-Id` | l'identifiant de ton app (ex: `banque_app`) |
| `X-Timestamp` | l'heure actuelle, en secondes depuis epoch |
| `X-Signature` | la signature Ed25519 du message canonique (voir ci-dessous), encodée en base64url |

### Le message signé

```
{MÉTHODE}\n{CHEMIN}\n{TIMESTAMP}\n{CORPS_JSON_BRUT}
```

Exemple concret pour un appel `POST /login` avec le corps
`{"account_id":"alice","password":"secret"}` envoyé à l'instant
`1784100000` :

```
POST
/login
1784100000
{"account_id":"alice","password":"secret"}
```

(les `\n` sont de vrais retours à la ligne, pas les caractères `\` et `n`)

- `{MÉTHODE}` : en majuscules (`GET`, `POST`...)
- `{CHEMIN}` : le chemin de l'URL, sans le domaine, **mais avec la query
  string si ta requête en a une** (`/login`, pas `https://.../login` ; et
  `/me/logs?limit=200`, pas juste `/me/logs`, si tu appelles la route avec
  ce paramètre — sinon la signature ne correspondra pas à ce que le serveur
  recalcule de son côté, et tu auras un `0x0013` alors que ta clé est bonne)
- `{CORPS_JSON_BRUT}` : exactement la chaîne de caractères envoyée comme
  corps de la requête, telle quelle. Chaîne vide si pas de corps (GET).

Tu signes ce message avec ta clé privée (algorithme **Ed25519**), et tu
mets le résultat encodé en base64url dans `X-Signature`.

**Convention d'encodage** : partout dans le protocole (clé publique, clé
privée dans ton fichier de secrets, signature), le base64url est utilisé
**sans caractères de padding `=` final** — c'est ce que produisent la
plupart des libs par défaut si tu retires le padding toi-même (ex:
`base64.urlsafe_b64encode(...).rstrip(b"=")` en Python). Si ta lib ajoute
du padding par défaut, retire-le avant d'envoyer.

L'API vérifie cette signature avec la clé publique associée à ton
`X-App-Id`. Si ça ne correspond pas (mauvaise signature, corps modifié en
transit, timestamp trop vieux...), la requête est rejetée avant même
d'être traitée.

### La session utilisateur (deuxième niveau)

Une fois qu'un utilisateur se connecte via `POST /login`
(voir section 5), tu reçois un `session_token`. Pour toutes les requêtes
suivantes qui nécessitent d'être connecté, ajoute un quatrième header :

```
Authorization: Bearer <session_token>
```

Ce token expire après 1h d'inactivité (renouvelé à chaque requête), **et
au plus tard 24h après sa création**, même s'il est utilisé en continu :
passé ce délai, reconnecte simplement l'utilisateur. Un `POST /logout` le
révoque immédiatement.

Une session est aussi coupée (`0x0018`) dès que :
- le **mot de passe du compte est changé** (par l'utilisateur ou par un
  admin) — seule la session qui a fait le changement reste valide ;
- le compte est **suspendu ou bloqué** par un admin (une réactivation
  ultérieure ne ressuscite pas les anciennes sessions : il faut se
  reconnecter) ;
- le compte est **supprimé**.

Tant qu'un compte est suspendu, bloqué ou temporairement bloqué, **toute**
route à session répond `0x0004` ou `0x000B` (avec `remaining_seconds`, `-1`
= permanent), pas seulement `/login` et `/transfer`. Un compte **gelé**
(`0x0005`) garde la lecture (`/me`, `/me/logs`...) mais ne peut
plus rien modifier : `/transfer`, `/me/password` et `/apps/request`
répondent `0x0005`. Ton app doit donc savoir
afficher ces codes sur n'importe quel appel, pas seulement à la connexion.
Les changements d'état faits par l'API sont immédiats ; ceux faits
directement dans la base (outil de gestion) sont pris en compte sous une
minute.

---

## 4. Exemple de code complet (Python)

Le plus simple : récupère le fichier `poutapi_client.py` (fourni avec
l'outil "Demande de clé" ou l'app bancaire) et pose-le à côté de ton code.
Il gère toute la signature pour toi :

```python
from poutapi_client import PoutAPIClient
import json

# Charge ton identité (générée à l'étape 2)
with open("poutapi_secrets.json") as f:
    identity = json.load(f)

client = PoutAPIClient(
    base_url=identity["api_base_url"],
    app_id=identity["app_id"],
    private_key_b64url=identity["private_key"],
    public_key_b64url=identity["public_key"],
)

# Connexion d'un utilisateur
resp = client.login("alice", "motdepasse_alice")
if not resp.ok:
    print("Erreur de connexion :", resp.code, resp.message)
else:
    print("Connecté ! Solde et infos :", client.request("GET", "/me").data)

    # Faire un virement
    resp = client.transfer("bob", 50)
    print("Virement :", resp.code, resp.data)

    # Voir son propre historique
    resp = client.request("GET", "/me/logs")
    for log in resp.data.get("logs", []):
        print(log["timestamp"], log["event"], log["content"])
```

### Si tu codes dans un autre langage

Le protocole ne dépend d'aucune librairie particulière — n'importe quel
langage qui sait faire du HTTP et de la signature Ed25519 fonctionne. Il
te faut :
- Une lib de crypto supportant Ed25519 (quasiment tous les langages
  modernes en ont une : `cryptography` en Python, `crypto` intégré en
  Node.js, `libsodium` en C/C++/beaucoup d'autres, etc.)
- Construire le message canonique exactement comme décrit section 3
- L'encoder en base64url (variante base64 sans `+`/`/`/padding `=`)

---

## 5. Référence des routes

Toutes les routes ci-dessous (sauf `/health`) exigent les 3 headers
d'authentification d'app. Celles marquées **session** exigent en plus le
header `Authorization`.

| Méthode | Route | Session ? | Description |
|---|---|---|---|
| GET | `/health` | non | Vérifie que l'API répond |
| POST | `/signup` | non | `{account_id, password}` — crée un compte (solde 0) → `session_token` |
| POST | `/login` | non | `{account_id, password}` → `session_token` |
| POST | `/logout` | oui | Révoque la session en cours |
| GET | `/me` | oui | Infos du compte connecté (solde, statut) |
| GET | `/me/logs` | oui | Historique personnel (`?limit=100`) |
| POST | `/me/password` | oui | `{old_password, new_password}` — change le mot de passe du compte connecté |
| POST | `/transfer` | oui | `{to, amount}` — virement depuis le compte connecté |
| POST | `/apps/request` | oui | Soumet une demande de clé pour une nouvelle app |
| GET | `/apps/request/status` | oui | Statut de tes demandes en cours |

Les routes `/admin/*` existent mais sont réservées aux comptes ayant des
droits root — inutile de les appeler depuis une app classique, elles
renverront systématiquement `0x0008` sinon.

### À propos de `/signup`

Permet à n'importe qui de créer un compte PoutCoin depuis ton app, sans
validation par l'administrateur. Comme `/login`, elle ne demande pas de
session (juste les headers d'app habituels), et prend le même corps :
`{account_id, password}`.

Règles à respecter (ou tu recevras `0x0010`) :
- `account_id` : de 3 à 20 caractères, uniquement des lettres majuscules,
  minuscules et le tiret du bas `_`. Pas de chiffres, pas d'accents, pas
  d'espaces ni d'autres caractères spéciaux.
- `password` : 8 caractères minimum.

Ce que tu dois savoir pour construire ton écran d'inscription :
- Le compte est créé **avec un solde de 0 PoutCoin**, toujours.
- Si le nom est déjà pris (en ignorant la casse : `Alice` et `alice` sont
  considérés comme le même nom), tu reçois `0x000E`. C'est aussi le cas
  pour un nom réservé à l'administration du hameau.
- Une seule inscription réussie est possible **par adresse IP et par
  tranche de 24h**. Une deuxième tentative dans ce délai renvoie `0x0006`
  avec `remaining_seconds` (le temps avant de pouvoir réessayer). Une
  tentative qui échoue (nom déjà pris, format invalide) ne consomme **pas**
  ce quota (en revanche, des tentatives trop rapprochées depuis la même
  IP — 5 d'affilée, puis environ une par minute — renvoient aussi `0x0006`,
  même si elles échouent). Pense à le dire à tes utilisateurs dans le message d'erreur :
  deux personnes qui partagent la même connexion internet (même maison,
  même wifi) devront s'inscrire à un jour d'intervalle.
- En cas de succès, tu reçois directement un `session_token` (comme après un
  `/login`) : l'utilisateur est déjà connecté, inutile d'enchaîner avec un
  `/login`.

### À propos de `/me/password`

Nécessite l'**ancien** mot de passe (jamais de changement sans le prouver,
même via une app "de confiance") et un nouveau d'au moins 8 caractères. Ne
touche à rien côté app : aucun header supplémentaire, juste la session
habituelle. Si tu construis une UI pour les utilisateurs de ton app (pas
uniquement une app "vitrine" en lecture seule), pense à leur proposer cet
écran plutôt que de les renvoyer vers une autre app pour ça — c'est prévu
pour être appelé depuis n'importe quelle app, pas seulement l'app bancaire
officielle.

Comme `/login`, cette route est protégée contre le bruteforce de l'ancien
mot de passe : au bout de plusieurs échecs, elle renvoie temporairement
`0x000B` (avec `remaining_seconds`) même si tu retentes ensuite avec le bon
mot de passe. Gère ce cas dans ton UI comme tu le ferais pour un `0x000B`
reçu sur `/login`.

### Format des réponses

Chaque réponse est un JSON `{ code, message, ...champs }`, mais le
**statut HTTP varie aussi selon le code** (200 pour un succès, 401/403 pour
les problèmes d'auth, 404 pour "introuvable", 429 pour le rate limit, 500
pour une erreur serveur, etc.) — ne te fie pas uniquement au JSON, et si ta
librairie HTTP lève une exception sur les statuts non-2xx, configure-la
pour quand même lire le corps de la réponse plutôt que d'ignorer l'erreur :
le `code` dans le JSON reste la source de vérité pour distinguer les cas
(ex: `0x000B` compte bloqué vs `0x0001` mauvais mot de passe peuvent tous
les deux arriver en 401/403 selon le cas).

Champs additionnels renvoyés par route en cas de succès (`0x0000`) :

| Route | Champs renvoyés |
|---|---|
| `POST /signup` | `session_token`, `expires_at` (epoch ms) — même forme que `/login` |
| `POST /login` | `session_token`, `expires_at` (epoch ms) |
| `GET /me` | `account_id`, `balance`, `status` (`active`/`suspended`/`frozen`/`blocked`) |
| `GET /me/logs` | `logs`: tableau de `{ timestamp, event, content }`, plus récent en premier |
| `POST /me/password` | (aucun champ additionnel) |
| `POST /transfer` | `new_balance` (solde de l'émetteur après l'opération) |
| `POST /apps/request` | `request_id` |
| `GET /apps/request/status` | `requests`: tableau des demandes en cours |

`event` dans `/me/logs` est une chaîne libre (ex: `LOGIN_SUCCESS`,
`TRANSFER_SUCCESS`, `TRANSFER_FAILED`, `ADMIN_SET_STATUS`...) et `content`
un objet dont la forme dépend de l'événement — inspecte-le au cas par cas
plutôt que de supposer une structure fixe, de nouveaux types d'événements
peuvent apparaître avec le temps.

---

## 6. Codes d'erreur

Chaque réponse contient `{ "code": "0x...", "message": "..." }`.

| Code | Signification |
|---|---|
| `0x0000` | Succès |
| `0x0001` | Identifiants incorrects |
| `0x0002` | Compte introuvable |
| `0x0003` | Solde insuffisant |
| `0x0004` | Compte suspendu |
| `0x0005` | Compte gelé (lecture seule) |
| `0x0006` | Trop de requêtes (rate limit) — regarde `remaining_seconds` dans la réponse |
| `0x0007` | Commande/route invalide |
| `0x0008` | Action réservée aux comptes root |
| `0x000A` | Corps de requête trop volumineux (plus de 256 Ko) |
| `0x000B` | Compte bloqué — `remaining_seconds` indique le temps restant (`-1` = permanent) |
| `0x000E` | Nom de compte déjà pris (`/signup`) — la comparaison ignore la casse |
| `0x0010` | Arguments manquants ou invalides |
| `0x0013` | Signature d'app invalide |
| `0x0014` | App inconnue ou révoquée — contacte l'admin |
| `0x0016` | Timestamp hors fenêtre (vérifie l'horloge de ta machine) |
| `0x0017` | Session manquante |
| `0x0018` | Session invalide ou expirée — reconnecte l'utilisateur |
| `0x0019` | Une demande du même nom est déjà en attente |
| `0x001A` | Requête déjà utilisée (rejeu détecté) — signe une nouvelle requête, ne renvoie jamais deux fois le même triplé timestamp+signature |

---

## 7. Bonnes pratiques et limites à connaître

- **Ne fais jamais confiance aux données du client dans ton propre code
  serveur si tu en as un** (ex: ne recalcule pas un solde toi-même, relis
  toujours `/me` après une opération — c'est ce que fait déjà
  `poutapi_client.py`).
- **Rate limiting** : l'API limite le débit à plusieurs niveaux. Dès que tu
  reçois `0x0006` (HTTP 429), lis `remaining_seconds` (ou l'en-tête standard
  `Retry-After`) et attends ce délai avant de réessayer — ne boucle jamais
  dessus immédiatement. Limites actuelles (volontairement larges, elles
  peuvent être ajustées) :
  - par adresse IP : environ 300 requêtes en rafale, puis 10 par seconde.
    Plusieurs personnes derrière la même box partagent ce compteur ;
  - par session : environ 80 en rafale, puis 3 par seconde ;
  - `/login` : 20 tentatives rapprochées par IP, puis 1 toutes les 3 s.
    Après 5 échecs sur un même compte depuis une même IP : 1 essai toutes
    les 30 s. Cela ne bloque **pas** le compte lui-même (le vrai propriétaire
    peut toujours se connecter depuis une autre IP). Échouer sur plus de 15
    comptes différents en 10 minutes bloque l'IP 10 minutes. Un login limité
    renvoie `0x0006` (et non plus `0x000B`, réservé à un compte réellement
    bloqué par un admin) ;
  - `/signup` : 5 tentatives, puis environ 1 par minute, par IP (en plus du
    quota d'une inscription réussie par jour) ;
  - `/transfer` : 20 en rafale, puis 1 toutes les 3 s, par compte ;
  - `/apps/request` : 3 demandes, puis 1 toutes les 10 min ; 5 demandes en
    attente au maximum.

  Autres plafonds : corps de requête de 256 Ko,
  identifiants de 128 caractères, mots de passe de 1024 caractères.
- **Le rôle suit le compte, jamais ton app.** N'essaie pas de coder de la
  logique "si l'utilisateur est admin" dans ton app — demande simplement
  l'action à l'API (ex: une route `/admin/*`), elle refusera proprement
  si le compte connecté n'a pas les droits, avec `0x0008`.
- **Anti-rejeu** : en plus de la fenêtre de tolérance de timestamp
  (~2 minutes), l'API refuse désormais toute signature déjà vue (code
  `0x001A`). Une requête interceptée telle quelle (logs, proxy...) ne peut
  donc plus être rejouée, même dans la fenêtre de tolérance. Ça ne dispense
  pas pour autant de bonnes pratiques basiques : évite de logger tes
  requêtes signées en clair quelque part d'accessible, et signe toujours
  une requête juste avant de l'envoyer (ne réutilise jamais un couple
  timestamp+signature déjà généré, même par erreur de code côté client —
  ta prochaine tentative doit resigner depuis zéro).
- **Mots de passe** : envoie-les toujours en clair dans le corps JSON
  (comme avant, la sécurité vient du TLS + de la signature de la requête,
  pas d'un hachage côté client) — l'API se charge elle-même de les stocker
  hachés. Tu n'as rien à faire de spécial ici, c'est juste bon à savoir si
  quelqu'un te pose la question.
- Un utilisateur peut voir dans **son propre historique** (`/me/logs`)
  les actions qui le concernent, y compris si un admin modifie son
  compte — pense à formuler tes messages d'erreur/succès en gardant ça en
  tête (transparence assumée du système).

---

## 8. Besoin d'aide ?

En cas de blocage (erreur inattendue, doute sur le protocole, demande de
clé qui traîne), contacte l'administrateur du hameau — lui seul a accès
direct à la base de données et peut diagnostiquer les problèmes qui ne
sont pas visibles depuis l'API elle-même.

---

ATTENTION (à lire avant de modifier ce fichier, que tu sois une IA ou une personne) :
ce README est versionné. À CHAQUE modification, aussi petite soit-elle, incrémente le
numéro de version ci-dessous (V4 devient V5, puis V6...) dans le même commit, et laisse
cette consigne ainsi que la ligne de version TOUT EN BAS du fichier.

Version : V4

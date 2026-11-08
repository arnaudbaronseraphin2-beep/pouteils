# Guide développeur — Créer une app avec PoutSuite

PoutSuite te donne, sans que tu aies à gérer de mots de passe ni de base de données :

- **PoutAuth** : « Se connecter avec PoutCoin » (SSO).
- **PoutPay** : abonnements hebdomadaires, salaires hebdomadaires et achats uniques en PTC.
- **PoutCloud** : un petit stockage JSON par utilisateur.

Adresse du service : `https://poutsuite.onrender.com`
(hébergement gratuit : le premier appel après une période d'inactivité peut prendre 30 à 60 secondes).

## 1. Principe en deux phrases

L'utilisateur tape son mot de passe PoutCoin **uniquement sur une page PoutSuite**, jamais dans ton app. Ton app reçoit ensuite un code, qu'elle échange (côté serveur) contre un token, et c'est avec ce token qu'elle agit au nom de l'utilisateur.

## 2. Obtenir un `client_id` et un `client_secret`

Demande à l'administrateur du hameau de t'enregistrer. Donne-lui :

| Information | Exemple |
|---|---|
| Nom de l'app | `MonApp` |
| URL(s) de retour (`redirect_uris`) | `https://monapp.example/cb` (exactes, caractère pour caractère) |
| Compte qui encaisse les paiements (`payee_account`) | `ton_compte` |
| URL de webhook, facultative | `https://monapp.example/webhook` |

Tu reçois un `client_id`, un `client_secret` et un `webhook_secret`. Le secret n'est montré **qu'une seule fois**.

**Règles du `client_secret` :**
- il ne vit que sur ton **serveur**, dans des variables d'environnement ;
- jamais dans une app Android/iOS, dans du JavaScript de navigateur, ni dans un dépôt Git public ;
- une app mobile passe par un petit backend à toi qui parle à PoutSuite ;
- s'il fuite, préviens l'administrateur pour en obtenir un nouveau.

## 3. Le SDK Python

Prends `sdk/poutsuite_sdk.py` dans le dépôt PoutSuite, pose-le à côté de ton code, puis `pip install httpx`.

```python
from poutsuite_sdk import PoutSuite
ps = PoutSuite("https://poutsuite.onrender.com", CLIENT_ID, CLIENT_SECRET)

url  = ps.login_url("https://monapp.example/cb", state)   # 1. rediriger l'utilisateur vers cette URL
user = ps.exchange_code(code, "https://monapp.example/cb") # 2. au retour : user["account_id"], user["access_token"]
ps.is_subscribed(user["account_id"])                       # 3. abonnement à jour ? (True/False)
```

Un exemple complet en Flask est fourni plus bas. Dans un autre langage, tout se fait avec de simples requêtes HTTP (section 8).

## 4. PoutAuth — connexion

1. Tu redirige l'utilisateur vers `login_url(redirect_uri, state, scope)`.
2. Il se connecte sur la page PoutSuite et est renvoyé vers ta `redirect_uri` avec `?code=...&state=...`.
3. **Vérifie que `state` est identique à celui que tu avais généré** (protection contre les attaques CSRF), puis appelle `exchange_code`.
4. Tu obtiens `account_id` (qui est l'utilisateur), `access_token` (valable 1 h) et `scope`.

Le code ne vit que 2 minutes et ne sert qu'une seule fois : échange-le tout de suite.

**Scopes** (sépare-les par des espaces, ne demande que le nécessaire) :

| Scope | Donne accès à |
|---|---|
| `profile` | l'identifiant du compte |
| `pay` | la liste et la révocation de ses propres mandats |
| `cloud` | son espace de stockage PoutCloud |

L'identité de l'utilisateur (`account_id`) est fiable : c'est PoutSuite qui l'a vérifiée. Ne la demande jamais à l'utilisateur dans un formulaire de ton app.

## 5. PoutPay — paiements

Un **mandat** décrit un paiement : combien, vers qui, à quel rythme.

| `kind` | Effet |
|---|---|
| `subscription` | abonnement : prélevé chaque semaine vers `payee` (par défaut ton compte encaisseur) |
| `salary` | virement hebdomadaire de l'utilisateur qui valide vers `payee` |
| `once` | achat unique, débité immédiatement |

### Créer et faire valider un mandat

```python
m = ps.propose(amount=10, label="Premium", kind="subscription")
# m["mandate_id"] à conserver, m["consent_url"] : y rediriger l'utilisateur
```

Paramètres facultatifs : `payee="autre_compte"`, `start_in_days=7` (premier prélèvement différé ; sans cela, le premier prélèvement a lieu à la validation).

L'utilisateur arrive sur une page PoutSuite qui affiche le montant, le bénéficiaire et le rythme, se connecte et autorise. Si le premier paiement échoue (solde insuffisant), le mandat n'est pas activé.

### Savoir si un utilisateur est abonné

```python
ps.is_subscribed(account_id)   # True si le dernier prélèvement date de moins de 7 jours + 2 jours de grâce
ps.mandate(mandate_id)         # détail : status, next_run, last_paid, fails
ps.cancel(mandate_id)          # annuler côté app
```

### Statuts d'un mandat

| `status` | Signification |
|---|---|
| `pending` | créé, pas encore validé par l'utilisateur |
| `active` | en cours, prélèvements automatiques |
| `completed` | achat unique terminé |
| `suspended` | 3 échecs de suite : plus de prélèvement, à retraiter par l'administrateur |
| `needs_reauth` | l'utilisateur a changé son mot de passe : il doit revalider le mandat |
| `needs_review` | prélèvement interrompu : l'administrateur doit vérifier (aucun double débit automatique) |
| `failed` / `cancelled` | échec définitif / annulé |

En cas de solde insuffisant, PoutSuite réessaie 24 h plus tard, deux fois, puis suspend le mandat.

### Webhooks (facultatif mais conseillé)

Si tu as une URL de webhook, tu reçois un `POST` JSON à chaque événement : `mandate_activated`, `payment_succeeded`, `payment_failed`, `subscription_suspended`, `needs_reauth`, `mandate_failed`. **Vérifie toujours la signature** avant de faire confiance au message :

```python
from poutsuite_sdk import PoutSuite
ok = PoutSuite.verify_webhook(WEBHOOK_SECRET, request.get_data(), request.headers["X-PoutSuite-Signature"])
```

Ne te base pas uniquement sur les webhooks : ils peuvent se perdre, `is_subscribed` reste la source de vérité.

## 6. PoutCloud — stockage

Un espace par couple (ton app, utilisateur). Nécessite le scope `cloud` et le token de l'utilisateur.

```python
ps.cloud_put(token, "preferences", {"theme": "sombre"})
ps.cloud_get(token, "preferences")      # -> {"theme": "sombre"}
```

Limite : 1 Mo par valeur. Les données sont isolées : une autre app ne peut pas lire les tiennes. Le token expire au bout d'1 h : à l'expiration, renvoie l'utilisateur vers la connexion (elle est instantanée s'il est déjà identifié sur PoutSuite).

## 7. Exemple complet (Flask)

```python
import os, secrets
from flask import Flask, redirect, request, session
from poutsuite_sdk import PoutSuite

app = Flask(__name__)
app.secret_key = os.environ["FLASK_SECRET"]
ps = PoutSuite("https://poutsuite.onrender.com", os.environ["PS_ID"], os.environ["PS_SECRET"])
CB = "https://monapp.example/cb"

@app.route("/")
def home():
    if "user" not in session:
        session["state"] = secrets.token_urlsafe(16)
        return redirect(ps.login_url(CB, session["state"], "profile pay cloud"))
    return f"Salut {session['user']} ! Abonné : {ps.is_subscribed(session['user'])} <a href='/abo'>S'abonner</a>"

@app.route("/cb")
def cb():
    if request.args.get("state") != session.get("state"):
        return "state invalide", 400
    u = ps.exchange_code(request.args["code"], CB)
    session["user"], session["token"] = u["account_id"], u["access_token"]
    return redirect("/")

@app.route("/abo")
def abo():
    return redirect(ps.propose(10, "Premium")["consent_url"])
```

## 8. Sans le SDK (HTTP brut)

| Action | Requête |
|---|---|
| Connexion | naviguer vers `GET /oauth/authorize?client_id=&redirect_uri=&state=&scope=` |
| Échange du code | `POST /oauth/token` (formulaire : `grant_type=authorization_code`, `code`, `client_id`, `client_secret`, `redirect_uri`) |
| Identité d'un token | `GET /oauth/userinfo` avec `Authorization: Bearer <token>` |
| Révoquer un token | `POST /oauth/revoke` avec `Authorization: Bearer <token>` |
| Créer un mandat | `POST /pay/mandates` (JSON `kind`, `amount`, `label`, `payee`, `start_in_days`) |
| Détail / annuler | `GET /pay/mandates/{id}`, `POST /pay/mandates/{id}/cancel` |
| Abonné ? | `GET /pay/check/{account_id}` |
| Stockage | `GET /cloud`, `GET/PUT/DELETE /cloud/{clé}` avec `Authorization: Bearer <token>` (corps du PUT : `{"value": ...}`) |

Les routes `/pay/mandates*` et `/pay/check/*` s'authentifient avec les en-têtes `X-Client-Id` et `X-Client-Secret`.

## 9. Codes d'erreur

Chaque réponse d'erreur contient `{ "code": "0x...", "message": "..." }`.

| Code | Signification |
|---|---|
| `0x0000` | Succès |
| `0x0008` | Droit manquant (scope absent ou action réservée à l'administrateur) |
| `0x0009` | Introuvable (mandat, clé de stockage) |
| `0x000A` | Valeur trop volumineuse |
| `0x0010` | Paramètres invalides (par exemple `redirect_uri` non enregistrée) |
| `0x0014` | `client_id` ou `client_secret` invalide |
| `0x0017` | Token manquant |
| `0x0018` | Token ou code invalide / expiré : reconnecte l'utilisateur |

Pendant une connexion ou un paiement, l'utilisateur peut aussi voir des codes venant de PoutCoin (`0x0001` identifiants incorrects, `0x0003` solde insuffisant, `0x000B` compte bloqué, etc.) : la page les affiche elle-même, tu n'as rien à gérer.

## 10. Bonnes pratiques et limites

- **Ne demande jamais le mot de passe PoutCoin** dans ton app, même « pour simplifier ». Si ton app a un champ mot de passe PoutCoin, c'est qu'elle est mal conçue.
- **Vérifie toujours `state`** au retour de la connexion.
- **Ne te fie pas à un `account_id` envoyé par ton frontend** : ne l'accepte que s'il vient d'un échange de code que ton serveur a fait lui-même.
- **Ne donne un accès payant qu'après `is_subscribed`**, vérifié côté serveur à chaque session.
- **Les tokens ne survivent pas à un redémarrage de PoutSuite** : prévois de renvoyer l'utilisateur vers la connexion en cas de `0x0018`.
- **Un mandat d'abonnement prélève toutes les semaines, jusqu'à annulation** : affiche clairement le montant et le rythme avant de rediriger vers la page de validation. L'utilisateur peut aussi révoquer lui-même (`POST /pay/me/mandates/{id}/revoke`, scope `pay`).
- Les paiements passent par PoutAPI : un compte gelé, suspendu ou bloqué ne peut pas payer.

## 11. Besoin d'aide ?

Contacte l'administrateur du hameau. Les journaux de PoutSuite lui permettent de voir ce qui s'est passé pour un mandat ou une connexion précise.

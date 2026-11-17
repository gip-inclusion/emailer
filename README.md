# emailer

Mise en forme et envoi des campagnes e-mail de La Plateforme de l'inclusion via l'**API transactionnelle Brevo**.

L'outil sépare deux choses :

- **le rendu** : un module par type d'e-mail transforme des données JSON en fichiers HTML prêts à envoyer ;
- **l'envoi**, commun à tous les types : `emailer send` / `schedule` envoie chaque fichier HTML d'un dossier à ses destinataires, une copie individuelle par personne.

## Principe : un fichier HTML = un e-mail

Quel que soit le type d'e-mail, l'envoi lit les `*.html` d'un dossier. Chaque fichier porte lui-même son objet et ses destinataires :

```html
<html>
<head><title>Objet de l'e-mail</title></head>
<body>
<!-- to: premiere.personne@example.org -->
<!-- to: deuxieme.personne@example.org -->
…
</body>
</html>
```

- `<title>` devient l'**objet** (préfixé `[TEST]` en mode test).
- Chaque `<!-- to: … -->` reçoit **sa propre copie** (pas de vrai « cc », les adresses ne se voient pas entre elles ; doublons retirés).
- `{{ params.EMAIL }}` dans le HTML est remplacé par l'adresse du destinataire de la copie (utile pour un lien de formulaire d'avis).

Un e-mail produit ailleurs (HTML déjà mis en forme en amont) peut donc être envoyé tel quel, à condition d'y ajouter `<title>` et les `<!-- to: -->`.

## Types d'e-mails disponibles

| Type | Destinataires | Rendu | Entrée |
|---|---|---|---|
| Recommandations SPS (solutions de parcours) | conseillers France Travail | `emailer render` (`emailer/render.py` + `template.py`) | 1 JSON par conseiller, schéma `docs/schema/email.schema.json` |
| Fins de contrats IAE — vue prescripteur | conseillers prescripteurs | `python -m emailer.render_prescripteurs` | liste JSON de conseillers (voir l'en-tête du module) |
| Fins de contrats IAE — vue SIAE | structures IAE employeuses | `python -m emailer.render_siae` ou `emailer.render_structure` | liste JSON de structures (voir l'en-tête du module) |

`emailer/charte.py` regroupe ce qui est commun à tous les rendus (couleurs, logo, échappement, encart avis, tableaux qui s'empilent sur mobile).

**Ajouter un type d'e-mail** : créer `emailer/render_<type>.py` qui importe `charte.py` et écrit un HTML par e-mail avec `<title>` et `<!-- to: -->`. L'envoi n'a rien à changer.

## 1. Installation (une fois)

```bash
uv sync
cp .env.example .env      # puis renseigner les variables ci-dessous
```

| Variable | Requise | Pour quoi |
|---|---|---|
| `BREVO_API_KEY` | oui (envoi) | clé API Brevo ; idéalement **une clé par personne**, révocable seule |
| `BREVO_SENDER` | oui (envoi) | adresse d'expéditeur, **validée** dans Brevo |
| `TEST_RECIPIENTS` | oui pour `--test` | vos adresses de test, séparées par des virgules |
| `EMAILER_TUNNEL_HOST` / `EMAILER_TUNNEL_USER` | oui (envoi) | box du tunnel SOCKS (voir ci-dessous) ; valeurs transmises hors Git |
| `BREVO_PROXY` | optionnel | proxy SOCKS (sinon `--via`) ; le tunnel doit être ouvert |

Le rendu n'a besoin d'aucune variable. Elles ne servent qu'à `send`, `schedule` et `cancel`.

### Tunnel d'envoi (adresse IP fixe)

Brevo n'accepte les appels API que depuis des adresses IP autorisées. Les appels passent donc par une box à IP fixe, via un tunnel SSH (le contenu reste chiffré de bout en bout : la box ne voit ni la clé ni les e-mails).

Pour un nouvel utilisateur : générer une clé SSH et la faire ajouter sur la box par quelqu'un qui l'administre.

```bash
ssh-keygen -t ed25519 -f ~/.ssh/sps-tunnel
```

Puis, avant chaque envoi :

```bash
set -a; . ./.env; set +a
ssh -i ~/.ssh/sps-tunnel -D 1080 -N -f "$EMAILER_TUNNEL_USER@$EMAILER_TUNNEL_HOST"
export BREVO_PROXY=socks5h://127.0.0.1:1080
```

Utiliser `socks5h` (et non `socks5`) : le tunnel n'autorise que `api.brevo.com`.

## 2. Produire les HTML

Selon le type d'e-mail (voir le tableau ci-dessus). Exemple pour les recommandations SPS :

```bash
uv run emailer deanon out/json/ --csv <le-vrai.csv> -o out/json-nom/   # optionnel : vrais noms, sans LLM
uv run emailer render out/json-nom/ -o out/html/                       # ou out/json/ pour un rendu anonyme
```

Ouvrir quelques fichiers dans un navigateur pour relire avant d'envoyer.

## 3. Envoyer

**Test** — chaque fichier part vers `TEST_RECIPIENTS`. À lancer sur un **petit dossier d'échantillon**, pas sur toute la campagne (1 envoi par fichier et par adresse de test) :

```bash
uv run emailer send out/echantillon/ --test
```

**Envoi immédiat** aux vrais destinataires :

```bash
uv run emailer send out/html/ --tag ma-campagne-2026-11
```

**Envoi programmé** (recommandé pour une campagne : on peut l'annuler jusqu'à l'heure prévue) :

```bash
uv run emailer schedule out/html/ --at 2026-11-24T07:00:00.000+01:00 --tag ma-campagne-2026-11
```

- Le fuseau compte : `+01:00` en hiver, `+02:00` en été. Brevo accepte environ 72 h d'avance.
- La commande affiche un **`runId`**, aussi noté dans `out/schedules.jsonl` (sans aucune adresse). Ce fichier reste sur la machine de la personne qui a programmé : c'est elle qui peut annuler.
- `--tag` (répétable) permet ensuite de filtrer les statistiques de la campagne dans Brevo.

**Annuler** un envoi programmé (tunnel ouvert) :

```bash
uv run emailer cancel <runId>
```

**Adresses bloquées** : avant chaque envoi réel, `send` et `schedule` retirent les adresses de la blocklist Brevo (adresses invalides, désinscriptions, plaintes spam). Si la blocklist est injoignable, l'envoi continue sans ce filtre. Pour le désactiver : `--no-blocklist-filter`.

## 4. Après l'envoi

Les statistiques par campagne sont dans Brevo (filtre par tag), ou via l'API `GET /v3/smtp/statistics/aggregatedReport?tag=…` (tunnel ouvert).

Purger les sorties nominatives :

```bash
uv run emailer purge                      # out/json-nom + out/html de plus de 7 jours (demande confirmation)
uv run emailer purge --older-than 0 -y    # tout, sans confirmation
```

## Données personnelles

- `data/` et `out/` ne sont **pas versionnés** : y déposer les données sources et les sorties. Les transmettre par un canal sûr, jamais par le dépôt.
- `.env` contient les secrets : ne pas le partager, chacun a le sien.
- Avec un assistant IA (Claude Code…) : les consignes du dépôt (`CLAUDE.md`) lui interdisent de lire les fichiers nominatifs ; les scripts sont déterministes et n'affichent que des compteurs.

> `emailer convert <md> -o <dir>` est un ancien adaptateur Markdown → JSON pour les recommandations SPS, conservé mais hors du chemin habituel.

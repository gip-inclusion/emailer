# emailer

Outil de **mise en forme HTML et d'envoi via l'API Brevo** des campagnes e-mail de La Plateforme de l'inclusion. Il sert à **toutes sortes d'e-mails** : chaque type a son module de rendu ; l'envoi (`emailer send` / `schedule` / `cancel`) est commun et ne dépend que des fichiers HTML. Mode d'emploi pour les humains : `README.md`.

Contrat d'envoi : un `*.html` = un e-mail ; `<title>` = objet ; chaque `<!-- to: … -->` reçoit une **copie individuelle** ; `{{ params.EMAIL }}` = adresse de la copie. Un HTML pré-rendu en amont s'envoie tel quel une fois `<title>` + `<!-- to: -->` injectés.

## Types d'e-mails (contexte métier)

- **Recommandations SPS** (Solutions de Parcours Structurées) → **conseillers France Travail**. Chaque e-mail liste, **par bénéficiaire** (demandeur d'emploi convoqué cette semaine), des **solutions de parcours** : structures IAE (les-emplois), formations, dispositifs (EPIDE, PLIE, SEVE), services data·inclusion par contrainte (mobilité, santé, famille, logement, numérique…). Recommandations auto-générées, à valider en entretien. Flux par défaut **anonyme** (`Bénéficiaire #N`, `DE #x`) ; vrais noms injectés seulement par `deanon`.
- **Fins de contrats IAE — vue prescripteur** → conseillers prescripteurs ; **vue SIAE** → structures IAE employeuses. Campagnes mensuelles (sept., oct. 2026…), souvent livrées en amont sous forme de **JSON pré-rendu** (`{objet, html, destinataires[]…}`).

IAE = insertion par l'activité économique.

## Contrainte cardinale (données personnelles)

Les données nominatives (noms, adresses, diagnostics, contrats) n'existent que dans les **entrées non versionnées** (`data/`, CSV réel — p. ex. `data/Lille.csv`, JSON fournis par lot) et les **sorties** sous `out/`. Rendu, `deanon` et envoi sont du Python déterministe : **aucun LLM ne lit de données nominatives**. Un agent peut exécuter les scripts (stdout = compteurs only) mais ne doit jamais `Read` les sorties nominatives : `out/json-nom/**`, `out/html/**`, ni les HTML/JSON de campagne (`out/Prescripteurs-fins-contrats/**`, `out/SIAE-fins-contrats/**`, etc.).

## Fichiers

Package `emailer/` (CLI `uv run emailer …`) :
- **Commun** — `emailer/brevo.py` : envoi / envoi programmé / annulation (**générique** : lit les `*.html`, une copie par `<!-- to: -->`, substitue `{{ params.EMAIL }}`, pré-filtre blocklist). `emailer/charte.py` : **charte + helpers partagés** (couleurs, logo, URL formulaire Tally, `esc`/`fr`/`mailto`, encart avis, CSS tableau responsive). `emailer/purge.py` : purge des sorties nominatives.
- **Type SPS** — `emailer/deanon.py` (JSON anonyme + CSV réel → JSON nominatif `out/json-nom/`, **substitution déterministe par clé `de_id`, aucun LLM**) ; `emailer/render.py` + `emailer/template.py` (JSON → **un HTML par conseiller**, `out/html/`) ; `emailer/schema.py` + `docs/schema/email.schema.json` (contrat JSON donné à l'agent amont) ; `emailer/convert.py` (**adaptateur legacy** MD→JSON, hors chemin nominal).
- **Type fins de contrats** — `emailer/render_prescripteurs.py` (vue prescripteur, `out/Prescripteurs-fins-contrats/`) ; `emailer/render_siae.py` (vue SIAE groupée par prescripteur) et `emailer/render_structure.py` (vue structure IAE, cartes-tableaux par segment, `out/SIAE-fins-contrats/`). Lancer : `uv run python -m emailer.render_<type> …` (voir l'en-tête de chaque module).
- **Nouveau type d'e-mail** = un `emailer/render_<type>.py` qui importe `charte.py` et produit des HTML conformes au contrat d'envoi ; rien à changer dans `brevo.py`.
- `data/` — entrées **sensibles, non versionnées**. `out/` — entrées JSON + sorties (gitignoré).
- `.env` — secrets (voir `.env.example`).

## Rendu / envoi (exemple : type SPS)

Entrée nominale = **JSON** (un par conseiller, format `docs/schema/email.schema.json`) déposé dans `out/json/`. Procédure pas-à-pas : voir `README.md`. En bref :

```bash
uv sync
uv run emailer deanon   out/json/ --csv <le-vrai.csv> -o out/json-nom/   # optionnel (vrais noms)
uv run emailer render   out/json-nom/                  -o out/html/      # ou out/json/ pour l'anonyme
# tunnel SOCKS ouvert + BREVO_PROXY exporté (cf. ci-dessous), puis :
uv run emailer send     out/html/ --test                                  # comptes de test (.env)
uv run emailer send     out/html/                                         # vrais conseillers (e-mail embarqué)
uv run emailer schedule out/html/ --at 2026-06-29T07:00:00.000+02:00      # → runId
uv run emailer cancel   <runId>                                           # annule un programmé
uv run emailer purge                                                      # efface les sorties nominatives > 7 j (confirmation)
```

Débranchement : sans `deanon`, `render out/json/` produit des HTML anonymes (`Bénéficiaire #N`) envoyables en test.

## Format d'entrée (type SPS)

Le contrat d'entrée est **JSON** (un fichier par conseiller), défini dans `docs/schema/email.schema.json`. Points clés : `section.type` est une **string ouverte** (types inconnus → gabarit générique) ; contenus libres via blocs riches `aside`/`before_block`/`after_block` (`{format: markdown|html, content}`) ; intro + remarques portées par le template. (`emailer convert` reste un adaptateur legacy MD→JSON, hors chemin nominal.)

## Design des e-mails

Principes valables pour **tous les types** (les puces bénéficiaire/tension/sommaire concernent le type SPS). Le COMMENT — couleurs, tailles, structure HTML exacte, espacements — vit dans le code, **qui fait foi** (`emailer/charte.py` pour le commun, puis `render.py` + `template.py` pour SPS, `render_<type>.py` pour les autres) : modifier la mise en forme = modifier ces modules, pas dupliquer un pixel-spec ici.

- **Lisible desktop ET mobile**, sans dépendre d'une media query (cf. Outlook/Gmail ci-dessous).
- **Anticiper Outlook** comme client cible : table wrapper + styles **100 % inline**, pas de CSS hostile (fl/grid/positionnement), largeur fixe (~700px).
- **Valoriser l'espace vertical sans excès** : assez d'air pour scanner rapidement, mais pas de blancs gratuits.
- **Mobile : éviter les doubles marges horizontales** (gouttière du wrapper + padding interne) → bord à bord propre. ⚠️ Le `<style>`/`@media` du `<head>` est strippé par Gmail mobile : ne pas en dépendre pour la lisibilité de base, c'est un bonus.
- **Hiérarchie visuelle claire** : panneau bénéficiaire > boîte de groupe (`▸`) > nom de structure/contrainte > légende de section discrète.
- **Un panneau par bénéficiaire**, séparés par l'**élévation** (ombre) plutôt que des bordures ; **sommaire** « Accès rapide » avec ancres `#de-N` (dégrade en simple liste là où l'ancre ne saute pas — Gmail/Outlook).
- **Pastille de tension** sous le nom de la structure ; **CTA bleus libellés par domaine** (Dora, Les Emplois, formation, Agefiph…) ; **avis** en pied = lien méta discret, **pas** un CTA solution.
- **Blocs riches** `aside`/`before_block`/`after_block` (markdown/HTML) pour le contenu éditorial libre (ex- « 💡 Le saviez-vous ? »).
- **Liens** : téléphones → `tel:`, e-mails → `mailto:`. **Données échappées** (`esc`) partout, sauf les blocs riches (HTML assumé par l'amont).

## Envoi Brevo

- Sender : variable d'env **`BREVO_SENDER`** (requise, définie dans `.env`) — doit être un expéditeur/domaine **validé** dans Brevo.
- Objet préfixé **`[TEST]`** en mode `--test` (l'objet vient du `<title>`).
- **IP autorisées** : l'IP appelante doit être ajoutée dans Brevo (Réglages > IP autorisées) sinon **401** (IPv6, change selon le réseau).
- Destinataires de test : variable d'env **`TEST_RECIPIENTS`** (liste séparée par des virgules), forcés quand `--test` est passé.
- **Pré-filtre blocklist** (par défaut, hors `--test`) : `send`/`schedule` récupèrent la **blocklist transactionnelle Brevo** (`GET /v3/smtp/blockedContacts` — hard bounces, désinscriptions, spam) et **excluent ces adresses avant l'appel** (déterministe, aucun LLM ; aucune adresse affichée/loggée). Brevo les droppe de toute façon → c'est une optimisation (0 appel inutile, stats propres, désinscriptions respectées). Si la blocklist est injoignable, l'envoi continue **sans** pré-filtre (dégradation propre). Désactiver : **`--no-blocklist-filter`**.
- **Envoi programmé** (`schedule --at <ISO8601>`) : `scheduledAt` met en file côté Brevo ; l'envoi part des serveurs Brevo, pas de votre IP. La restriction d'IP autorisée s'applique à **l'appel API** (401 si IP non listée au moment de programmer), **pas** à l'envoi différé → une seule fenêtre IP autorisée à la programmation suffit. Horizon limité (~72 h, à vérifier). Alternative : élargir/désactiver la restriction d'IP dans Brevo si l'IPv6 change trop. Format `scheduledAt` accepté : `2026-06-29T07:00:00.000+02:00`.
- **Annulation** : pas d'annulation via la **GUI** Brevo → seulement par API (`DELETE /v3/smtp/email/{messageId}`). `schedule` génère un **`runId`** (affiché + loggé dans `out/schedules.jsonl` avec les `messageId`, sans aucune adresse/nom) ; `uv run emailer cancel <runId>` supprime tout le lot. ⚠️ l'annulation passe aussi par le tunnel (IP autorisée requise au moment du `cancel`).

### Egress à IP fixe (tunnel SOCKS)

L'IPv6 résidentielle change → on route les appels Brevo via une **box à IP fixe whitelistée**, par un **tunnel SSH SOCKS** (TLS de bout en bout : la box ne voit ni la clé ni le HTML nominatif). **Les coordonnées réelles de la box (hôte, IP, users) ne sont PAS versionnées** : elles vivent dans `.env` (gitignoré) ; chaque utilisateur a les siennes (transmises hors Git) et sa propre clé SSH ajoutée sur la box. Renseigner dans `.env` : `EMAILER_TUNNEL_HOST`, `EMAILER_TUNNEL_USER`.

```bash
ssh -i ~/.ssh/sps-tunnel -D 1080 -N -f "$EMAILER_TUNNEL_USER@$EMAILER_TUNNEL_HOST"   # ouvre le SOCKS
BREVO_PROXY=socks5h://127.0.0.1:1080 uv run emailer send out/html/ --test         # ou: --via socks5h://127.0.0.1:1080
```
⚠️ Utiliser **`socks5h`** (DNS résolu côté box) — requis pour le `PermitOpen api.brevo.com:443` du tunnel. L'IP de la box est une **IP flexible** (réattachable à une future box).

## À garder en tête

- Le `…` de troncature présent dans certaines lignes vient de la **source**, conservé tel quel.
- Les anciens scripts (`render.py`, `render2.py`, `send.py`, `bin/replace_ids_by_names.sh`) sont remplacés par `emailer/` et supprimés.
- Confirmer le layout exact des colonnes nom/prénom du CSV réel (p. ex. `Lille.csv` — un exemple parmi d'autres) avant de figer `deanon` (cf. `emailer/deanon.py`).

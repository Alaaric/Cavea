# Décisions d'architecture (ADR)

Format : contexte → décision → conséquences → alternatives rejetées.

**À lire avant de proposer un changement de stack ou de modélisation.** La plupart des
« améliorations » évidentes ont déjà été écartées ici, pour des raisons écrites.

---

## ADR-001 — Pas de Doctrine ORM, DBAL seul

**Statut** : acté

### Contexte

L'ORM est le défaut de l'écosystème Symfony. La tentation est de l'utiliser sans y penser.

### Décision

**Doctrine DBAL uniquement** (+ `doctrine/migrations`). Pas d'ORM.

### L'argument qu'il ne faut PAS utiliser

> « L'ORM couple le domaine à la base. »

**C'est faux ici, et ça se voit en entretien.** Doctrine ORM est un *Data Mapper*, pas un
Active Record : avec un mapping XML, l'agrégat n'a aucun `use Doctrine\`, aucun attribut,
aucune classe parente. Le couplage redouté existe avec Eloquent, où l'entité hérite du
modèle et parle à la base. Il n'existe pas avec Doctrine correctement configuré.

Un lead Symfony corrigera cet argument en trois secondes.

### Les vraies raisons

1. Le write model est **un** agrégat, petit. Unit of Work, identity map, lazy loading :
   inutilisés.
2. Les read models sont des projections SQL. L'ORM ne servirait jamais à les lire.
3. L'UoW **complique** le lock optimiste + retry plus qu'il n'aide.
4. Le lock optimiste devient explicite et lisible :
   `UPDATE … WHERE version = :expected` + `rowCount()`, au lieu du `@Version` magique.

### Conséquences

- Repositories à la main : ~80 lignes par agrégat
- Hydratation manuelle, constructeurs privés + named constructors
- **Migrations écrites à la main** — pas de `diff`, il n'y a plus d'entités mappées.
  C'est le vrai coût de cette décision, et il se sent dès l'étape 1.
- Middleware de transaction maison (celui de Symfony dépend de l'EntityManager)

### Rejeté

- **Doctrine ORM + mapping XML** : marcherait très bien, mais n'apporte rien ici (cf. raisons 1–3)
- **PDO nu** : on perdrait `doctrine/migrations` pour rien
- **Cycle ORM** : mêmes bénéfices, moins de doc, aucun gain

---

## ADR-002 — Symfony 7.4 LTS plutôt que 8.x

**Statut** : acté

### Contexte

Au 16/07/2026 : Symfony 8.1 est la stable (mai 2026, exige PHP 8.4+). Symfony 7.4 est la
LTS (nov. 2025), correctifs jusqu'en nov. 2028, sécurité jusqu'en nov. 2029.

### Décision

**7.4 LTS.** Objectif de stabilité assumé : un projet portfolio doit rester déployable et
à jour longtemps sans travail de maintenance.

### Conséquences

- Pas les nouveautés 8.x
- Fenêtre de tranquillité jusqu'en 2028
- Choix défendable en entretien : c'est le choix « entreprise »

### À savoir

Sur cette stack, **seuls Symfony et Node ont une vraie notion de LTS**. PHP, PostgreSQL,
React et Three.js n'en ont pas — ils ont des fenêtres de support. « La LTS de React » n'existe pas.

Pour eux, la politique est : **version stable avec la plus longue fenêtre restante.**
D'où PHP 8.5 (sécurité → fin 2029) et PostgreSQL 18 (→ 2030).

---

## ADR-003 — RabbitMQ *et* Mercure

**Statut** : acté

### Contexte

Confusion fréquente : « on veut RabbitMQ pour faire de l'EDA async, pas Mercure ».

### Décision

**Les deux.** Ce ne sont pas des alternatives — ils sont l'un derrière l'autre.

| | RabbitMQ | Mercure |
|---|---|---|
| Sens | serveur ↔ serveur | serveur → navigateur |
| Protocole | AMQP | SSE (sur HTTP) |
| Rôle | l'EDA async entre workers | le dernier kilomètre vers l'écran |
| Alternatives | transport Doctrine, Redis, Kafka | WebSocket, polling |

Un navigateur ne parle pas AMQP. RabbitMQ ne peut pas pousser vers un onglet.

### Pourquoi RabbitMQ plutôt que le transport Doctrine

- delayed exchange → saga d'expiration sans bricolage
- vraies DLQ
- plus lisible sur un CV que « une table Postgres »
- coût : ~200 Mo de RAM sur le VPS, assumé, avec une limite mémoire explicite

### Pourquoi Mercure plutôt qu'un WebSocket

Le flux est **unidirectionnel** : le serveur pousse des deltas, le client agit par POST HTTP.
SSE suffit. Mercure = un binaire Go, un container, et `new EventSource(url)` côté React.
Écrire un serveur WebSocket serait du travail en pure perte.

---

## ADR-004 — Spec paramétrique, aucune coordonnée en base

**Statut** : acté

### Décision

`VenueCatalog` stocke un spec paramétrique en JSONB (~2 Ko / 2000 places).
`Ticketing` matérialise `(sectionId, rowLabel, seatNumber)` + statut.
**Aucun x/y/z en base.** Le frontend dérive les positions.

### Raison

La position d'un siège dans l'espace est un détail de présentation. Le domaine connaît
l'identité et l'état. Stocker des coordonnées, c'est faire fuiter le rendu dans le métier —
et se condamner à une migration à chaque ajustement visuel.

### Conséquence : le risque à traiter

L'algo est implémenté deux fois (PHP + TS). S'ils divergent, les identités divergent
silencieusement. **Le golden fixture est la contrepartie obligatoire de cette décision**
(voir `GEOMETRY.md`). Ce n'est pas un test bonus, c'est ce qui rend l'ADR viable.

---

## ADR-005 — L'agrégat est la section, pas la performance

**Statut** : acté

### Contexte

Frontière d'agrégat la plus tentante : `Performance` porte tout son inventaire.

### Décision

Un `SeatInventory` par `(performanceId, sectionId)`.

### Raison

Avec un agrégat par performance, 2000 acheteurs simultanés se battent pour le même objet :
le lock optimiste échoue en permanence, le retry n'aide pas, on a réinventé le mutex global.

La section est la plus petite frontière qui préserve l'invariante utile (pas de double
booking) tout en permettant la concurrence entre sections.

### Conséquence

Une commande touchant plusieurs sections touche plusieurs agrégats → pas d'atomicité inter-agrégats.
Traité par saga, pas par transaction. C'est du DDD orthodoxe, et c'est voulu.

---

## ADR-006 — CQRS ciblé, pas généralisé

**Statut** : acté

### Décision

Séparation stricte (write / read / projections) **uniquement** dans `Ticketing` et `Ordering`.
Ailleurs : repository classique, requête directe, aucune projection.

### Raison

Un projet où *tout* est over-engineered est aussi suspect qu'un projet sans archi. Un jury
lit la capacité à **choisir** où mettre la complexité, pas la capacité à en mettre partout.

### Règle opérationnelle

Avant d'ajouter une projection : nommer en une phrase le problème de lecture qu'elle résout.
Si la phrase ne vient pas, la projection ne se fait pas.

---

## ADR-007 — Performance, pas Event

**Statut** : acté

### Contexte

Le premier découpage utilisait `Event` pour le spectacle daté : `EventId`,
`/events/{id}/availability`. Or `Domain\Event\` désigne les messages de domaine.

### Le problème

**Le mot « event » avait deux sens dans un projet dont la thèse est DDD + EDA.**
`Domain\Event\Event` était garanti d'apparaître sous quelques semaines. Aucune code review
ne survit à ça.

### Décision

- `Show` = l'œuvre (« Hamlet »)
- `Performance` = la représentation datée. **C'est ce qu'on achète.**
- `Event` = **uniquement** un message de domaine

### Bonus

Ce n'est pas un contournement : c'est le vocabulaire réel du métier. Une billetterie vend
des représentations, pas des spectacles. Le renommage rend la modélisation *plus* juste,
pas moins.

### Coût

Fait avant toute ligne de code. Six semaines plus tard, c'eût été 200 fichiers.

---

## ADR-008 — SVG 2D avant Three.js

**Statut** : acté

### Décision

L'étape 1 rend la salle en **SVG 2D**, zéro Three.js. Three.js n'arrive qu'à l'étape 2,
sur la même data et le même flux.

### Raison

Les vraies difficultés sont la concurrence et la saga, pas le rendu. En commençant par la
3D, on obtient une belle salle et pas de backend — l'échec classique de ce genre de projet.

Le SVG prouve la boucle complète command → event → RabbitMQ → projection → Mercure → écran.
Une fois qu'elle tourne, Three.js n'est qu'un changement de couche de rendu.

### Conséquence

Aucune. Le SVG est jeté à l'étape 2 et ce n'est pas du travail perdu : c'est ce qui a validé
l'architecture.

---

## ADR-009 — PolyForm Noncommercial 1.0.0

**Statut** : acté

### Contexte

Objectif : « je ne veux pas que ce soit repris ».

### Ce qu'une licence ne règle pas

Deux menaces distinctes :

1. quelqu'un déploie le code commercialement → une licence règle ça
2. un candidat clone le repo et le présente comme son projet en entretien → **aucune licence
   ne règle ça**

Pour un portfolio, la menace réelle est la n°2. Un plagiaire ne lit pas le LICENSE.

### Décision

**PolyForm Noncommercial 1.0.0.** Source-available, pas open source : lecture, fork et
usage perso autorisés, usage commercial interdit. Rédigée par des juristes, en langage clair.

### Conséquences pratiques

- **Absente du menu GitHub** : ce menu ne liste que des licences approuvées OSI. Or une
  licence qui interdit l'usage commercial ne peut pas être open source (clause de
  non-discrimination de l'OSD). Ajouter le fichier à la main.
- Texte officiel copié depuis polyformproject.org, **jamais retapé de mémoire**. Un texte
  juridique approximatif est pire que pas de licence.
- Le nom s'attache via une ligne `Required Notice:` en tête, pas via un champ à remplir.
- Pas de badge dans la sidebar GitHub (même liste de détection). Cosmétique — mettre une
  ligne explicite dans le README, plus lue de toute façon.
- Dépendances toutes en MIT (Symfony, DBAL, React, Three.js) : aucune contrainte amont.

### Rejeté

- **Aucun fichier** : « tous droits réservés » par défaut, mais ambigu, et ressemble à un
  oubli plutôt qu'à un choix
- **AGPL-3.0** : le réflexe habituel, mais elle **autorise** la reprise — elle la rend
  seulement contraignante. Ne correspond pas à l'objectif énoncé.
- **CC BY-NC-ND** : Creative Commons déconseille elle-même ses licences pour du code

### La vraie protection

Historique Git public et étalé, instance déployée en ligne avant tout le monde, articles
expliquant les arbitrages. Un clone n'a rien de tout ça — et son auteur ne saura pas
répondre à « pourquoi lock optimiste plutôt que pessimiste ici ? ».

---

## ADR-010 — Schéma de persistance de l'étape 1

**Statut** : acté

### Contexte

`ROADMAP.md` fixe le périmètre de l'étape 1 : une section `grid`, une `Performance`,
l'inventaire, le hold, la saga d'expiration, la projection `seat_availability`. Avant
d'écrire la première migration, plusieurs points ne sont pas tranchés par
`ARCHITECTURE.md` — en particulier la granularité de `seat_inventory` et ce que
recouvre exactement l'« index unique partiel » qu'il mentionne. Le détail du schéma
(MCD/MLD) est dans `docs/DATA_MODEL.md` ; cet ADR fixe les choix structurants qui le
rendent possible.

### Décision

1. **Un schéma Postgres par bounded context** (`venue_catalog`, `ticketing`, `ordering`,
   `notification`), un seul Postgres physique. Zéro FK traversant un schéma — une
   référence à un autre contexte est un ID copié, reçu par événement d'intégration,
   jamais une clé étrangère.

2. **Pas de table `Seat` dans `VenueCatalog`.** Il ne stocke que le spec paramétrique
   (JSONB). `SeatLayoutGenerator` est une fonction pure, invoquée une seule fois à la
   création de la `Performance`. C'est `Ticketing` qui matérialise et qui possède
   l'identité du siège dans ses propres lignes — jamais de second aller-retour vers
   `VenueCatalog` après matérialisation (cf. `ARCHITECTURE.md` § Frontière).

3. **`seat_inventory` : une ligne par siège**, avec sa propre colonne `version`.
   L'agrégat « la section » (ADR-005) reste une frontière **transactionnelle** — le
   handler `HoldSeats` lit/écrit plusieurs lignes d'une même section dans une seule
   transaction — pas une ligne physique unique en JSONB. Cohérent avec l'
   `UPDATE … WHERE version = :expected` déjà décrit dans `ARCHITECTURE.md`, et plus
   simple à écrire à la main en DBAL qu'une manipulation de JSON.

4. **`Hold` est une entité à part**, sans table de jonction many-to-many : un hold ne
   détient chaque siège qu'une fois.
   - `hold(id, performance_id, expires_at, idempotency_key UNIQUE, status)`
   - `hold_seat(hold_id, performance_id, section_id, row_label, seat_number, released_at NULL)`,
     **append-only** — jamais d'update, un insert au hold, `released_at` renseigné à la
     libération.

   L'index unique partiel de `ARCHITECTURE.md` porte sur cette table :
   ```sql
   CREATE UNIQUE INDEX ON hold_seat (performance_id, section_id, row_label, seat_number)
     WHERE released_at IS NULL;
   ```
   C'est le filet ultime contre le double hold, indépendant du contrôle de version : si
   le check applicatif a un bug, l'index bloque quand même. `seat_inventory` garde en
   parallèle un statut + version dénormalisés pour les lectures rapides, écrits dans la
   même transaction que l'insert dans `hold_seat`.

5. **Le prix est snapshotté, mais pas sur chaque siège.** Une table
   `performance_section(performance_id, section_id, label, price_cents)`, remplie une
   fois à la création de la `Performance` depuis `VenuePublished`. `seat_inventory`
   référence `section_id` et se joint pour le prix — pas de duplication sur 200+ lignes
   de sièges identiques.

6. **Pas de table d'idempotence générique.** Une contrainte `UNIQUE` sur
   `hold.idempotency_key` suffit pour l'étape 1 — une seule commande mutative existe. Une
   table partagée entre contextes violerait la décision 1 ; le pattern se répète
   localement si un autre contexte en a besoin plus tard.

7. **`seat_availability` est une table matérialisée dédiée**, une ligne par
   performance, colonne `buffer BYTEA` déjà packée. Le projecteur flippe l'octet
   concerné à chaque `SeatsHeld`/`SeatsReleased`, jamais de recalcul complet. Implique
   que `seat_inventory` porte une colonne `seat_index` (0..N-1, ordre du golden
   fixture) — nécessaire de toute façon pour le picking Three.js à l'étape 2, donc pas
   un ajout gratuit.

8. **`Show` n'est pas modélisé à l'étape 1.** `Performance` porte un simple champ
   `title`. Une table `Show` attend qu'un besoin de lecture réel apparaisse (ex.
   l'Accueil groupant plusieurs Performances) — conforme à ADR-006, « nommer le
   problème de lecture avant d'ajouter ».

### Conséquences

- Le détail des tables (colonnes, types, contraintes) est dans `docs/DATA_MODEL.md`,
  pas ici — cet ADR ne fixe que la structure et le pourquoi.
- Trois défenses contre le double booking cohabitent, comme voulu par
  `ARCHITECTURE.md` : l'index partiel sur `hold_seat` (filet ultime), le
  `version` optimiste sur `seat_inventory` (contrôle applicatif), le retry sur
  `ConcurrencyException` dans le middleware Messenger.
- `hold_seat` grossit indéfiniment (append-only) ; un nettoyage périodique des lignes
  libérées de longue date est un problème d'exploitation, pas de modélisation — à
  traiter plus tard, pas maintenant.

### Rejeté

- **Une ligne JSONB par section pour `seat_inventory`** : complique l'`UPDATE`
  optimiste écrit à la main et le picking par `instanceId` à l'étape 2.
- **Table d'idempotence générique cross-contexte** : violerait l'isolation des
  schémas (décision 1).
- **Table `Seat` séparée dans `VenueCatalog`** : `VenueCatalog` ne doit rien savoir de
  la vente ; ajouterait un aller-retour que la frontière interdit explicitement.
- **`Show` modélisé dès l'étape 1** : aucun besoin de lecture ne le justifie encore.

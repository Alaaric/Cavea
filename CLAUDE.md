# Cavea

Billetterie événementielle : salles paramétriques, inventaire de sièges, hold temporaire,
paiement, plan de salle 3D cliquable avec prévisualisation « vue depuis le siège ».

Projet perso à visée portfolio. DDD / hexagonal, CQRS ciblé, event-driven, CI/CD complète.
**L'over-engineering systématique est un anti-objectif.**

> *Cavea* : le terme latin pour la cuvette de gradins d'un théâtre romain, divisée en
> *maeniana* (étages) et *gradus* (rangées) — soit exactement le modèle du spec de salle.
> **Le code reste en anglais courant.** Aucune classe ne s'appellera `Maenianum`.

## Documentation

| Fichier | Contenu |
|---|---|
| `docs/ARCHITECTURE.md` | Bounded contexts, arborescence, CQRS, messaging, persistance, concurrence |
| `docs/GEOMETRY.md` | Spec paramétrique, algo de génération, golden fixture, Three.js |
| `docs/DECISIONS.md` | Les ADR. **À lire avant de proposer un changement de stack.** |
| `docs/DATA_MODEL.md` | MCD/MLD de l'étape 1 — schéma de persistance |
| `docs/ROADMAP.md` | Étapes, definition of done |

Ce fichier ne contient que ce qui doit être vrai à **chaque** tour. Le détail est dans `docs/`.

---

## Versions — figées, vérifiées le 16/07/2026

| Techno | Version | Ne pas bouger sans décision explicite |
|---|---|---|
| PHP | **8.5** | Support actif → 12/2027, sécurité → 12/2029 |
| Symfony | **7.4 LTS** | **Pas 8.x.** Correctifs → 11/2028, sécurité → 11/2029 |
| PostgreSQL | **18** | **Pas 19** (beta, GA vers 09-10/2026). Supporté → 2030 |
| Node | **24 « Krypton »** | Active LTS → **10/2026**, puis maintenance. EOL 04/2028 |
| React | **19.2.x** | — |
| Three.js | **version exacte** | Casse entre mineures. **Jamais de `^`.** |
| RabbitMQ | 4.x | + plugin delayed message exchange |

Épingler majeure **et** mineure partout : `Dockerfile`, `composer.json`, `package.json`,
`.nvmrc`, matrice CI. Build reproductible ou rien.

**Échéance à surveiller : octobre 2026.** Node 24 passe en maintenance, et Node 27 inaugure
le nouveau modèle (une majeure/an, toutes LTS). Migration de toolchain, risque nul, à planifier.

---

## Commandes

Tout passe par le Makefile. Ne jamais lancer `php`, `composer` ou `npm` sur l'hôte.

```bash
make up / down / sh
make test            # phpunit + vitest
make test-unit       # domaine seul, aucun I/O
make test-integration
make qa              # phpstan + cs-fixer + deptrac + eslint + tsc — ce que la CI vérifie
make migration       # squelette vide, SQL écrit à la main
make migrate
make fixtures
make worker          # messenger:consume async delayed
```

Un commit passe `make qa && make test` avant d'être poussé. Sans exception.

---

## Langue ubiquitaire

**Un mot = un sens, dans un seul contexte.** Si deux contextes se disputent un mot,
la frontière est mal placée.

| Terme | Sens exact | Contexte |
|---|---|---|
| `Venue` | La salle physique | VenueCatalog |
| `Seat` | Position physique, existe indépendamment de toute vente | VenueCatalog |
| `Section` | Groupe de rangées partageant une géométrie et un tarif | VenueCatalog |
| `Show` | L'œuvre (« Hamlet ») | VenueCatalog |
| `Performance` | **Une représentation datée.** C'est ce qu'on achète. | Ticketing |
| `Hold` | Blocage temporaire, TTL 10 min, non payé | Ticketing |
| `Allocation` | Siège définitivement attribué après paiement | Ticketing |
| `Order` | Intention d'achat : panier + paiement | Ordering |
| `Ticket` | Le titre émis après paiement, avec son QR | Ordering |

### Interdit absolu : le mot « event »

`Event` désigne **uniquement** un message de domaine (`Domain\Event\SeatsHeld`).
Le spectacle daté s'appelle `Performance`. Jamais `EventId`, jamais `/events/{id}`.

Un projet dont la thèse est DDD + EDA ne peut pas avoir deux sens pour « event ».
C'est la règle la plus facile à violer et la plus coûteuse à réparer.

---

## Règles de données

- **Argent** : entiers, en centimes, via un VO `Money`. **Jamais de float.** Jamais de `float` en base.
- **Temps** : tout en UTC en base (`timestamptz`). La timezone est une propriété du `Venue`,
  appliquée à l'affichage uniquement.
- **IDs** : UUIDv7 générés **côté client**, passés dans la commande. Les handlers retournent `void`.
- **Idempotence des commandes** : toute commande mutative porte une clé d'idempotence.
  Un double-clic ne crée pas deux holds.
- **Coordonnées** : aucune. Voir `docs/GEOMETRY.md`.

---

## Règles non négociables

### Dépendances

`Domain` ← `Application` ← `Infrastructure`. Vérifié par deptrac en CI.

Dans `Domain/` : pas de `use Doctrine\`, pas de `use Symfony\`, pas de
`new \DateTimeImmutable()` (injecter le port `Clock`), pas de génération d'ID,
pas de réseau, pas de fichier, pas de `$_ENV`.

Un agrégat est testable sans conteneur DI, sans base, en < 1 ms.

### Persistance

**Pas de Doctrine ORM. DBAL seul.** Repositories et migrations écrits à la main.
Lock optimiste explicite (`UPDATE … WHERE version = :expected`, vérifier `rowCount()`).

Motivation réelle dans `docs/DECISIONS.md` — **et ce n'est pas « l'ORM couple le domaine »,
qui est faux pour un Data Mapper.** Ne pas ressortir cet argument.

### Concurrence

**L'agrégat est la section, pas la performance.** Un `SeatInventory` par
`(performanceId, sectionId)`. Sinon 2000 personnes se battent pour un agrégat unique.

Trois défenses obligatoires : unique index partiel + lock optimiste + retry.
Jamais de lock pessimiste sur l'inventaire complet.

### Messaging

**RabbitMQ ≠ Mercure. Ils sont l'un derrière l'autre, pas en concurrence.**
RabbitMQ = serveur↔serveur. Mercure = serveur→navigateur (SSE).

Tout handler d'événement est **idempotent** : AMQP est at-least-once.
Un événement de domaine ne sort jamais de son contexte — il devient un événement
d'intégration, à payload primitif et versionné.

### Géométrie

Aucune coordonnée en base. Le spec paramétrique est la source unique, dérivé
identiquement en PHP et en TS. Le **golden fixture** verrouille les deux.
**Ne jamais régénérer le golden pour faire passer la CI.**

### Frontend

Un seul `InstancedMesh` pour tous les sièges. État → `setColorAt()`, jamais de
reconstruction de géométrie. Si Mercure tombe, bascule en polling — jamais d'écran cassé.

---

## Pièges connus

- ❌ Le mot « event » pour une représentation → c'est `Performance`
- ❌ Coordonnées x/y/z en base → la géométrie est de la présentation
- ❌ Un mesh par siège → 20 000 draw calls, la page meurt
- ❌ Agrégat = la performance entière → contention globale
- ❌ Confondre RabbitMQ et Mercure → serveur↔serveur vs serveur→navigateur
- ❌ Justifier l'absence d'ORM par « ça couple le domaine » → faux pour un Data Mapper
- ❌ Réintroduire Doctrine ORM « juste pour cette entité » → non
- ❌ `new DateTimeImmutable()` dans le domaine → port `Clock`
- ❌ Handler non idempotent → AMQP va le rejouer
- ❌ Projection ajoutée « au cas où » → nommer le problème de lecture, ou renoncer
- ❌ Golden fixture régénéré pour faire passer la CI → c'est le bug qu'il détecte
- ❌ `sleep()` dans un test de saga → `FrozenClock`
- ❌ Migration dans l'entrypoint du container app → job séparé
- ❌ `^` sur Three.js → version exacte
- ❌ Float pour de l'argent → centimes en entier
- ❌ Upgrade Symfony 8.x / PostgreSQL 19 → on est en LTS, c'est un choix
- ❌ Nommer une classe en latin parce que le projet s'appelle Cavea → non

---

## Commits

Conventional Commits, scope = bounded context.

```
feat(ticketing): hold expiration saga
fix(venue-catalog): arc row seat count off-by-one
refactor(ordering): extract payment port
```

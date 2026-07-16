# Architecture

Détail de ce que `CLAUDE.md` résume. Les *pourquoi* sont dans `DECISIONS.md`.

---

## Bounded contexts

Quatre contextes isolés. **Aucun ne référence les classes d'un autre.** Communication
uniquement par événements d'intégration.

| Contexte | Responsabilité | Ne sait pas |
|---|---|---|
| `VenueCatalog` | Salles, spec géométrique, sections, shows, tarifs | Ce qu'est une vente |
| `Ticketing` | Performances, inventaire, hold, allocation. **Cœur du projet.** | Qui paie, comment |
| `Ordering` | Panier, commande, saga d'expiration, paiement, tickets | Les invariantes de placement |
| `Notification` | Emails, confirmations | Tout le reste |

`Ticketing` est le seul contexte qui mérite une modélisation tactique complète.
Les autres restent volontairement plus légers. C'est un choix, pas une dette.

### Frontière VenueCatalog / Ticketing

`VenueCatalog` possède le spec et le publie (`VenuePublished`).
`Ticketing` le consomme à la création d'une `Performance` et **matérialise l'inventaire** :
une ligne par siège, avec `(sectionId, rowLabel, seatNumber)` et un statut.

Après matérialisation, `Ticketing` ne redemande plus rien à `VenueCatalog`. Une salle
republiée ne modifie pas rétroactivement les performances déjà ouvertes à la vente.

---

## Arborescence

```
src/
  <Context>/
    Domain/            # zéro dépendance
      Model/           # agrégats, entités, VO
      Event/           # événements de domaine (le SEUL sens du mot "Event")
      Repository/      # interfaces uniquement (ports)
      Exception/
      Service/         # domain services, ssi le comportement n'a pas d'agrégat naturel
    Application/       # orchestration. Dépend de Domain uniquement.
      Command/         # DTO + Handler
      Query/           # DTO + Handler
      Port/            # interfaces sortantes (Clock, PaymentGateway, IdGenerator…)
      EventListener/
      Integration/     # événements d'intégration publiés par ce contexte
    Infrastructure/    # adapters. Dépend de tout.
      Persistence/     # repositories DBAL, hydratation, SQL
      Http/            # controllers, requests, responses
      Projection/      # projecteurs vers les read models
      Messaging/       # RabbitMQ, middleware de transaction
      Mercure/
      Payment/
  Shared/              # VO transverses (Money, SeatId…), bus, kernel. Rester très maigre.
```

`Shared/` est un piège classique : tout finit par y atterrir. Règle — un VO n'y entre que
s'il est utilisé par **au moins trois** contextes et n'a **aucune** règle métier.

---

## Règle de dépendance

`Domain` ← `Application` ← `Infrastructure`, vérifié par deptrac en CI.

Interdits dans `Domain/` :

| Interdit | Remplacer par |
|---|---|
| `use Doctrine\…`, `use Symfony\…` | rien, le domaine est en PHP nu |
| `new \DateTimeImmutable()` | port `Clock` |
| `Uuid::v7()`, `uniqid()` | port `IdGenerator`, ou ID passé dans la commande |
| Appel réseau, fichier, `$_ENV` | port dédié |

Un agrégat se teste sans conteneur DI, sans base, en < 1 ms.

---

## Persistance — DBAL, sans ORM

### Le principe

DBAL n'est pas un ORM : abstraction PDO + query builder. Elle vit exclusivement dans
`Infrastructure/Persistence/`. Le domaine ne la voit jamais. `doctrine/migrations`
fonctionne sans l'ORM.

**Attention à l'argumentaire.** Doctrine ORM est un Data Mapper : avec un mapping XML,
le domaine n'aurait aucun `use Doctrine\`. « L'ORM couple le domaine à la base » est faux
ici (vrai pour Eloquent, où l'entité hérite du modèle). Les vraies raisons sont dans
`DECISIONS.md` § ADR-001.

### Ce que ça implique

**Repositories à la main.** Interface dans `Domain/Repository/`, implémentation DBAL dans
`Infrastructure/Persistence/`. ~80 lignes par agrégat.

```php
interface SeatInventoryRepository
{
    public function get(PerformanceId $p, SectionId $s): SeatInventory;
    public function save(SeatInventory $inventory): void;  // throws ConcurrencyException
}
```

**Hydratation manuelle.** `fromRows(array $rows): SeatInventory`, constructeur privé,
named constructor. Pas de reflection, pas de proxy.

**Lock optimiste explicite** :

```sql
UPDATE seat_inventory
   SET status = :status, version = version + 1
 WHERE id = :id AND version = :expectedVersion
```

`rowCount() === 0` → `ConcurrencyException`. Trois lignes lisibles au lieu du `@Version` magique.

**Migrations à la main.** Pas de `diff` — il n'y a plus d'entités mappées. `make migration`
génère un squelette vide, tu écris le SQL. Le schéma est une décision, pas un dérivé.

**Middleware de transaction maison.** Celui de Symfony s'appuie sur l'EntityManager.
Écrire un `DbalTransactionMiddleware` (ouvre / commit / rollback sur la `Connection`),
~30 lignes, dans `Infrastructure/Messaging/`.

---

## CQRS — ciblé

Séparation stricte **uniquement** dans :

- `Ticketing` — write model riche en invariantes, read model qui sert 20 000 sièges en un GET
- `Ordering` — sagas et historique

Ailleurs (auth, profil, back-office salle) : repository classique, requête directe,
**pas de projection**. Le métier est pauvre, la séparation coûterait plus qu'elle ne rapporte.

Un projet où *tout* est over-engineered est aussi suspect qu'un projet sans archi.
Avant d'ajouter une projection : nommer en une phrase le problème de lecture qu'elle résout.
Sinon, non.

- **Command handlers** : retournent `void`. ID généré côté client, passé dans la commande.
- **Query handlers** : ne touchent jamais aux agrégats. SQL direct via DBAL sur les read
  models. Retournent des DTO plats.

---

## Messaging

### RabbitMQ et Mercure ne font pas la même chose

Ce ne sont **pas** des alternatives. Ils sont l'un derrière l'autre.

- **RabbitMQ** — broker, **serveur ↔ serveur**. Transporte les messages entre le contrôleur
  HTTP et les workers. C'est l'EDA async.
- **Mercure** — push **serveur → navigateur** via SSE. Le dernier kilomètre, que RabbitMQ ne
  peut pas faire : un navigateur ne parle pas AMQP. Flux unidirectionnel, donc SSE suffit —
  le client POST en HTTP normal pour agir.

```
POST /performances/{id}/holds → HoldSeats → agrégat → SeatsHeld
  → Messenger → [RabbitMQ] → worker → projection mise à jour
  → publish → [Mercure hub] → navigateur (EventSource) → setColorAt() → siège orange
```

### Deux niveaux d'événements

| | Événement de domaine | Événement d'intégration |
|---|---|---|
| Exemple | `Ticketing\Domain\Event\SeatsHeld` | `Ticketing\Application\Integration\SeatsHeldV1` |
| Portée | interne au contexte | contrat public entre contextes |
| Payload | objets du domaine | primitifs uniquement |
| Versionné | non | oui (suffixe `V1`) |

**Un événement de domaine ne sort jamais de son contexte.**

### Transports

```yaml
sync:      # queries
async:     # projections, notifications          → AMQP
delayed:   # saga d'expiration des holds         → AMQP + delayed exchange
failed:    # DLQ, à monitorer
```

### La saga d'expiration

```
HoldSeats
  → agrégat vérifie les invariantes
  → SeatsHeld (expiresAt = now + 10 min)
  → message différé sur `delayed`
  → à T+10min : ReleaseExpiredHolds
      si PaymentConfirmed reçu entre temps → no-op (idempotent)
      sinon → SeatsReleased → projection → Mercure → sièges reverdis
```

**Tout handler est idempotent.** AMQP garantit at-least-once. Un `ReleaseExpiredHolds`
rejoué sur un hold déjà converti ne casse rien.

---

## Concurrence — le problème dur

Le double booking est **la** invariante à protéger. Trois défenses, toutes obligatoires :

1. **Unique index partiel** sur `(performance_id, seat_id)` pour les statuts actifs.
   Le filet ultime. S'il saute, tout le reste est cassé.
2. **Lock optimiste explicite** — `UPDATE … WHERE version = :expected`, vérifier `rowCount()`.
3. **Retry sur `ConcurrencyException`** dans le middleware Messenger : 3 tentatives, backoff.
   Au-delà → `SeatsUnavailable` renvoyé au client.

Jamais de lock pessimiste sur l'inventaire complet : ça sérialise toute la salle.

**L'agrégat est la section, pas la performance.** Un `SeatInventory` par
`(performanceId, sectionId)`. Sinon 2000 personnes se battent pour un agrégat unique et
tu as réinventé le mutex global.

Test obligatoire : N holds concurrents sur le même siège, exactement un réussit.

---

## Scalping — le problème métier qu'on ne peut pas ignorer

C'est *le* sujet réel de la billetterie. Un jury qui connaît le secteur le demandera.

Minimum viable, à documenter explicitement même si l'implémentation est simple :

- rate limit par IP et par compte sur `HoldSeats`
- plafond de sièges par commande, porté par la `Performance` (invariante de domaine, pas config)
- captcha au-delà d'un seuil de tentatives
- `HoldRejected` tracé, avec le motif

Le plafond par commande appartient au domaine `Ticketing` — c'est une règle métier, pas
une protection technique. Le rate limit appartient à l'infrastructure.

---

## Dégradation

| Panne | Comportement attendu |
|---|---|
| Mercure indisponible | le front bascule en polling sur `/availability`, backoff exponentiel |
| RabbitMQ indisponible | les commandes échouent proprement (503), pas de perte silencieuse |
| Worker en retard | les holds expirent quand même (le TTL est en base, pas dans le message) |
| Projection en retard | l'écran est en retard, jamais faux — la source de vérité reste l'agrégat |

Le point important : **le TTL du hold vit en base**, pas seulement dans le message différé.
Si le worker est mort, un hold expiré reste expiré du point de vue de l'invariante.

---

## Tests

| Type | Cible | Contrainte |
|---|---|---|
| Unitaires domaine | agrégats, VO, invariantes | aucun I/O, aucun mock de framework, < 1 ms |
| Application | handlers, ports en fake | fakes à la main, **pas de Mockery** |
| Intégration | repositories DBAL, projections, concurrence | Postgres réel dans Docker |
| Golden | cohérence géométrique PHP/TS | PHPUnit + Vitest (voir `GEOMETRY.md`) |
| E2E | parcours d'achat | Playwright, happy path uniquement |

**Fakes > mocks** dans le domaine et l'application. Un `InMemorySeatInventoryRepository`
est plus lisible et plus solide qu'une pile de `->expects()->willReturn()`.

Les deux tests qui comptent le plus :

1. holds concurrents sur le même siège → exactement un gagnant
2. saga d'expiration → avec un `FrozenClock`, **jamais de `sleep()`**

---

## CI/CD

```
PR:    qa (phpstan max, cs-fixer, deptrac, eslint, tsc) → test-unit → test-integration → build
main:  ↑ + build images → push GHCR → deploy SSH → smoke test
```

- Images buildées en CI, **jamais sur le VPS**. Le VPS fait `docker compose pull && up -d`.
- Tag = SHA du commit. `latest` en plus, jamais seul.
- **Migrations dans un job séparé**, avant le démarrage de l'app. Jamais dans l'entrypoint :
  deux replicas migreraient en parallèle.
- Healthchecks partout. Traefik ne route que vers du healthy.
- Rollback = redéployer le SHA précédent, en une commande.

### Contraintes VPS

Postgres, RabbitMQ (~200 Mo) et Mercure sont les gros consommateurs de RAM.
**Limites mémoire explicites dans le compose, sur chaque service.** Pas de container superflu.

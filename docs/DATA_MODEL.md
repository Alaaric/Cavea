# Modèle de données — étape 1

MCD puis MLD, limités au périmètre de l'étape 1 (`ROADMAP.md`) : le spec `grid`,
une `Performance`, l'inventaire, le hold, la projection de disponibilité.
`Ordering` et `Notification` sont hors périmètre — aucune commande ni paiement
à l'étape 1, donc rien à modéliser là-bas pour l'instant.

Les décisions structurantes qui rendent ce schéma possible sont dans
`DECISIONS.md` § ADR-010. Ce fichier ne réexplique pas le *pourquoi*, seulement
le *quoi*.

**Rappel de la règle qui gouverne tout ce document** : un schéma Postgres par
bounded context, **zéro clé étrangère entre schémas**. Une flèche qui semble
traverser `venue_catalog` → `ticketing` sur le papier est en réalité une valeur
copiée à la matérialisation, jamais une contrainte `REFERENCES`.

---

## Schéma `venue_catalog`

### MCD

**Entité**

| VENUE_SPEC | |
|---|---|
| **id** | UUID |
| venue_name | texte |
| timezone | texte (IANA, ex. `Europe/Paris`) |
| version | entier |
| spec | JSONB — le spec paramétrique, voir `GEOMETRY.md` |
| published_at | timestamptz |

Aucune association : à l'étape 1, `VenueCatalog` ne connaît qu'un seul objet.
**Immuable** — une correction du spec insère une nouvelle ligne (`version + 1`),
ne met jamais à jour une ligne existante (ADR-004).

### MLD

```sql
CREATE TABLE venue_catalog.venue_spec (
    id            uuid PRIMARY KEY,
    venue_name    text NOT NULL,
    timezone      text NOT NULL,
    version       integer NOT NULL,
    spec          jsonb NOT NULL,
    published_at  timestamptz NOT NULL,
    UNIQUE (venue_name, version)
);
```

---

## Schéma `ticketing`

### MCD

**Entités**

| PERFORMANCE | |
|---|---|
| **id** | UUID (v7, généré côté client) |
| title | texte |
| venue_spec_id | UUID — valeur copiée depuis `venue_catalog.venue_spec.id`, pas une FK |
| venue_spec_version | entier — copiée au même moment |
| starts_at | timestamptz (UTC) |
| seat_cap_per_order | entier — plafond anti-scalping (ADR, `ARCHITECTURE.md` § Scalping) |
| created_at | timestamptz |

| SECTION *(matérialisée par performance)* | |
|---|---|
| **performance_id + section_id** | composite |
| label | texte |
| price_cents | entier — snapshot du tarif, jamais recalculé après coup |

| SEAT_INVENTORY | |
|---|---|
| **performance_id + section_id + row_label + seat_number** | composite |
| seat_index | entier, 0..N-1, ordre du golden fixture |
| status | énum `free` / `held` / `sold` — dénormalisé, lecture rapide |
| version | entier — verrou optimiste |

| HOLD | |
|---|---|
| **id** | UUID (v7) |
| performance_id | — |
| expires_at | timestamptz |
| idempotency_key | UUID, unique |
| status | énum `active` / `expired` / `converted` |
| created_at | timestamptz |

| HOLD_SEAT *(association porteuse d'attribut, append-only)* | |
|---|---|
| **hold_id + performance_id + section_id + row_label + seat_number** | composite |
| released_at | timestamptz, nullable |

**Associations et cardinalités**

```
PERFORMANCE (1,1) ──< CONTIENT >── (0,n) SECTION
SECTION     (1,1) ──< DÉCLINE  >── (0,n) SEAT_INVENTORY
PERFORMANCE (1,1) ──< OUVRE    >── (0,n) HOLD
HOLD        (1,1) ──< RÉCLAME  >── (1,n) HOLD_SEAT
SEAT_INVENTORY (0,1) ──< RÉCLAMÉ PAR >── (0,n) HOLD_SEAT
```

- Un `HOLD` réclame **au moins un** siège (1,n côté `HOLD_SEAT`) — un hold vide n'a pas
  de sens.
- Un `SEAT_INVENTORY` peut apparaître dans **plusieurs** `HOLD_SEAT` au fil du temps
  (0,n) : un siège tenu, libéré, retenu plus tard par quelqu'un d'autre — c'est
  l'historique append-only qui porte cette répétition, pas `seat_inventory` lui-même.
  Côté « en ce moment », la cardinalité réelle est (0,1) : un siège n'a jamais plus
  d'un `HOLD_SEAT` actif — c'est exactement ce que l'index unique partiel impose.

### MLD

```sql
CREATE TABLE ticketing.performance (
    id                  uuid PRIMARY KEY,
    title               text NOT NULL,
    venue_spec_id       uuid NOT NULL,       -- copié, pas de FK cross-schéma
    venue_spec_version  integer NOT NULL,
    starts_at           timestamptz NOT NULL,
    seat_cap_per_order  integer NOT NULL,
    created_at          timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE ticketing.performance_section (
    performance_id  uuid NOT NULL REFERENCES ticketing.performance(id),
    section_id      text NOT NULL,
    label           text NOT NULL,
    price_cents     integer NOT NULL,
    PRIMARY KEY (performance_id, section_id)
);

CREATE TABLE ticketing.seat_inventory (
    performance_id  uuid NOT NULL,
    section_id      text NOT NULL,
    row_label       text NOT NULL,
    seat_number     integer NOT NULL,
    seat_index      integer NOT NULL,
    status          text NOT NULL DEFAULT 'free'
                        CHECK (status IN ('free', 'held', 'sold')),
    version         integer NOT NULL DEFAULT 0,
    PRIMARY KEY (performance_id, section_id, row_label, seat_number),
    FOREIGN KEY (performance_id, section_id)
        REFERENCES ticketing.performance_section(performance_id, section_id),
    UNIQUE (performance_id, seat_index)
);

CREATE TABLE ticketing.hold (
    id                uuid PRIMARY KEY,
    performance_id    uuid NOT NULL REFERENCES ticketing.performance(id),
    expires_at        timestamptz NOT NULL,
    idempotency_key   uuid NOT NULL UNIQUE,
    status            text NOT NULL DEFAULT 'active'
                          CHECK (status IN ('active', 'expired', 'converted')),
    created_at        timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE ticketing.hold_seat (
    hold_id         uuid NOT NULL REFERENCES ticketing.hold(id),
    performance_id  uuid NOT NULL,
    section_id      text NOT NULL,
    row_label       text NOT NULL,
    seat_number     integer NOT NULL,
    released_at     timestamptz,
    PRIMARY KEY (hold_id, performance_id, section_id, row_label, seat_number),
    FOREIGN KEY (performance_id, section_id, row_label, seat_number)
        REFERENCES ticketing.seat_inventory(performance_id, section_id, row_label, seat_number)
);

-- Le filet ultime contre le double hold (ADR-010 § 4) : un siège n'a jamais
-- plus d'une ligne active à un instant donné, quel que soit l'état du
-- contrôle de version applicatif.
CREATE UNIQUE INDEX hold_seat_active_uniq
    ON ticketing.hold_seat (performance_id, section_id, row_label, seat_number)
    WHERE released_at IS NULL;
```

### Projection — hors MCD

`seat_availability` est un read model (CQRS ciblé, ADR-006), pas une entité
conceptuelle : elle ne porte aucune règle métier, seulement un cache que le
projecteur réécrit. Elle n'apparaît donc pas dans le MCD ci-dessus.

```sql
CREATE TABLE ticketing.seat_availability_projection (
    performance_id  uuid PRIMARY KEY REFERENCES ticketing.performance(id),
    buffer          bytea NOT NULL,   -- 1 octet par siège, indexé par seat_index
    updated_at      timestamptz NOT NULL
);
```

Le projecteur, sur `SeatsHeld` / `SeatsReleased` : localise l'octet via
`seat_index`, le modifie, réécrit `buffer` en entier (un seul `UPDATE`, pas de
`SELECT` intermédiaire sur `seat_inventory`).

---

## Ce qui n'est délibérément pas là

- `Show`, `Order`, `Ticket`, `Allocation`, paiement — étapes ultérieures ou hors
  roadmap actuelle (`ROADMAP.md` § Hors roadmap). Les ajouter maintenant serait
  modéliser au-delà du besoin (ADR-006).
- Toute colonne x/y/z — la géométrie ne vit jamais en base (ADR-004,
  `GEOMETRY.md`).

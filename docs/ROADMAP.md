# Roadmap

L'ordre est délibéré : **la boucle métier complète avant tout visuel.**
Justification dans `DECISIONS.md` § ADR-008.

**Étape actuelle : 1.**

---

## Definition of done

Une étape n'est pas finie parce que « ça compile ». Elle est finie quand :

- [ ] `make qa` passe (phpstan max, cs-fixer, deptrac, eslint, tsc)
- [ ] `make test` passe, y compris les tests d'intégration
- [ ] les tests spécifiques à l'étape existent et passent
- [ ] c'est **déployé sur le VPS**, healthcheck vert
- [ ] le parcours se fait à la main, dans un navigateur, sur l'instance déployée
- [ ] les décisions non triviales sont dans `DECISIONS.md`

**Ne pas passer à l'étape suivante avant que toutes les cases soient cochées.**
L'objectif de cette discipline : ne jamais se retrouver avec une belle salle 3D et pas de backend.

---

## Étape 1 — La boucle [cœur]

Une section rectangulaire, 200 sièges, plat. **Rendu SVG 2D, zéro Three.js.**

Objectif : faire tourner la boucle complète.

```
POST /performances/{id}/holds
  → HoldSeats → SeatInventory → SeatsHeld
  → RabbitMQ → worker → projection
  → Mercure → SSE → le SVG se recolore
```

C'est ici que sont les vraies difficultés. Le reste du projet est du décor par-dessus.

**Contenu**

- `VenueCatalog` : spec minimal (une section `grid`), `SeatLayoutGenerator` PHP
- `Ticketing` : `Performance`, `SeatInventory`, `HoldSeats`, `SeatsHeld`, `SeatsReleased`
- Repository DBAL à la main, lock optimiste explicite, migrations à la main
- `DbalTransactionMiddleware`
- Saga d'expiration sur `delayed`
- Projection `seat_availability` + endpoint `/availability` en `Uint8Array`
- Mercure + `EventSource` côté front, fallback polling
- SVG cliquable, recoloration sur delta

**Tests obligatoires de l'étape**

- [ ] N holds concurrents sur le même siège → exactement un gagnant
- [ ] saga d'expiration avec `FrozenClock`, jamais de `sleep()`
- [ ] `ReleaseExpiredHolds` rejoué sur un hold converti → no-op
- [ ] golden fixture PHP + TS sur le spec `grid`
- [ ] deptrac : aucune violation de la règle de dépendance

**Le piège de l'étape** : les migrations à la main. C'est la seule vraie perte de confort du
choix DBAL, et elle se sent tout de suite. Ne pas céder et réintroduire l'ORM.

---

## Étape 2 — InstancedMesh

Remplacer le SVG par un `InstancedMesh` de cubes. **Même data, même flux.**

- Un seul `InstancedMesh`, `BoxGeometry` suffit
- `setColorAt` sur delta, jamais de reconstruction
- Picking par `instanceId` → `Array<SeatId>`
- `dispose()` au démontage
- Version Three épinglée à l'exact

Si l'étape 2 demande de toucher au backend, c'est que l'étape 1 était mal découpée.

---

## Étape 3 — La vraie géométrie

- Sections en `arc`
- `rake` linéaire
- Plusieurs sections, plusieurs `tier`
- Golden fixture étendu au spec complet

C'est ici que la numérotation par rangée (`n_r` variable) devient réelle.

---

## Étape 4 — Vue depuis le siège

La feature signature.

- Tween caméra vers `pos + (0, 1.2, 0)`
- `lookAt` centre scène, ajustement FOV, 3–4 s

Doit être irréprochable : c'est ce qu'on retiendra du projet.

---

## Étape 5 — Polish

- LOD
- Hover (mesh de surbrillance séparé)
- Labels de sections
- Vraie géométrie de siège low-poly
- C-value sur le rake (courbe concave au lieu de linéaire)

---

## Hors roadmap, à décider

Ces sujets sont identifiés mais pas planifiés. Ne pas les commencer sans arbitrage explicite.

- **Scalping** — le problème métier réel de la billetterie (cf. `ARCHITECTURE.md`).
  Un jury du secteur le demandera. Minimum : rate limit, plafond par commande porté par la
  `Performance`, captcha au seuil. Le plafond appartient au domaine, le rate limit à l'infra.
- **Paiement réel** — Stripe est mocké derrière le port `PaymentGateway`. Le brancher pour de
  vrai n'apporte rien à la démonstration architecturale.
- **Event sourcing sur `Ticketing`** — tentant, mais le CQRS + projections suffit à démontrer
  le propos. À ne faire que si un besoin réel apparaît (audit, replay).
- **Node 24 → 26** — échéance octobre 2026, quand 24 passe en maintenance. Toolchain de build
  uniquement, risque nul.

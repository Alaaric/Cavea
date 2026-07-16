# Géométrie

Le cœur technique du projet : générer une salle crédible à partir de ~2 Ko de JSON,
identiquement en PHP et en TypeScript.

---

## Principe fondateur

**Les coordonnées ne sont pas de la donnée métier.**

- `VenueCatalog` stocke un **spec paramétrique** (JSONB, ~2 Ko pour 2000 places).
- `Ticketing` matérialise l'inventaire : une ligne par siège, avec
  `(sectionId, rowLabel, seatNumber)` et un statut. **Aucun x/y/z en base. Jamais.**
- Le frontend dérive les positions du même spec.

Le domaine connaît l'identité et l'état d'un siège, pas sa position dans l'espace.
La géométrie est un détail de présentation : elle vit côté frontend.

---

## Le spec

```json
{
  "venueId": "olympia",
  "stage": { "width": 18, "depth": 8 },
  "sections": [
    {
      "id": "ORCH-C",
      "label": "Orchestre Centre",
      "tier": 0,
      "shape": "arc",
      "innerRadius": 12,
      "rowCount": 18,
      "rowDepth": 0.95,
      "seatWidth": 0.55,
      "angleStart": -0.6,
      "angleEnd": 0.6,
      "baseElevation": 0,
      "rake": { "type": "linear", "rise": 0.12 }
    }
  ]
}
```

Unités : mètres, radians. Le spec est **immuable** une fois publié — une correction crée
une nouvelle version de `Venue`, elle ne modifie pas les performances déjà ouvertes.

---

## L'algorithme

Scène à l'origine, public en **+Z**, θ mesuré depuis l'axe **+Z**.

Pour chaque rangée `r` d'une section en arc :

```
R_r   = innerRadius + r × rowDepth        // rayon
y_r   = baseElevation + r × rise          // élévation (le gradin)
arc_r = R_r × (angleEnd − angleStart)     // longueur d'arc
n_r   = floor(arc_r / seatWidth)          // nb de sièges sur cette rangée
```

Pour le siège `s` de la rangée `r`, centré dans l'arc :

```
offset = (arc_r − n_r × seatWidth) / 2 / R_r
θ_s    = angleStart + offset + (s + 0.5) × seatWidth / R_r
pos    = (R_r·sin θ_s, y_r, R_r·cos θ_s)
rotY   = θ_s + π                          // le siège regarde la scène
```

Le détail qui rend le résultat crédible : le rayon croît avec les rangées, donc `n_r` croît
aussi — les rangées ont naturellement des longueurs différentes, comme dans une vraie salle.
Et ça donne gratuitement un vrai problème de domaine : la numérotation est **par rangée**,
pas globale.

### Le rake

`linear` suffit largement pour commencer.

Extension possible (étape 5) : la vraie formule de sightline des architectes, dite
**C-value** — le dégagement vertical entre l'œil d'un spectateur et celui de la rangée
devant, typiquement 90–120 mm. Elle donne une courbe concave au lieu d'une droite.
~10 lignes de plus, profil de gradin visiblement différent.

---

## Le contrat back / front

Implémenté deux fois :

| | Fichier | Rôle |
|---|---|---|
| PHP | `VenueCatalog\Domain\Service\SeatLayoutGenerator` | matérialiser l'inventaire (IDs seulement) |
| TS | `frontend/src/venue/layout.ts` | rendu (IDs + positions) |

Le PHP n'a besoin **que de l'ordre et des identités**. Il ne calcule jamais de position.
Le TS calcule les deux.

---

## Golden fixture — le garde-fou

### Le risque

**C'est le risque principal du projet.** Si PHP et TS parcourent le spec dans un ordre
différent, les identités de sièges divergent silencieusement. Un client clique A3 et
achète F12. Invisible en dev, catastrophique en prod, et aucun test classique ne l'attrape.

### La parade

```
tests/fixtures/golden/
  venue-spec.json        # spec de référence — NE JAMAIS MODIFIER
  seat-ids.txt           # liste ordonnée attendue
  seat-ids.sha256        # hash
```

Deux tests assèrent le même hash :

- `tests/Unit/VenueCatalog/GoldenLayoutTest.php` (PHPUnit)
- `frontend/src/venue/layout.golden.test.ts` (Vitest)

Les deux tournent en CI. **Si tu touches à l'algo d'un seul côté, la CI hurle.**

### La règle

**Ne jamais régénérer le golden pour faire passer un test.** Si le hash change, soit
quelque chose est cassé, soit le changement est intentionnel — et alors il se discute
dans la PR, avec le nouveau hash committé délibérément et justifié dans le message.

Régénérer le golden pour faire verdir la CI annule exactement la protection qu'il apporte.

### Ce qui n'est pas hashé

Les **positions** (flottants). Le déterminisme des flottants cross-langage n'est pas garanti
et n'a aucune importance ici : seul l'ordre des identités doit être partagé.

---

## Rendu Three.js

### InstancedMesh, une seule fois

**Un seul `InstancedMesh` pour tous les sièges.** 20 000 sièges = 1 draw call.
Jamais un mesh par siège, sous aucun prétexte.

Géométrie low-poly. Au début, un `BoxGeometry` suffit — vu de loin, ça passe très bien.

### État → couleur

```js
mesh.setColorAt(i, color);
mesh.instanceColor.needsUpdate = true;
```

Libre / en cours de réservation / vendu / sélectionné. **Jamais de reconstruction de
géométrie sur changement d'état** — juste un buffer qui change.

### Picking

`raycaster.intersectObject(mesh)` → `instanceId` → lookup dans un `Array<SeatId>`
construit à la génération, indexé par `instanceId`. O(1).

Sur 20 000 instances le raycast reste sous la milliseconde : **pas de GPU picking**, inutile.

### Hover

Mesh de surbrillance séparé qu'on repositionne. Plus simple qu'un attribut par instance.

### Hygiène

- `dispose()` systématique des geometries et materials au démontage. Le leak Three.js en SPA
  est réel et se voit après trois navigations.
- **Version épinglée à l'exact.** Three casse entre mineures. Jamais de `^`.

---

## Flux de données

```
GET  /venues/{id}/layout            → spec, immuable, ETag + max-age long
GET  /performances/{id}/availability → Uint8Array, 1 octet/siège, indexé comme les sièges
SSE  /.well-known/mercure            → deltas { seatId, status }
POST /performances/{id}/holds        → HoldSeats
```

**Availability en binaire, pas en JSON** : 20 000 sièges = 20 Ko au lieu de ~1,5 Mo.
Puis Mercure ne pousse que les deltas, et le front fait `setColorAt` sur les instances
concernées.

Côté React : `new EventSource(url)`. Trois lignes.

**Si Mercure tombe** : bascule en polling sur `/availability` avec backoff exponentiel.
Jamais d'écran cassé.

---

## Vue depuis le siège

La feature signature. Une fois les positions calculées, c'est quasi gratuit :

```
clic → tween caméra vers pos + (0, 1.2, 0)   // hauteur d'œil assis
     → lookAt(centre de la scène)
     → ajustement du FOV
     → 3–4 s
```

En quelques secondes, le client voit littéralement ce qu'il achète.
C'est ce qu'on retiendra du projet — elle doit être irréprochable.

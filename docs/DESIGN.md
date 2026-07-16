# Cavea — Direction artistique

Billetterie de spectacle. L'utilisateur choisit une représentation, explore la salle en 3D,
clique un siège, voit la vue depuis ce siège, réserve.

**Le job de l'interface : disparaître.** Le seul moment fort est la salle. Tout le reste
conduit à elle ou s'efface devant elle.

- Interface en **français**
- **Desktop d'abord.** Sur mobile, pas de 3D : une liste de sièges qui doit être bonne,
  pas une excuse.

---

## La thèse

Direction demandée : théâtre nocturne — sombre, velours, or, cinématique.

**Le piège** : « velours bordeaux + filigrane doré + Didone » est le pastiche que fait
chaque site d'opéra depuis 2003. Sur fond noir avec un accent doré, c'est aussi exactement
ce que produit un générateur quand on ne lui donne pas d'idée.

**Le recadrage** : une salle de théâtre la nuit, ce n'est pas de la décoration dorée.
C'est **du noir, et une seule chose éclairée**. Quand la salle s'éteint, les dorures
disparaissent — on ne les voit plus. Il reste le noir et un rectangle de lumière.

Donc :

- **L'or n'est pas une couleur, c'est de la lumière.** Il n'apparaît que là où la lumière
  tombe. Jamais en bordure, jamais en filet, jamais en aplat, jamais en ornement.
- **Le velours n'est pas du bordeaux.** Le velours est un *comportement de la lumière* :
  il l'absorbe. C'est le noir le plus profond de la salle, teinté chaud, avec un souffle
  ténu en lumière rasante. Le velours, ce sont les quasi-noirs — pas un accent rouge.
- **La salle est noire. Le plateau est allumé.**

Cette thèse n'est pas décorative : elle *est* l'exigence produit. La salle 3D est le seul
moment fort, tout le reste s'efface. Même principe. C'est ce qui fait que la direction
mérite d'être prise.

### Ce qu'on ne fait pas

- ❌ Aplats bordeaux ou pourpre
- ❌ Filets, cadres, séparateurs dorés
- ❌ Textures de velours, motifs, filigranes, ornements
- ❌ Rideau de scène en illustration
- ❌ Masques de théâtre, projecteurs stylisés, pellicule
- ❌ Dégradés dorés
- ❌ Sérif à contraste élevé partout — **une seule occurrence par écran** (voir Type)

---

## Couleur

```css
--house:       #0A0807;  /* la salle. Noir chaud, jamais bleuté. Fond par défaut. */
--velvet:      #17100E;  /* surface d'un cran. Panneaux, cartes. */
--velvet-lit:  #241814;  /* velours en lumière rasante. Bordures, surfaces surélevées. */
--limelight:   #F2E4C8;  /* la lumière. Crème chaud. Texte principal. */
--ash:         #6B615A;  /* éteint, en retrait. Texte secondaire, libellés inertes. */
--gilt:        #C9962C;  /* l'or = émission uniquement. Voir la règle ci-dessous. */
--ember:       #B4442E;  /* erreur et destructif uniquement. Jamais ailleurs. */
```

**Règle de l'or.** `--gilt` a un budget : **trois occurrences maximum par écran**.
Il est réservé à ce qui émet : le siège sélectionné, le compte à rebours du hold, l'anneau
de focus, l'unique action principale. S'il apparaît une quatrième fois, en retirer une.

**Pas de bleu.** Aucune part de cette palette n'est froide. Un gris bleuté de dark mode
générique casserait la thèse en une seule occurrence.

---

## L'échelle de lumière des sièges

C'est le cœur fonctionnel du design, et le seul endroit où la couleur porte du sens.

**Principe : l'état d'un siège est porté par la luminance, pas par la teinte.**
Les sièges disponibles sont éclairés. Les sièges vendus sont retournés au noir de la salle.

Trois bénéfices, et c'est pour ça que ce n'est pas négociable :

1. **Accessible par construction.** La luminance est le seul canal que tout le monde
   partage. Un daltonien lit cette carte sans aide. Le rouge/vert conventionnel est
   précisément la paire hostile.
2. **C'est la métaphore, littéralement.** La lumière tombe sur ce qui est libre.
3. **C'est le bon comportement de scan.** On cherche ce qu'on peut acheter : ça doit briller.

```css
--seat-sold:      #1A1411;  /* rendu à la salle. Non interactif, non focusable. */
--seat-held:      #4E443C;  /* bloqué par quelqu'un d'autre. Visible, éteint. */
--seat-free:      #9C8A6B;  /* éclairé. */
--seat-hover:     #BCA87F;  /* un cran de plus. */
--seat-selected:  #F3C64B;  /* le plus brillant de l'écran + anneau + halo. */
```

### Règles

- **Contraste ≥ 3:1 entre deux états adjacents de l'échelle.** À vérifier au contrastomètre,
  pas à l'œil. Si un ajustement casse le ratio, c'est l'esthétique qui cède.
- **La teinte ne porte jamais seule.** Elle renforce, elle n'informe pas.
- `--seat-selected` se distingue de `--seat-hover` par **trois** canaux : luminance, teinte,
  et un anneau. Jamais par la couleur seule.
- **Légende obligatoire**, visible, pas dans un tooltip. On inverse une convention forte
  (rouge/vert) : ça se paie en explicitation.
- **État en toutes lettres au survol et au focus** : « Rang F, siège 12 — libre — 45,00 € ».
- `--seat-sold` sort du parcours clavier. On ne tabule pas sur ce qu'on ne peut pas acheter.

---

## Type

L'intuition dit « Didone partout, c'est le théâtre ». Non. Le vrai sujet est ailleurs :
**une billetterie est une interface de chiffres** déguisée en soirée. Rangs, numéros de
siège, horaires, prix, compte à rebours. La signature typographique, ce sont les chiffres.

| Rôle | Police | Usage |
|---|---|---|
| Display | **Bodoni Moda** | **Une seule occurrence par écran** : le titre de la représentation. Grand, tracking serré, en `--limelight`. **Jamais en or.** Jamais dans un bouton, un libellé, un menu. |
| Corps | **Geist Sans** | Tout le reste. Neutre, précis, s'efface. |
| Données | **Geist Mono** | Rangs, numéros de siège, prix, horaires, compte à rebours. Chiffres tabulaires obligatoires. |

Substitutions autorisées si les critères tiennent : display = Didone à contraste élevé ;
corps = grotesque neutre à chiffres tabulaires ; données = mono à chiffres lisibles.

### Échelle

```
display    clamp(3rem, 6vw, 5.5rem)  / 0.95 / tracking -0.02em
h1         1.75rem / 1.2
h2         1.25rem / 1.3
corps      0.9375rem / 1.6
data       0.875rem / 1.4   tabular-nums
libellé    0.75rem / 1.3    tracking 0.08em, capitales, --ash
```

Un seul niveau de libellé en capitales espacées. Deux, c'est du costume.

---

## Layout

La salle est **quasi plein cadre sur le noir**. Le chrome est une fine couche qui flotte
par-dessus, sans cadre, sans panneau opaque qui viendrait rogner la scène.

```
┌────────────────────────────────────────────────────────┐
│  Cavea            Hamlet · jeu. 12 mars, 20h30         │  ← barre fine, --house, sans bordure
├────────────────────────────────────────────────────────┤
│                                                        │
│                    [ LA SCÈNE ]                        │
│                                                        │
│               · · · · · · · · · ·                      │
│             · · · · · · · · · · · ·                    │  ← la salle, plein cadre
│           · · · · · · · · · · · · · ·                  │
│                                                        │
│  ┌──────────┐                          ┌────────────┐  │
│  │ légende  │                          │ 2 places   │  │  ← flottants, --velvet à 92%,
│  └──────────┘                          │ 90,00 €    │  │     jamais opaques
│                                        │ 09:47      │  │
│                                        │ [Réserver] │  │
│                                        └────────────┘  │
└────────────────────────────────────────────────────────┘
```

- Pas de sidebar. Pas de cadre autour du canvas. La salle touche les bords.
- Les panneaux flottants sont en `--velvet` à ~92 % d'opacité avec un léger flou : la salle
  transparaît. Rien n'est opaque devant la scène.
- Rayons de bordure : 2px. Presque rien. On n'est pas dans une app SaaS.
- Bordures : `--velvet-lit`, 1px, et seulement quand elles séparent vraiment quelque chose.

---

## Signature et mouvement

### Le noir se fait

En entrant sur la carte des sièges : la page passe de l'ambiance (info représentation,
chaud, dimmé) au noir, puis la salle se révèle — les sièges s'allument **en vague depuis la
scène vers le fond**, ~800 ms, easing lourd.

Une fois par session, pas à chaque navigation. Personne ne veut attendre le théâtre trois
fois de suite.

### La vue depuis le siège — le paiement de la promesse

Au clic sur un siège : **tout le chrome disparaît en fondu**. Barre, panneaux, légende.
Noir total, et la vue. La caméra descend vers le siège en 3–4 s, easing lent et pesé.
Le chrome revient quand la caméra s'immobilise.

C'est le seul moment long de l'interface, et il est mérité : l'utilisateur l'a demandé.

### Le reste

Rien ne rebondit. Rien ne ressort. Easing lourd, sortie lente — le poids d'un rideau, pas
d'un ressort. Durées : 120 ms pour un survol, 240 ms pour une transition d'état, jamais
au-delà sauf les deux moments ci-dessus.

**Une seule chose pulse sur tout le site : le compte à rebours du hold.** C'est la seule
tension réelle du parcours. Elle mérite d'être ressentie. Rien d'autre n'a le droit de bouger
tout seul.

`prefers-reduced-motion` : la vague devient un fondu, le tween caméra devient une coupe.
Le parcours ne perd aucune information.

---

## Mobile — la liste, pas l'excuse

Pas de 3D. Ce n'est pas une dégradation à cacher, c'est une vue à part entière.

- Sections en groupes repliables, avec le nombre de places restantes et la fourchette de prix
- Chaque rang = une bande horizontale de pastilles de sièges
- **La même échelle de lumière s'applique.** Un siège vendu est sombre ici aussi.
- Cible tactile 44px minimum, y compris pour les pastilles
- La vue depuis le siège reste disponible : une image statique pré-rendue, pas de canvas

Ne jamais écrire « fonctionnalité indisponible sur mobile ». La liste est la fonctionnalité.

---

## Écriture

Français. Casse de phrase. Voix active. Verbes simples. Le vocabulaire ne change pas entre
le bouton et sa confirmation.

| Ne pas écrire | Écrire |
|---|---|
| Valider | Réserver ces places |
| Soumettre | Payer 90,00 € |
| Erreur : siège indisponible | Ce siège vient d'être pris. Choisissez-en un autre. |
| Aucune sélection | Cliquez un siège éclairé pour commencer. |
| Session expirée | Vos places ont été libérées. Elles sont peut-être encore disponibles. |

- Les erreurs ne s'excusent pas et ne sont jamais vagues. Elles disent ce qui s'est passé
  et quoi faire.
- Un écran vide est une invitation, pas un constat.

### Typographie française — non négociable

- Espace insécable avant `: ; ! ?` et à l'intérieur des guillemets `« … »`
- Prix : `45,00 €` — virgule décimale, espace insécable avant l'euro
- Heures : format 24h, `20h30`
- Dates : mois en minuscules — `jeu. 12 mars`
- Capitales accentuées : `À`, `É`, `Ê`

Le compte à rebours en `mm:ss`, chiffres tabulaires, sinon il tremble à chaque seconde.

---

## Plancher de qualité

- Focus visible partout : anneau `--gilt` 2px, offset 2px. C'est un des trois usages de l'or.
- Contraste texte ≥ 4.5:1, échelle des sièges ≥ 3:1 entre états adjacents
- Navigable au clavier, y compris la carte : flèches entre sièges, Entrée pour sélectionner
- `prefers-reduced-motion` respecté
- Les sièges vendus ne sont pas focusables
- Rien d'important n'est porté par la couleur seule

---

## Écrans à produire

1. **Accueil** — les représentations à venir. Sobre, dense, sans hero marketing.
   Le display sert ici, une fois, sur le spectacle mis en avant.
2. **Fiche représentation** — infos, tarifs par section, l'action qui mène à la salle.
3. **La salle** — l'écran principal. C'est là que va tout l'effort.
4. **Vue depuis le siège** — plein écran, sans chrome.
5. **Panier / paiement** — calme, contrasté, le compte à rebours toujours visible.
6. **Confirmation** — le billet. Le seul endroit où l'or peut respirer un peu.
7. **Mobile : la liste de sièges** — traitée comme un écran à part entière.

# TPG Departures — PWA temps réel

Progressive Web App affichant les prochains départs TPG pour des arrêts configurés, avec rafraîchissement automatique. Conçue pour être ajoutée à l'écran d'accueil iPhone comme une app native.

## Fonctionnement

L'app interroge l'API publique [transport.opendata.ch](https://transport.opendata.ch) (gratuite, sans authentification, CORS activé) et affiche les prochains départs filtrés par ligne et par direction pour chaque arrêt configuré.

### Fichiers

| Fichier | Rôle |
|---------|------|
| `index.html` | App complète (HTML + CSS + JS embarqués) |
| `manifest.json` | Manifeste PWA pour le mode standalone iOS/Android |
| `email-visual-identity.md` | Charte graphique e-butler (référence pour le design) |

### Arrêts configurés

| Arrêt | ID DIDOK | Lignes | Direction(s) filtrée(s) |
|-------|----------|--------|-------------------------|
| Rue du Lac | `8592900` | 2, 6 | L2: Jonction/Bernex · L6: Vernier |
| Place des Eaux-Vives | `8592883` | 2, 6, 9, 10, 25 | L2: Jonction/Bernex · L6: Vernier · L9: Lignon · L10: toutes · L25: Jardin Botanique |
| CERN | `8587918` | 18 | L18: Palettes |
| Gare Cornavin | `8587057` | 6, 9, 14, 18 | L6: Plage · L9: Belle-Terre · L14: Gravière · L18: CERN |
| Bel-Air | `8587387` | 2, 14, 18 | L2: Plage · L14: Blandonnet · L18: CERN |
| Merle d'Aubigné | `8592854` | 2, 6 | L2: Jonction/Bernex · L6: Vernier |

La configuration se modifie dans l'objet `CONFIG` en haut du bloc `<script>` dans `index.html`.

### API

```
GET https://transport.opendata.ch/v1/stationboard?id={DIDOK_ID}&limit=15
```

Champs utilisés par départ dans `stationboard[]` :
- `number` — numéro de ligne (ex: `"14"`)
- `category` — `"T"` = tram, `"B"` = bus
- `to` — destination (ex: `"Bernex, Vailly"`)
- `stop.departure` — heure prévue (ISO 8601)
- `stop.delay` — retard en minutes (integer)
- `stop.prognosis.departure` — heure réelle prédite

Pas de clé API requise. Limite de débit (rate limit) gérée avec backoff exponentiel.

### Fonctionnalités

- **Filtrage par direction** — chaque ligne peut être filtrée par destination terminus via le champ `directions` dans la config
- **Limite par ligne** — seuls les 2 prochains départs par ligne sont affichés (`maxDeparturesPerLine: 2`)
- **Couleurs TPG officielles** — chaque ligne a sa couleur officielle (source: tpg.ch), affichée dans des badges arrondis
- **Rafraîchissement auto** toutes les 30s (appels API) + re-calcul des countdowns toutes les 15s (sans appel)
- **Gestion visibilité** — pause en arrière-plan, refresh immédiat au retour
- **Résilience** — un arrêt en erreur n'affecte pas les autres, les données précédentes sont conservées
- **Mode sombre** par défaut, respecte `prefers-color-scheme`
- **Safe areas iOS** — compatible notch et Dynamic Island
- **Refresh manuel** via bouton `[reload]` dans le header

### Design

Identité visuelle inspirée de la charte e-butler :

- Police monospace (Courier New / Monaco / Menlo)
- Fond sombre `#1a1a1a`, texte `#e0e0e0`
- Esthétique terminal compacte
- Couleurs d'état : rose `#ff0099` (accent/retard), vert lime `#32cd32` (imminent)

Couleurs des lignes TPG :

| Ligne | Couleur | Hex |
|-------|---------|-----|
| 2 | Jaune-vert | `#D2DB4A` |
| 6 | Bleu | `#008CBE` |
| 9 | Rouge | `#E2001D` |
| 10 | Vert | `#006E3D` |
| 14 | Violet | `#5A1E82` |
| 18 | Magenta | `#B82F89` |
| 25 | Brun | `#A05909` |

## Installation

### Consultation locale

Ouvrir `index.html` dans un navigateur. L'app fonctionne immédiatement.

### Déploiement GitHub Pages

1. Créer un repo GitHub
2. Pousser `index.html` et `manifest.json`
3. Settings > Pages > Source: Deploy from branch > `main` / `/ (root)`
4. L'app est accessible à `https://<user>.github.io/<repo>/`

### Ajout à l'écran d'accueil iPhone

1. Ouvrir l'URL GitHub Pages dans Safari
2. Tap sur l'icône partage (carré avec flèche)
3. "Sur l'écran d'accueil"
4. L'app s'ouvre en mode plein écran sans barre Safari

## Configuration

Modifier l'objet `CONFIG` dans `index.html` :

```javascript
const CONFIG = {
  refreshInterval: 30000,       // intervalle API en ms
  countdownInterval: 15000,      // intervalle re-calcul countdown en ms
  departuresPerStop: 15,         // nombre de départs demandés à l'API
  maxDeparturesPerLine: 2,       // max départs affichés par ligne et par arrêt
  stops: [
    {
      id: "8592900",
      name: "Rue du Lac",
      lines: ["2", "6"],
      directions: {              // filtrage par direction (optionnel)
        "2": ["Genève, Jonction", "Bernex, Cressy"],
        "6": ["Vernier, village"],
      }
    },
    // ...
  ]
};
```

Le champ `directions` est un objet optionnel : clé = numéro de ligne, valeur = tableau de destinations autorisées (correspondance exacte avec le champ `to` de l'API). Si absent ou si une ligne n'a pas d'entrée, tous les départs de cette ligne sont affichés.

Pour trouver l'ID DIDOK d'un arrêt :
```
https://transport.opendata.ch/v1/locations?query=NOM_ARRET&type=station
```

Pour relever les noms de destination exacts d'une ligne :
```
https://transport.opendata.ch/v1/stationboard?id={DIDOK_ID}&limit=30
```

---

## Idées d'amélioration

### Intégration iOS Shortcuts (alternative au navigateur)

L'objectif est de recevoir les prochains départs sans ouvrir le navigateur — idéalement via une notification ou un message dans l'app Shortcuts.

Pistes explorées :

**a) Shortcut appelant l'API directement (le plus viable)**
iOS Shortcuts peut faire des requêtes HTTP nativement via l'action "Obtenir le contenu de l'URL". On pourrait créer un Shortcut qui :
1. Appelle `transport.opendata.ch/v1/stationboard?id=...` pour chaque arrêt
2. Parse le JSON (Shortcuts supporte le parsing JSON natif)
3. Filtre par ligne
4. Affiche le résultat dans une notification ou une alerte

Avantage : pas besoin du fichier HTML, pas besoin de serveur. Le Shortcut est autonome.
Inconvénient : la logique de filtrage et formatage est à refaire dans le langage visuel de Shortcuts, ce qui est laborieux pour 6 arrêts.

**b) Shortcut + Scriptable (app tierce)**
L'app [Scriptable](https://scriptable.app) permet d'exécuter du JavaScript sur iOS. On pourrait :
1. Extraire la logique JS de l'app dans un script Scriptable
2. Créer un widget iOS ou une notification avec les résultats
3. Déclencher via un Shortcut ou une automatisation programmée

Avantage : on réutilise le même code JS, widgets natifs iOS.
Inconvénient : nécessite d'installer l'app Scriptable.

**c) Serveur + Push Notifications**
Un serveur (ex: Cloudflare Worker, gratuit) interroge l'API à intervalles et envoie des push notifications via un service comme Pushover ou ntfy.sh.
Avantage : notifications vraiment passives, pas d'action requise.
Inconvénient : nécessite un composant serveur à maintenir.

La piste (a) est la plus simple à tester en premier. La piste (b) offre le meilleur équilibre entre flexibilité et simplicité.

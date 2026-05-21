# JPG Datamoshing Explorer — Spécifications techniques

## 1. Objectif et contexte

Construire une application web (HTML/CSS/JS, sans framework lourd, vanilla ou avec une dépendance minimale optionnelle) permettant :

1. **D'explorer** la structure interne d'un fichier JPG en visualisant simultanément :
   - sa représentation hexadécimale (octets bruts),
   - son rendu image décodé,
   - la correspondance bidirectionnelle entre les zones d'octets et les zones de pixels.

2. **D'éditer** ces octets en direct pour produire des effets de *datamoshing* (corruptions visuelles contrôlées), avec re-rendu live de l'image après chaque modification.

Cible : artistes numériques, glitch artists, et personnes voulant comprendre concrètement le format JPG. L'outil doit être à la fois pédagogique et créatif.

## 2. Rappels techniques sur le format JPG

Toute personne implémentant cet outil doit comprendre ces éléments. Les détailler dans le code via des commentaires aide les utilisateurs avancés à lire la source.

### 2.1 Structure générale

Un fichier JPG (JFIF/Exif) est une séquence de **segments** identifiés par des **marqueurs** sur 2 octets, commençant tous par `0xFF` suivi d'un octet identifiant.

Marqueurs principaux à reconnaître :

| Marqueur | Hex | Signification | Suivi de longueur ? |
|---|---|---|---|
| SOI | `FFD8` | Start Of Image | non |
| APP0 | `FFE0` | JFIF header | oui (2 octets big-endian) |
| APP1 | `FFE1` | Exif / XMP | oui |
| APP2-APPF | `FFE2`–`FFEF` | métadonnées diverses | oui |
| DQT | `FFDB` | Quantization Table | oui |
| DHT | `FFC4` | Huffman Table | oui |
| SOF0 | `FFC0` | Start Of Frame (baseline DCT) | oui |
| SOF2 | `FFC2` | Start Of Frame (progressive) | oui |
| DRI | `FFDD` | Define Restart Interval | oui |
| SOS | `FFDA` | Start Of Scan (début des données entropy-coded) | oui (header) |
| RST0-RST7 | `FFD0`–`FFD7` | Restart markers (dans le scan data) | non |
| COM | `FFFE` | Commentaire | oui |
| EOI | `FFD9` | End Of Image | non |

**Important** : après le marqueur SOS et son header, on entre dans la zone *entropy-coded data* (scan data). Dans cette zone, tout `0xFF` réel dans le bitstream est suivi d'un `0x00` (« byte stuffing ») pour ne pas être confondu avec un marqueur. Les seuls `0xFFxx` non stuffés autorisés dans le scan data sont les RST markers et le EOI final.

### 2.2 MCU (Minimum Coded Units)

Le scan data encode l'image par blocs appelés **MCU**. Selon le chroma subsampling déclaré dans SOF0 :

- 4:4:4 → MCU = 8×8 pixels
- 4:2:2 → MCU = 16×8 pixels
- 4:2:0 → MCU = 16×16 pixels (le plus courant pour les JPG du web)

Les MCU sont écrites en ordre raster (gauche→droite, haut→bas). Chaque MCU contient des coefficients DCT pour Y, Cb, Cr, encodés en Huffman puis en bitstream. La **taille en octets de chaque MCU est variable** (compression entropique) — un MCU « uniforme » fait quelques bits, un MCU « détaillé » beaucoup plus.

### 2.3 Restart markers

Si DRI est présent avec une valeur N > 0, alors tous les N MCU on insère un RST marker (`FFD0` à `FFD7` cyclique). Ces marqueurs **réinitialisent le prédicteur DC** et permettent une resync du décodeur en cas de corruption antérieure. C'est techniquement la **clé du datamoshing propre** : entre deux RST markers, on peut tout faire — supprimer, remplacer, dupliquer — sans casser le reste de l'image.

Beaucoup d'encodeurs JPG (notamment ceux du web, smartphones) **n'insèrent pas** de RST markers par défaut. L'outil doit gérer ce cas et proposer une option d'injection (ré-encodage via canvas en activant les RST).

### 2.4 Effets de datamoshing classiques

- **Bit/byte flip dans le scan data** → glitchs colorés localisés et propagation horizontale jusqu'au prochain RST (ou jusqu'à la fin si aucun).
- **Swap de deux blocs entre RST markers** → translation/duplication de zones d'image.
- **Suppression de RST markers** → fusion des erreurs de prédiction DC, dérives chromatiques.
- **Corruption de DQT** → effet global sur toute l'image (saturation, postérisation, couleurs altérées).
- **Truncation prématurée** → zone basse en gris (le décodeur ne reçoit pas la fin du scan).
- **Corruption de DHT** → l'image peut devenir indécodable ou produire des chaos très intenses (à utiliser avec parcimonie).

## 3. Architecture générale

### 3.1 Stack

- HTML/CSS/JS vanilla. Pas de build step requis. Le fichier doit pouvoir s'ouvrir directement par double-clic.
- Single-page, single-file de préférence (`index.html` autonome). Si scinder en plusieurs fichiers améliore vraiment la lisibilité, c'est acceptable, mais sans bundler.
- Décodage image : utiliser l'API native `<canvas>` + `Image` du navigateur (le navigateur fait le décodage JPG natif, on n'a pas à le réimplémenter).
- Manipulation binaire : `ArrayBuffer`, `Uint8Array`, `DataView`.
- Web Worker pour le parsing Huffman complet (mode lourd, optionnel — voir §4.3).

### 3.2 Layout

Trois zones principales, organisées en grille redimensionnable (utiliser des split-panes ou simplement du CSS grid avec poignées en JS) :

```
┌───────────────────────────┬───────────────────────────┐
│                           │                           │
│       HEX VIEWER          │      IMAGE PREVIEW        │
│       (gauche)            │      (droite)             │
│                           │                           │
│                           │                           │
├───────────────────────────┴───────────────────────────┤
│              STRUCTURE & TOOLS PANEL                  │
│              (bas, hauteur ajustable)                 │
└───────────────────────────────────────────────────────┘
```

- **Hex viewer** : scrollable, affichage classique `offset | 16 octets hex | ASCII`.
- **Image preview** : canvas affichant le rendu courant, avec overlay pour les highlights.
- **Structure & tools** : panneau avec onglets (Structure / Édition / Mapping / Export).

### 3.3 Modèle de données central

Une seule source de vérité : un `Uint8Array` global représentant le fichier en cours.
Toute modification se fait sur ce buffer, puis déclenche :

1. Re-render du hex viewer (zone visible uniquement),
2. Re-décodage de l'image via `Blob` → `URL.createObjectURL` → `Image` → `canvas`,
3. Re-parsing de la structure (rapide, < 50ms pour un JPG normal),
4. Mise à jour du mapping octets ↔ pixels.

Garder un historique (undo/redo) sous forme de stack de diffs (offset + ancienne valeur + nouvelle valeur), pas de snapshots complets — sinon la mémoire explose vite.

## 4. Fonctionnalités détaillées

### 4.1 Chargement de fichier

- Bouton « Ouvrir » + drag-and-drop sur toute la fenêtre.
- Accepter `.jpg`, `.jpeg`. Vérifier que les 2 premiers octets sont bien `FFD8` ; sinon afficher une erreur claire.
- Afficher dans la barre de status : nom de fichier, taille en octets, dimensions image, sous-échantillonnage chroma détecté, présence/absence de RST markers.

### 4.2 Parser de structure (toujours actif)

À chaque chargement ou modification, parser linéairement le fichier pour produire une liste de segments :

```js
// Structure suggérée
{
  segments: [
    { type: 'SOI', offset: 0, length: 2 },
    { type: 'APP0', offset: 2, length: 18, info: { identifier: 'JFIF', version: '1.01', ... } },
    { type: 'DQT', offset: 20, length: 69, info: { tableId: 0, precision: 8 } },
    // ...
    { type: 'SOS', offset: N, length: L, info: { components: [...] } },
    { type: 'SCAN', offset: N+L, length: M, // données entropy-coded
      rstMarkers: [ { offset: ..., index: 0 }, ... ] },
    { type: 'EOI', offset: ..., length: 2 }
  ],
  width: 1920, height: 1080,
  subsampling: '4:2:0',
  mcuSize: { w: 16, h: 16 },
  mcuCount: { x: 120, y: 68, total: 8160 },
  restartInterval: 0  // ou N si DRI présent
}
```

Le panneau « Structure » affiche cet arbre, chaque ligne cliquable pour sauter à l'offset correspondant dans le hex viewer.

### 4.3 Mapping octets ↔ pixels (cœur de l'outil)

L'outil propose **trois modes**, sélectionnables dans le panneau « Mapping » :

#### Mode A — Proportionnel (rapide, approximatif)

- Pour un octet à l'offset `o` dans le scan data (entre `scan_start` et `scan_end`), on calcule sa position relative `r = (o - scan_start) / (scan_end - scan_start)`.
- On considère que `r` correspond à la même fraction du nombre total de MCU.
- L'index MCU est `mcu_index = floor(r * mcu_count.total)`, converti en coordonnées (x, y) en MCU puis en pixels.
- Toujours disponible, calcul instantané. **Imprécis localement** mais donne une intuition correcte globalement.
- Pour les octets en dehors du scan data (headers), pas de highlight image — afficher à la place un tooltip texte (« en-tête JFIF », « table de quantification », etc.).

#### Mode B — RST-based (rapide, précis aux frontières)

- Si RST markers présents : on connaît exactement l'offset de début/fin de chaque segment de N MCU consécutifs.
- Entre deux RST, on retombe sur du proportionnel mais sur un segment beaucoup plus court → précision bien meilleure.
- Si pas de RST markers : proposer un bouton « Injecter des RST markers » qui : décode l'image en canvas, ré-encode via canvas.toBlob avec un encodeur JPEG custom (ou en utilisant un mini-encodeur JS comme `jpeg-js` chargé via CDN si on accepte une dépendance), avec un `restartInterval` réglable (recommandé : 1 ou 8 MCU par segment).

#### Mode C — Huffman complet (lent, exact)

- Décoder réellement le bitstream Huffman pour identifier la frontière de chaque MCU.
- Étapes :
  1. Parser les tables DHT (DC et AC pour Y, Cb, Cr) en arbres de décodage.
  2. À partir de SOS, lire bit par bit le scan data, en gérant le byte unstuffing (`FFxx` → `FF` si `xx == 00`).
  3. Pour chaque MCU : décoder les 6 blocs (Y0, Y1, Y2, Y3, Cb, Cr en 4:2:0), chaque bloc consistant en 1 coefficient DC + 63 coefficients AC, chacun précédé d'un code Huffman donnant (run-length, taille du coefficient).
  4. Mémoriser l'offset binaire (octet + bit) de début de chaque MCU.
- Implémentation **en Web Worker** obligatoire : sur une image full-HD, le parsing peut prendre 1-3 secondes.
- Cacher le résultat tant que le scan data n'est pas modifié dans une zone parsée.
- En cas de modification d'un octet : invalider seulement à partir du RST marker précédent (ou du début si pas de RST) ; reparser de là.

L'utilisateur choisit le mode via un sélecteur. Indiquer clairement le compromis vitesse/précision.

### 4.4 Highlight bidirectionnel

**Octets → Image** : quand la souris survole un octet dans le hex viewer, le canvas image affiche un rectangle semi-transparent sur la zone de pixels correspondante (1 MCU minimum, plus si l'octet représente plusieurs MCU en mode proportionnel).

**Image → Octets** : quand la souris survole un pixel dans le canvas, le hex viewer scrolle automatiquement (optionnel, on peut le rendre désactivable) et highlight l'octet ou la plage d'octets correspondant à la MCU sous le curseur.

**Sélection persistante** : un clic verrouille la sélection. Shift+clic étend. Cmd/Ctrl+clic ajoute une sélection multiple disjointe.

Les highlights distinguent visuellement :
- Survol simple : couleur claire, opacité 30%
- Sélection verrouillée : couleur saturée, opacité 50%, bordure
- Octet courant (curseur du clavier) : encadré net

Couleurs suggérées (compatibles dark mode, qui est le mode par défaut esthétique de cet outil) :
- En-têtes / marqueurs : jaune `#e8b86d`
- Scan data : bleu `#6d9be8`
- RST markers : vert `#6de89b`
- Zone sélectionnée pour édition : rouge `#e86d6d`

### 4.5 Édition

Le hex viewer doit être **éditable** :

- Mode hex : taper deux caractères hex remplace l'octet sous le curseur, avance d'un octet.
- Mode ASCII : taper un caractère remplace l'octet correspondant.
- Touches `Insert` / `Delete` : insérer / supprimer un octet (à utiliser avec parcimonie, ça décale tout le reste).
- Sélection par drag ou Shift+flèches → opérations groupées.

Outils d'édition rapide dans le panneau « Édition », opérant sur la sélection courante :

| Outil | Description |
|---|---|
| Randomize | Remplace les octets sélectionnés par des valeurs aléatoires |
| XOR | XOR la sélection avec une constante (saisir en hex) |
| Add/Sub | Ajoute ou soustrait une constante (modulo 256) à chaque octet |
| Shift | Décale la sélection de N positions vers la gauche ou la droite |
| Reverse | Inverse l'ordre des octets de la sélection |
| Repeat | Répète le premier octet sur toute la sélection |
| Inject RST | Insère un marqueur RST à l'offset courant (utile pour partitionner manuellement) |
| Delete RST | Supprime tous les RST markers dans la sélection |
| Swap blocks | Demande une seconde sélection puis swap |
| Bit flip | Flip N bits aléatoires dans la sélection |

**Contraintes** :
- Avertir si l'édition touche un en-tête critique (DQT, DHT, SOF). C'est autorisé (c'est du datamoshing après tout) mais l'image peut devenir totalement indécodable.
- L'édition dans le scan data doit gérer le byte stuffing : si l'utilisateur écrit `FF` quelque part, l'outil propose d'insérer automatiquement le `00` suivant (option configurable).

### 4.6 Re-rendu live

Après chaque modification :
- Régénérer un `Blob` à partir du `Uint8Array`,
- Créer une `URL.createObjectURL`,
- Créer une `Image` et l'attendre via `onload` / `onerror`,
- Si succès : dessiner sur le canvas.
- Si erreur : le canvas affiche le dernier rendu valide en grisé, plus un overlay « Image indécodable » avec l'offset probable de l'erreur (peut être déduit en parsant jusqu'à où ça reste valide).

Throttler les re-rendus (debounce 100-150ms) pendant la frappe ou les sliders d'outils, pour ne pas saturer.

### 4.7 Export

- Télécharger `.jpg` (buffer courant tel quel).
- Télécharger `.png` (rendu canvas — utile si le JPG est indécodable, capture le dernier rendu valide).
- Reset (retour au fichier original chargé).
- Historique : panneau latéral optionnel listant les N dernières opérations, cliquable pour restaurer un état.
- Export diff binaire `.json` (uniquement les octets modifiés : `[{ offset, original, modified }]`).
- Export séquence d'opérations `.json` (liste ordonnée des outils appliqués avec leurs paramètres — permet le replay exact).
- Copier canvas en base64 (copie dans le presse-papier le rendu courant encodé en `data:image/png;base64,...`).
- Export structure parsée `.json` (segments, offsets, longueurs, métadonnées — sortie du parser §4.2).

## 5. Panneau « Structure » (vue détaillée)

Arbre/liste affichant les segments parsés. Pour chaque segment :

- Type (SOI, DQT, etc.) avec icône de couleur correspondant au highlight.
- Offset (en hex) et longueur (en décimal et hex).
- Bouton « Aller à » qui scrolle le hex viewer et highlight le segment entier.
- Pour DQT : afficher la matrice 8×8 décodée, en couleur (heatmap).
- Pour DHT : afficher le nombre de codes par longueur, et un aperçu des arbres.
- Pour SOF : afficher dimensions, composants, subsampling.
- Pour SCAN : afficher la liste des RST markers (s'il y en a) avec leurs offsets et l'index MCU correspondant.

## 6. Détails techniques d'implémentation

### 6.1 Hex viewer performant

Un JPG de 5 MB = 5 millions d'octets = ~312 500 lignes de 16 octets. **Ne pas rendre tout le DOM** sinon le navigateur crashe.

Implémenter une **virtualisation** :
- Hauteur totale du conteneur scrollable = `nbLines * lineHeight`.
- Sur scroll, ne rendre que les lignes visibles (+ buffer de quelques dizaines au-dessus/en-dessous).
- Approche simple : un `position: absolute` par ligne visible avec `top` calculé.

Hauteur de ligne fixe, police monospace.

### 6.2 Décodage natif

```js
function renderImage(uint8array) {
  const blob = new Blob([uint8array], { type: 'image/jpeg' });
  const url = URL.createObjectURL(blob);
  const img = new Image();
  img.onload = () => {
    canvas.width = img.naturalWidth;
    canvas.height = img.naturalHeight;
    ctx.drawImage(img, 0, 0);
    URL.revokeObjectURL(url);
  };
  img.onerror = () => {
    showDecodeError();
    URL.revokeObjectURL(url);
  };
  img.src = url;
}
```

Subtilité : certains navigateurs sont « tolérants » aux JPG corrompus et affichent quand même une image partielle. C'est exactement ce qu'on veut. Tester sur Chromium et Firefox — leurs comportements diffèrent légèrement, c'est intéressant à exposer à l'utilisateur (note dans la doc).

### 6.3 Parser de marqueurs (pseudo-code)

```js
function parseJpeg(buffer) {
  const segments = [];
  let offset = 0;
  const view = new DataView(buffer.buffer);

  while (offset < buffer.length - 1) {
    if (buffer[offset] !== 0xFF) {
      // anomalie (corruption?), avancer ou stopper
      break;
    }
    const marker = buffer[offset + 1];

    if (marker === 0xD8) { // SOI
      segments.push({ type: 'SOI', offset, length: 2 });
      offset += 2;
    } else if (marker === 0xD9) { // EOI
      segments.push({ type: 'EOI', offset, length: 2 });
      break;
    } else if (marker >= 0xD0 && marker <= 0xD7) { // RST (ne devrait pas apparaître hors scan)
      segments.push({ type: `RST${marker - 0xD0}`, offset, length: 2 });
      offset += 2;
    } else if (marker === 0xDA) { // SOS
      const len = view.getUint16(offset + 2);
      segments.push({ type: 'SOS', offset, length: len + 2, info: parseSosHeader(buffer, offset, len) });
      offset += 2 + len;
      // Maintenant on lit le scan jusqu'au prochain marqueur non-stuffed et non-RST
      const scanStart = offset;
      const rstMarkers = [];
      while (offset < buffer.length - 1) {
        if (buffer[offset] === 0xFF && buffer[offset + 1] !== 0x00) {
          const m = buffer[offset + 1];
          if (m >= 0xD0 && m <= 0xD7) {
            rstMarkers.push({ offset, index: m - 0xD0 });
            offset += 2;
            continue;
          }
          break; // marqueur réel, fin du scan
        }
        offset++;
      }
      segments.push({ type: 'SCAN', offset: scanStart, length: offset - scanStart, rstMarkers });
    } else {
      // marqueur générique avec longueur
      const len = view.getUint16(offset + 2);
      segments.push({ type: markerName(marker), offset, length: len + 2, info: parseGeneric(marker, buffer, offset, len) });
      offset += 2 + len;
    }
  }
  return segments;
}
```

### 6.4 Encodage MCU → pixels

Pour le subsampling 4:2:0 (le plus courant), une MCU = 16×16 pixels. Si l'image fait `W × H` :

```js
mcuCountX = Math.ceil(W / mcuWidth);
mcuCountY = Math.ceil(H / mcuHeight);
mcuTotal = mcuCountX * mcuCountY;

// MCU index → coordonnées pixels du coin haut-gauche
function mcuToPixel(index) {
  const my = Math.floor(index / mcuCountX);
  const mx = index % mcuCountX;
  return {
    x: mx * mcuWidth,
    y: my * mcuHeight,
    w: Math.min(mcuWidth, W - mx * mcuWidth),
    h: Math.min(mcuHeight, H - my * mcuHeight)
  };
}
```

### 6.5 Web Worker pour Huffman

Le worker reçoit le buffer + les tables DHT/SOS parsées, et retourne un tableau `mcuOffsets[i] = { byteOffset, bitOffset }`. Communication via `postMessage` avec transfert du buffer (transferable) pour éviter la copie.

Pendant le calcul, afficher une barre de progression. Permettre l'annulation.

## 7. UX et état initial

À l'ouverture, l'app affiche un écran d'accueil avec :
- Zone de drop centrale grande et claire.
- Un mini-tutoriel (3-4 lignes) sur ce qu'est le datamoshing JPG.
- Optionnel : un bouton « Charger un exemple » pour démarrer immédiatement avec un JPG fourni.

Une fois un fichier chargé, le layout principal apparaît. Sauvegarder en `localStorage` les préférences UI (taille des panneaux, mode de mapping préféré, etc.).

## 8. Style visuel

Thème sombre par défaut, esthétique d'éditeur de code / outil pro :

- Background `#1a1a1a`
- Surfaces `#222` et `#2a2a2a`
- Bordures `#333` / `#444`
- Texte `#ddd` principal, `#888` secondaire
- Accent chaud `#e8b86d` (orange/ambre) pour les boutons et highlights primaires
- Police monospace dans le hex viewer (`'JetBrains Mono', 'SF Mono', 'Consolas', monospace`)
- Police UI : système (`-apple-system, sans-serif`)

Pas de fioritures, pas d'animations excessives. Transitions sobres (150ms). L'outil doit donner une sensation d'instrument, pas de site web.

## 9. Priorisation de l'implémentation

Construire dans cet ordre, en testant chaque étape :

1. **Squelette HTML/CSS** + chargement de fichier + affichage hex de base (sans virtualisation, sur petits fichiers d'abord).
2. **Décodage et affichage image** dans le canvas.
3. **Parser de marqueurs JPG** + panneau Structure cliquable.
4. **Virtualisation du hex viewer** (passer à des gros fichiers).
5. **Mode mapping A (proportionnel)** + highlight bidirectionnel au survol.
6. **Édition basique** d'octets dans le hex + re-render live.
7. **Outils d'édition rapide** (randomize, XOR, etc.).
8. **Mode mapping B (RST-based)** + injection de RST markers.
9. **Historique undo/redo** + export.
10. **Mode mapping C (Huffman complet)** en Web Worker (optionnel, peut être différé).
11. **Polish** : tooltips, raccourcis clavier (Ctrl+Z, Ctrl+S, etc.), drag-and-drop, écran d'accueil.

À chaque étape, l'application doit rester utilisable. Préférer des features moins nombreuses mais qui marchent à un truc complet mais fragile.

## 10. Pièges connus

- **Byte stuffing** : ne pas oublier de gérer `FF00` dans le scan, en particulier lors de l'édition. Si l'utilisateur tape `FF` dans le scan, on doit soit auto-insérer un `00`, soit l'avertir clairement.
- **Cache navigateur** : `URL.createObjectURL` doit être suivi de `revokeObjectURL` pour ne pas fuir de la mémoire. Sur de gros fichiers édités intensivement, c'est critique.
- **Performances de re-rendu** : sur un JPG > 5MB, créer un blob + image à chaque keystroke est lent. Debouncer agressivement (150ms minimum) et permettre un mode « pas de live preview » pour les utilisateurs qui veulent éditer librement sans attendre.
- **Dimensions du canvas** : pour les très grandes images (> 4000px), utiliser `transform: scale()` en CSS pour l'affichage, pas en redimensionnant le canvas natif (sinon on perd en qualité de highlight).
- **Encoders différents** : un JPG produit par Photoshop, par un smartphone, par MozJPEG, par GIMP n'ont pas les mêmes en-têtes. L'outil doit être robuste à cette diversité (parser tolérant, sans assumptions sur l'ordre des segments).
- **Progressive JPEG (SOF2)** : structure interne très différente (plusieurs scans entremêlés pour DC/AC séparés). Pour la v1, on peut limiter le support aux baseline JPG (SOF0) et afficher un avertissement si SOF2 est rencontré.

## 11. Tests suggérés

Tester avec :
- Un petit JPG simple (< 100 KB, dimensions paires multiples de 16, 4:2:0, sans RST).
- Un JPG avec RST markers (en générer un via `jpegtran -restart 1` ou similaire).
- Un JPG d'appareil photo plein de métadonnées Exif (APP1).
- Un JPG progressif (pour vérifier le warning).
- Un fichier non-JPG (pour vérifier la détection).
- Un JPG très grand (> 10 MB) pour vérifier la virtualisation.

## 12. Livrables attendus

- `index.html` (ou ensemble HTML/CSS/JS minimaliste).
- Un `README.md` court expliquant comment lancer (`python -m http.server` ou double-clic), les raccourcis clavier, et les fonctionnalités.
- Code commenté généreusement, particulièrement aux endroits techniques (parser, mapping, Huffman).
- Aucune dépendance externe non-CDN. Si CDN nécessaire (par exemple pour un encodeur JPEG custom), le justifier et permettre un fallback.

---

*Fin de la spec. Implémenter dans l'ordre du §9. En cas de doute, privilégier la robustesse à la richesse fonctionnelle.*
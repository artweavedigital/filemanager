# ⚡ ImageWork Optimizer pour ZwiiCMS 14.6

**ImageWork** est une extension "Plug & Play" pour le cœur de ZwiiCMS. Elle ajoute des capacités de compression moderne (AVIF/WebP) et de redimensionnement intelligent directement dans le gestionnaire de fichiers (FileManager) natif, sans altérer les images originales.

------

## 📂 Architecture & Emplacements

Le module est conçu pour être centralisé dans le répertoire `core/picturetools/` afin de faciliter la maintenance.

| **Composant**           | **Fichier**                           | **Action**                                                   |
| ----------------------- | ------------------------------------- | ------------------------------------------------------------ |
| **Injecteur**           | `core/layout/layout.class.php`        | **Modifié** : Appelle le module JS pour l'admin.             |
| **Interface (UI)**      | `core/picturetools/imagework.js`      | **Créé** : Gère l'affichage de l'éclair `⚡` et de la modale. |
| **Processeur (Moteur)** | `core/picturetools/img_processor.php` | **Créé** : Effectue les calculs PHP, rotations et compressions. |
| **Sortie**              | `site/file/source/optimized/`         | **Dossier** : Stocke les fichiers compressés générés.        |

------

## 🛠 Installation & Changements apportés

### 1. Activation dans le Core

Le fichier `core/layout/layout.class.php` a été modifié dans son constructeur pour injecter le script uniquement lorsque l'utilisateur est authentifié comme administrateur :

PHP

```
public function __construct(core $core) {
    parent::__construct();
    $this->core = $core;

    // Injection du module ImageWork
    if (isset($_COOKIE['ZWII_AUTH_KEY'])) {
        echo '<script src="core/picturetools/imagework.js" defer></script>';
    }
}
```

### 2. Logique du Processeur PHP

Le moteur utilise la bibliothèque **GD** de PHP (8.1+ requis pour l'AVIF).

- **Redimensionnement proportionnel** : Le script calcule automatiquement la hauteur en fonction de la largeur saisie pour conserver le ratio d'aspect.
- **Sécurité** : Il empêche l'agrandissement d'images plus petites que la cible pour éviter la pixellisation.

------

## 🚀 Guide d'utilisation du FileManager

### Étape 1 : Localisation de l'outil

Dans le gestionnaire de fichiers de ZwiiCMS, chaque vignette d'image (`.jpg`, `.png`, `.webp`) affiche désormais un **éclair vert `⚡`** au survol ou de manière permanente en haut à droite.

### Étape 2 : Configuration de l'image

En cliquant sur l'éclair, une fenêtre modale élégante s'ouvre :

1. **Format de sortie** : Choisissez **AVIF** (poids plume, futuriste) ou **WebP** (universel).
2. **Largeur max** : Saisissez la largeur cible (ex: 1200px). La hauteur sera adaptée automatiquement par le serveur.

### Étape 3 : Résultat

- L'image originale reste intacte dans son dossier source.
- La version optimisée est générée dans `site/file/source/optimized/`.
- Un message confirme le **pourcentage de gain de poids** (souvent entre -70% et -95%).

------

## 📝 Notes pour le futur développeur

- **Modification de l'UI** : Les styles CSS de la fenêtre modale sont injectés directement via `imagework.js` (balise `<style>`).

- **Compatibilité Iframe** : Le script `imagework.js` utilise un intervalle de scan (`setInterval`) pour détecter l'Iframe du FileManager, car celui-ci est chargé dynamiquement par Zwii.

- **Ratio d'aspect** : La formule utilisée dans `img_processor.php` pour la hauteur est :

  `$newHeight = floor($height * ($newWidth / $width));`

- **Debug** : En cas de message "Image introuvable", vérifiez que le chemin `data-path` envoyé par le FileManager correspond bien à l'arborescence réelle sous `site/file/source/`.

------


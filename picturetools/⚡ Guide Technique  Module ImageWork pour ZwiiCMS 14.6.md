

------

# ⚡ Guide Technique : Module ImageWork pour ZwiiCMS 14.6

Ce module est une extension personnalisée pour le cœur de ZwiiCMS. Il permet d'optimiser les images (conversion AVIF/WebP et redimensionnement intelligent) directement depuis le gestionnaire de fichiers.

------

## 🛠 1. Architecture du Module

Le module est conçu pour être le moins intrusif possible. Il repose sur trois composants :

| **Fichier**           | **Emplacement**                       | **Rôle**                                             |
| --------------------- | ------------------------------------- | ---------------------------------------------------- |
| **Le Cerveau (PHP)**  | `core/picturetools/img_processor.php` | Traitement lourd : redimensionnement et compression. |
| **L'Interface (JS)**  | `core/picturetools/imagework.js`      | Injection du bouton ⚡ et fenêtre modale de réglages. |
| **L'Injecteur (PHP)** | `core/layout/layout.class.php`        | Active le module pour les administrateurs.           |

------

## 💾 2. Code des Fichiers

### 📄 Fichier A : `core/layout/layout.class.php`

**Modification :** Localisez le constructeur de la classe `layout` et remplacez-le pour inclure l'injection du script.

PHP

```
<?php
class layout extends common {
    private $core;

    public function __construct(core $core) {
        parent::__construct();
        $this->core = $core;

        // Injection ImageWork : uniquement si l'utilisateur est connecté à l'admin
        if (isset($_COOKIE['ZWII_AUTH_KEY'])) {
            echo '<script src="core/picturetools/imagework.js" defer></script>';
        }
    }
    // ... reste du code original ...
}
```

### 📄 Fichier B : `core/picturetools/imagework.js`

**Rôle :** Gère l'UI et la communication asynchrone (Fetch).

JavaScript

```
(function() {
    // Styles élégants pour la fenêtre modale
    const style = document.createElement('style');
    style.innerHTML = `
        .iw-modal { position:fixed; top:50%; left:50%; transform:translate(-50%, -50%); background:#fff; z-index:10000; padding:20px; border-radius:8px; box-shadow:0 10px 40px rgba(0,0,0,0.4); width:320px; font-family:sans-serif; }
        .iw-overlay { position:fixed; top:0; left:0; width:100%; height:100%; background:rgba(0,0,0,0.5); z-index:9999; backdrop-filter: blur(2px); }
        .iw-group { margin-bottom:12px; }
        .iw-label { display:block; font-size:12px; color:#666; margin-bottom:4px; }
        .iw-input, .iw-select { width:100%; padding:8px; border:1px solid #ccc; border-radius:4px; box-sizing:border-box; }
        .iw-btn-go { background:#28a745; color:#fff; width:100%; padding:10px; border:none; border-radius:4px; cursor:pointer; font-weight:bold; margin-top:10px; }
    `;
    document.head.appendChild(style);

    window.openImageWork = function(pathValue) {
        const overlay = document.createElement('div');
        overlay.className = 'iw-overlay';
        const modal = document.createElement('div');
        modal.className = 'iw-modal';
        modal.innerHTML = `
            <div style="font-weight:bold; margin-bottom:15px; text-align:center;">⚡ OPTIMISEUR IMAGEWORK</div>
            <div class="iw-group">
                <span class="iw-label">Format de sortie :</span>
                <select id="iw-format" class="iw-select">
                    <option value="avif">AVIF (Performances 2026)</option>
                    <option value="webp">WebP (Compatibilité)</option>
                </select>
            </div>
            <div class="iw-group">
                <span class="iw-label">Largeur max (px) :</span>
                <input type="number" id="iw-width" class="iw-input" value="1600">
            </div>
            <button class="iw-btn-go" id="iw-save">Lancer la compression</button>
            <button onclick="this.parentElement.parentElement.remove()" style="width:100%; background:none; border:none; margin-top:10px; cursor:pointer; color:#999; font-size:11px;">Annuler</button>
        `;
        document.body.appendChild(overlay);
        overlay.appendChild(modal);

        modal.querySelector('#iw-save').onclick = function() {
            const fd = new FormData();
            fd.append('src', pathValue);
            fd.append('format', document.getElementById('iw-format').value);
            fd.append('width', document.getElementById('iw-width').value);
            
            this.innerHTML = "Traitement en cours...";
            this.disabled = true;

            fetch('core/picturetools/img_processor.php', { method: 'POST', body: fd })
            .then(r => r.json()).then(data => {
                if(data.status === 'success') {
                    alert("Succès ! Image sauvegardée dans 'optimized/'\nGain : " + data.gain);
                } else {
                    alert("Erreur : " + data.error);
                }
                overlay.remove();
            }).catch(() => alert("Erreur serveur"));
        };
    };

    function injectBtn(doc) {
        doc.querySelectorAll('figure[data-name]').forEach(fig => {
            const path = fig.getAttribute('data-path') || fig.getAttribute('data-name');
            if (path && path.match(/\.(jpg|jpeg|png)$/i) && !fig.querySelector('.p-tools')) {
                const btn = document.createElement('div');
                btn.className = 'p-tools';
                btn.innerHTML = '⚡';
                btn.style = "position:absolute; top:5px; right:5px; background:#28a745; color:#fff; padding:2px 6px; border-radius:3px; cursor:pointer; z-index:1000; font-weight:bold; font-size:11px;";
                btn.onclick = (e) => { e.preventDefault(); e.stopPropagation(); window.openImageWork(path); };
                fig.appendChild(btn);
            }
        });
    }

    setInterval(() => {
        injectBtn(document);
        document.querySelectorAll('iframe').forEach(i => { try { injectBtn(i.contentDocument); } catch(e){} });
    }, 2000);
})();
```

### 📄 Fichier C : `core/picturetools/img_processor.php`

**Rôle :** Moteur de traitement GD.

PHP

```
<?php
header('Content-Type: application/json');

// Chemins relatifs vers la racine de Zwii
$base_dir = "../../site/file/source/"; 
$dest_dir = "../../site/file/source/optimized/";

$src_raw = $_POST['src'] ?? '';
$format  = $_POST['format'] ?? 'avif';
$newW    = (int)($_POST['width'] ?? 1600);

// Sécurité et nettoyage du chemin
$src_path = str_replace(['../', './'], '', $src_raw);
$full_src = $base_dir . $src_path;

if (!file_exists($full_src)) {
    echo json_encode(['status' => 'error', 'error' => 'Source introuvable']);
    exit;
}

// 1. Création de la ressource image
$info = getimagesize($full_src);
switch ($info['mime']) {
    case 'image/jpeg': $img = imagecreatefromjpeg($full_src); break;
    case 'image/png':  $img = imagecreatefrompng($full_src); break;
    case 'image/webp': $img = imagecreatefromwebp($full_src); break;
    default: die(json_encode(['error' => 'Format non supporté']));
}

// 2. Redimensionnement proportionnel (Aspect Ratio conservé)
$w = imagesx($img);
$h = imagesy($img);
if ($newW > 0 && $newW < $w) {
    $newH = floor($h * ($newW / $w));
    $tmp = imagecreatetruecolor($newW, $newH);
    imagealphablending($tmp, false);
    imagesavealpha($tmp, true);
    imagecopyresampled($tmp, $img, 0, 0, 0, 0, $newW, $newH, $w, $h);
    imagedestroy($img);
    $img = $tmp;
}

// 3. Sauvegarde dans le dossier optimized
if (!is_dir($dest_dir)) mkdir($dest_dir, 0755, true);
$filename = pathinfo($src_path, PATHINFO_FILENAME) . '.' . $format;
$out = $dest_dir . $filename;

$res = ($format === 'avif') ? imageavif($img, $out, 65) : imagewebp($img, $out, 75);

if ($res) {
    echo json_encode([
        'status' => 'success',
        'gain' => round((1 - (filesize($out) / filesize($full_src))) * 100) . '%'
    ]);
} else {
    echo json_encode(['status' => 'error', 'error' => 'Erreur de conversion']);
}
imagedestroy($img);
```

------

## 🚀 3. Guide d'utilisation

### Fonctionnement visuel

1. Ouvrez le **Gestionnaire de fichiers** dans ZwiiCMS.
2. Survolez n'importe quelle image : un petit éclair vert `⚡` apparaît.
3. Cliquez sur l'éclair pour ouvrir les options.

### Sortie des fichiers

- Les images originales ne sont **jamais modifiées**.
- Les nouvelles images sont stockées dans `site/file/source/optimized/`.
- Le script calcule la hauteur automatiquement pour éviter toute déformation.

### Maintenance

- **Version PHP requise** : 8.1+ (pour le support AVIF).
- **Droits d'écriture** : Le dossier `site/file/source/optimized/` doit être accessible en écriture par le serveur.

------

*Fin du document. Module développé en 2026 par les stagiaires de artweave.fr.fo pour ZwiiCMS. 14.6

------

Est-ce que cette version te convient pour tes archives ?
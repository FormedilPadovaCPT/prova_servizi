# Icone PWA

Questa cartella deve contenere le icone PNG per la PWA.

## Icone necessarie

| File | Dimensioni | Uso |
|------|------------|-----|
| `icon-72x72.png` | 72x72 px | Icona piccola |
| `icon-96x96.png` | 96x96 px | Shortcut |
| `icon-128x128.png` | 128x128 px | Google Play |
| `icon-144x144.png` | 144x144 px | Chrome |
| `icon-152x152.png` | 152x152 px | iPad |
| `icon-192x192.png` | 192x192 px | Android |
| `icon-384x384.png` | 384x384 px | Retina |
| `icon-512x512.png` | 512x512 px | Play Store |

## Come generare le icone

### Opzione 1: Online Generator
1. Vai su https://pwafactory.io/icon-generator/ o https://realfavicongenerator.net/
2. Carica il logo Formedil (formato quadrato, min 512x512px)
3. Scarica tutte le icone e copiale in questa cartella

### Opzione 2: Da riga di comando (ImageMagick)
```bash
# Installare ImageMagick, poi:
convert logo.png -resize 72x72 icon-72x72.png
convert logo.png -resize 96x96 icon-96x96.png
# ... ecc.
```

### Opzione 3: Script Python con Pillow
```python
from PIL import Image
logo = Image.open('logo.png')
sizes = [72, 96, 128, 144, 152, 192, 384, 512]
for size in sizes:
    img = logo.resize((size, size), Image.LANCZOS)
    img.save(f'icon-{size}x{size}.png')
```

## Note
- Le icone devono essere quadrate, senza sfrangiature
- Sfondo: usa il colore `#1e2d5e` (blu Formedil) o trasparente
- Formato: PNG con canale alpha

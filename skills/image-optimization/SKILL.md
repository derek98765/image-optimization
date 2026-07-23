# Image Optimization Skill

Convert all `.png` / `.jpg` / `.jpeg` images in `public/images` to `.webp`, resizing oversized images first based on their actual display size in the app.

## Prerequisites

Ensure `cwebp` is installed:
```bash
brew install webp
```

## Steps

### 1. Detect all PNG/JPG images
```bash
find public/images -type f \( -iname "*.png" -o -iname "*.jpg" -o -iname "*.jpeg" \) | sort
```

Get dimensions and file sizes for each:
```bash
for f in public/images/**/*; do
  dims=$(sips -g pixelWidth -g pixelHeight "$f" 2>/dev/null | awk '/pixel/{printf $2" "}')
  size=$(du -sh "$f" | cut -f1)
  echo "$size  $dims  $(basename $f)"
done
```

### 2. Check display sizes in the codebase

Search for all usages of the images in source files:
```bash
grep -rn "public/images\|/images/" src --include="*.jsx" --include="*.tsx" --include="*.js" --include="*.ts" -l
```

For each file, look for the CSS size applied to the `<img>` tag (e.g. `w-[40px] h-[40px]`, `w-full`, aspect ratios). Then determine the appropriate resize target:

| Display context | Resize target |
|----------------|---------------|
| Fixed icon (40–64px), 2× retina | **128px** |
| Card banner / hero (up to ~800px wide) | **800px** |
| Full-width hero (>800px) | **1200px** |
| Already small (≤256px) | No resize needed |

**Skip-if-close guard:** only resize if the image's actual longest side is
**more than 1.5× its tier's target**. If it's already within 1.5×, leave it
at its original size and go straight to WebP conversion — don't resize it.

e.g. a 900px-wide image destined for the 800px card-banner tier is *not*
resized (900 < 1.5 × 800 = 1200) — it converts to WebP at its original
900px. A 1400px image in that same slot *is* resized (1400 > 1200).

### 3. Resize oversized images

Use `sips` to resize in-place (macOS built-in) — only for images that fail
the skip-if-close guard above:
```bash
# Resize to 800px (longest side, preserves aspect ratio)
sips -Z 800 path/to/image.png --out path/to/image.png

# Resize to 128px
sips -Z 128 path/to/image.png --out path/to/image.png
```

### 4. Convert to WebP

```bash
cd public/images/<subfolder>

for f in *.png; do
  base="${f%.*}"
  cwebp -q 100 "$f" -o "${base}.webp"
done

# Handle JPEG separately (zsh glob quirk — *.jpg errors if no matches)
for f in *.jpg *.jpeg; do
  [ -f "$f" ] || continue
  base="${f%.*}"
  cwebp -q 100 "$f" -o "${base}.webp"
done
```

Quality 100 avoids visible compression artifacts, at the cost of larger
files than a lower quality setting — use when fidelity matters more than
file size.

### 5. Update source file references

**Static hardcoded paths** — find and replace `.png`/`.jpg` → `.webp` directly in source files.

**Dynamic paths built from API data** — the filename comes from the backend (e.g. `app.image = "finance_buddy.png"`). Patch the path-construction expression to swap the extension at runtime:

```js
// Before
`/images/app-icons/${app.image}`

// After
`/images/app-icons/${app.image.replace(/\.(png|jpe?g)$/i, ".webp")}`
```

Apply this pattern wherever a local `/images/` path is built dynamically from a variable that may carry a `.png` or `.jpg` extension.

### 6. Delete original files

```bash
cd public/images/<subfolder>
rm -f *.png *.jpg *.jpeg
```

Verify nothing remains:
```bash
ls | grep -E "\.png$|\.jpg$|\.jpeg$"
```

## Checklist (use TodoWrite)

- [ ] Detect all PNG/JPG images and their dimensions
- [ ] Check display sizes in source and determine resize targets
- [ ] Skip resize for any image already within 1.5× of its tier's target
- [ ] Resize oversized images to appropriate dimensions
- [ ] Convert all PNG/JPG to `.webp` at quality 100
- [ ] Update static source references to `.webp`
- [ ] Update dynamic path constructions to swap extension at runtime
- [ ] Delete original PNG/JPG files
- [ ] Verify no originals remain

## Notes

- Never convert `.svg` files — they are already vector and smaller than any raster format.
- Already-`.webp` files need no action.
- `sips -Z <n>` resizes the longest side while preserving aspect ratio.
- In zsh, `*.jpg` throws an error if no matches exist — always guard with `[ -f "$f" ] || continue`.
- If the image filename comes from an external API you don't control, use the runtime `.replace()` approach rather than trying to rename files.

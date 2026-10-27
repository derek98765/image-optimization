---
name: image-optimization
description: Converts all .png/.jpg/.jpeg images in public/images and src/assets to .webp, safely downscaling oversized images to their largest rendered size (with retina headroom) so they stay sharp. Use when you want to reduce image file sizes and improve page load speed.
allowed-tools: Read Glob Grep Bash
---

# Image Optimization Skill

Convert all `.png` / `.jpg` / `.jpeg` images in **`public/images`** and **`src/assets`** to `.webp`. Oversized images are downscaled first, but only to a *safe* size derived from how large each image actually renders, so it stays clear and high quality at every size and screen density.

## Prerequisites

Ensure `cwebp` is installed:
```bash
brew install webp
```

## Steps

### 1. Detect all PNG/JPG images

Scan both locations:
```bash
find public/images src/assets -type f \( -iname "*.png" -o -iname "*.jpg" -o -iname "*.jpeg" \) | sort
```

Get dimensions and file sizes for each:
```bash
find public/images src/assets -type f \( -iname "*.png" -o -iname "*.jpg" -o -iname "*.jpeg" \) | sort | while read -r f; do
  dims=$(sips -g pixelWidth -g pixelHeight "$f" 2>/dev/null | awk '/pixel/{printf $2" "}')
  size=$(du -sh "$f" | cut -f1)
  echo "$size  $dims  $f"
done
```

### 2. Find every usage and the largest rendered size

The two locations are referenced differently, so search for both:

- **`public/images`**: string paths (`"/images/..."`, template literals, JSON data, CSS `url(/images/...)`).
- **`src/assets`**: ES imports (`import hero from "@/assets/images/homepage/hero.png"`, relative `../assets/...`), `new URL(...)`, `require(...)`, and CSS `url(...)` in `src/**/*.css`. Resolve each import to the identifiers/JSX where it is used.

```bash
grep -rnE "/images/|assets/images|assets/hero" src public/index.html --include="*.jsx" --include="*.tsx" --include="*.js" --include="*.ts" --include="*.css" --include="*.json" --include="*.html"
```

For **every** usage of an image, find the largest size it renders at (CSS px) across all breakpoints: `w-[..]`/`h-[..]`/size tokens, `md:`/`lg:` overrides, `max-w-*`, parent container widths, `object-fit`/`aspect-*` crops, and `background-size`. Responsive `w-full` means the widest container it can fill, not the image's own width (e.g. the max content width, or full viewport width on a full-bleed hero).

**If an image is used in several places, use the largest rendered size among them.**

### 3. Compute the safe resize target

Don't use fixed tiers. Per image:

```
required_px = max_rendered_css_px × density_factor
```

- `density_factor` = **3** for small fixed elements (icons, avatars, logos ≤ ~100 CSS px, which are cheap and the most visible when soft), otherwise **2** (standard retina). Use 3 for any image that is full-bleed on mobile.
- `max_rendered_css_px` is measured on the dimension that drives the render (the longest rendered side, accounting for `object-cover` crops: if the box crops the image, size so the *scaled* image still covers the box on both axes).
- Round `required_px` **up** to the next multiple of 16.

**Safety rules** (all must hold before resizing):

1. **Never upscale.** If the image is already ≤ `required_px`, leave it at its original size.
2. **Skip-if-close guard:** only resize when the actual longest side is **more than 1.5× `required_px`**. Within 1.5×, keep the original dimensions and just convert to WebP.
3. **Unknown or dynamic usage, don't resize.** If you can't determine the rendered size (unused, passed through props, dynamic API images, or used by CSS you can't trace), skip the resize and only convert. Report these to the user.
4. **Preserve aspect ratio** and resize the longest side. Never crop.
5. **Never go below the largest rendered size × 2**, even if a rule above would suggest it.
6. Anything that may be zoomed, lightboxed, downloaded or used as an OG/social image keeps its original size.

Show the user a table (file, original px, largest render px, `required_px`, action) before changing anything.

### 4. Resize oversized images

Use `sips` to resize in-place (macOS built-in), only for images that passed the safety rules above:
```bash
# Resize the longest side to <required_px>, preserving aspect ratio
sips -Z <required_px> path/to/image.png --out path/to/image.png
```

### 5. Convert to WebP

Run per folder (both `public/images/**` and `src/assets/**`):

```bash
cd <folder>

for f in *.png; do
  [ -f "$f" ] || continue
  base="${f%.*}"
  cwebp -q 100 "$f" -o "${base}.webp"
done

# Handle JPEG separately (zsh glob quirk, since *.jpg errors if no matches)
for f in *.jpg *.jpeg; do
  [ -f "$f" ] || continue
  base="${f%.*}"
  cwebp -q 100 "$f" -o "${base}.webp"
done
```

Quality 100 avoids visible compression artifacts, at the cost of larger
files than a lower quality setting. Use when fidelity matters more than
file size.

Keep a `.webp` only if it is smaller than the original; otherwise delete the new `.webp` and leave the original untouched (and its references unchanged).

### 6. Update source file references

**Static hardcoded paths** (`public/images`): find and replace `.png`/`.jpg` → `.webp` directly in source files.

**ES imports** (`src/assets`): update the import specifier extension (`import hero from "@/assets/images/homepage/hero.png"` → `.webp`). Also update `new URL(...)`, `require(...)` and CSS `url(...)` references. Vite fingerprints imports, so a missed one fails the build; run `yarn build` (or `yarn tsc --noEmit`) afterwards to catch dangling imports.

**Dynamic paths built from API data**: the filename comes from the backend (e.g. `app.image = "finance_buddy.png"`). Patch the path-construction expression to swap the extension at runtime:

```js
// Before
`/images/app-icons/${app.image}`

// After
`/images/app-icons/${app.image.replace(/\.(png|jpe?g)$/i, ".webp")}`
```

Apply this pattern wherever a local `/images/` path is built dynamically from a variable that may carry a `.png` or `.jpg` extension.

### 7. Delete original files

Only delete originals whose `.webp` was kept and whose references were updated:
```bash
cd <folder>
rm -f <converted-originals>
```

Verify nothing converted remains and nothing still references the old name:
```bash
find public/images src/assets -type f \( -iname "*.png" -o -iname "*.jpg" -o -iname "*.jpeg" \)
grep -rnE "\.(png|jpe?g)" src --include="*.jsx" --include="*.tsx" --include="*.js" --include="*.ts" --include="*.css"
```

## Checklist (use TodoWrite)

- [ ] Detect all PNG/JPG images in `public/images` **and** `src/assets`, with dimensions
- [ ] Find every usage (string paths and ES imports) and the largest rendered size per image
- [ ] Compute `required_px` (render size × density, rounded up) and show the table
- [ ] Never upscale; skip resize within 1.5× of `required_px`; skip resize if usage is unknown
- [ ] Resize only the images that qualify
- [ ] Convert to `.webp` at quality 100, keeping it only if smaller
- [ ] Update static paths, ES imports and CSS `url()` references
- [ ] Update dynamic path constructions to swap extension at runtime
- [ ] Delete converted originals
- [ ] Run `yarn build` / typecheck and verify no stale references remain

## Notes

- Never convert `.svg` files. They are already vector and smaller than any raster format.
- Already-`.webp` files need no action.
- `sips -Z <n>` resizes the longest side while preserving aspect ratio.
- In zsh, `*.jpg` throws an error if no matches exist, so always guard with `[ -f "$f" ] || continue`.
- If the image filename comes from an external API you don't control, use the runtime `.replace()` approach rather than trying to rename files.
- Prefer a larger-than-needed result over a soft one. When in doubt, keep more pixels.

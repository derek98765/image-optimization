---
name: image-optimization
description: Audits and optimizes all images in the project. Resizes images that are larger than their rendered size, then converts everything to WebP using Google's cwebp tool for best performance. Use when you want to reduce image file sizes and improve page load speed.
allowed-tools: Read Glob Grep Bash
---

Optimize all images in the project by resizing oversized images and converting everything to WebP.

## Steps

### 1. Discover image directories

Before doing anything else, locate where images actually live in this project. Do not assume a fixed path — run a broad search:

```bash
find public src -type f \( -iname "*.png" -o -iname "*.jpg" -o -iname "*.jpeg" -o -iname "*.webp" \) \
  -not -path "*/node_modules/*" | head -5
```

Common locations include (but are not limited to):
- `public/images/`, `public/img/`
- `src/assets/images/`, `src/assets/img/`
- `src/images/`, `src/img/`

Use whichever directories actually contain images. If images are spread across multiple locations, process all of them.

### 2. Audit image dimensions

For each discovered image, check its pixel dimensions using `sips -g pixelWidth -g pixelHeight`. Cross-reference how the image is actually used in the codebase (grep for its filename in `.jsx`, `.tsx`, `.css` files) to determine the maximum rendered width. Flag any image whose source dimension is meaningfully larger than its rendered size (e.g. a 2560px image used at 1920px max).

### 3. Resize oversized images

For each flagged image, resize it down to the maximum rendered width (preserving aspect ratio) using `sips --resampleWidth`. Use these guidelines:
- Full-width hero/section images: max 1920px wide
- Half-width images (two-column layouts): max 1200px wide
- Card/thumbnail images: max 800px wide
- Icons and small UI images: leave as-is (don't upscale)

If unsure of the rendered size, default to 1920px for large images and 800px for smaller ones.

### 4. Convert all images to WebP

Convert every image (after resizing where needed) to WebP using **Google's `cwebp` tool** (install via `brew install webp` if not present):
```bash
cwebp -q 85 input.jpg -o output.webp
cwebp -q 85 input.png -o output.webp
```

Use quality `-q 85` as the default. For images with transparency (PNG), use `-q 90`.

After converting, for each original file that was successfully converted (i.e. a `.webp` counterpart exists at the same path):
1. **Delete the original** `.png` / `.jpg` / `.jpeg` file.
2. **Update all code references** — search `.jsx`, `.tsx`, `.ts`, `.css`, and `index.html` files (excluding `node_modules`) and replace every occurrence of the old extension with `.webp`. Use `perl -pi -e` for reliable in-place replacement:
   ```bash
   find src public -type f \( -name "*.jsx" -o -name "*.tsx" -o -name "*.ts" -o -name "*.css" -o -name "*.html" \) \
     -not -path "*/node_modules/*" | xargs perl -pi -e \
     's/\.(png|jpg|jpeg)(["'"'"'\s\)\}])/\.webp$2/gi'
   ```

### 5. Verify & summarise

After all conversions:
- Confirm no `.jpg`, `.jpeg`, `.png` files remain in the image directories (except files that failed conversion — keep those and flag them).
- Confirm all import/src references in code have been updated to `.webp`.
- **Always output a summary table** with every image, formatted as:

| Image | Original Dims | Final Dims | Before | After |
|-------|--------------|------------|--------|-------|
| path/to/image.webp | WxH | WxH | XX KB | XX KB |

Include totals at the bottom (total before KB/MB, total after KB/MB, overall % reduction).

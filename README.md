# image-optimization

A Claude Code skill that converts `.png` / `.jpg` / `.jpeg` images in
`public/images` to `.webp`, resizing oversized images first based on their
actual display size in the app.

## What's in this repo

```
skills/
  image-optimization/SKILL.md
```

Install by copying `skills/image-optimization` into `~/.claude/skills/`
(or a project's `.claude/skills/`).

## Prerequisites

- macOS (uses `sips`, the built-in image tool, for resizing).
- [`cwebp`](https://developers.google.com/speed/webp/docs/cwebp) installed:
  ```bash
  brew install webp
  ```
- Images to optimize living under `public/images` in the project. (If your
  project keeps images elsewhere, adjust the paths in Steps 1–4 before
  running.)

## What it does and why

Unoptimized PNG/JPG assets are one of the most common causes of slow page
loads. This skill automates the full cleanup in one pass:

1. **Detects** every PNG/JPG under `public/images` and reads its actual
   pixel dimensions.
2. **Checks how each image is actually displayed** in the codebase (grepping
   `.jsx`/`.tsx`/`.js`/`.ts` source for its usage and CSS size) and maps it
   to a resize tier — fixed icon, card banner, or full-width hero.
3. **Resizes only what's actually oversized** — a skip-if-close guard skips
   resizing any image whose original size is already within 1.5× of its
   tier's target, so a 900px image bound for an 800px slot is left alone
   instead of being downscaled unnecessarily. This avoids the classic
   failure mode of resizing being too aggressive and leaving images visibly
   soft.
4. **Converts everything to WebP** at quality 100, avoiding visible
   compression artifacts.
5. **Updates every source reference** — both static hardcoded paths and
   paths built dynamically from API data (e.g. `app.image` from a backend
   response) via a runtime `.replace()` pattern.
6. **Deletes the original PNG/JPG files** and verifies none remain.

`.svg` files are never touched (already vector, already smaller than any
raster format), and already-`.webp` files need no action.

## How to invoke

```
/image-optimization
```

Or just ask in conversation — e.g. "optimize the images in this project" —
and Claude will pick up the skill automatically based on its description.

## Example

Given `public/images/app-icons/finance-buddy.png` at 1400×1400px, used as a
64px fixed icon (2× retina) in `src/components/AppGrid.tsx`:

- **Resize target:** 128px tier (fixed icon, 2× retina).
- **Guard check:** 1400 > 1.5 × 128 (= 192), so it gets resized down to
  128px before conversion.
- **Convert:** `cwebp -q 100 finance-buddy.png -o finance-buddy.webp`.
- **Update references:** any hardcoded `"finance-buddy.png"` in source
  becomes `"finance-buddy.webp"`; if the filename is built dynamically from
  API data (e.g. `app.image`), the path expression is patched to
  `app.image.replace(/\.(png|jpe?g)$/i, ".webp")` instead of renaming files.
- **Delete original**, confirm no `.png`/`.jpg`/`.jpeg` remain in that
  folder.

## Notes

- `sips -Z <n>` resizes the longest side while preserving aspect ratio.
- In zsh, `*.jpg` throws an error if no matches exist — the skill guards
  every JPEG loop with `[ -f "$f" ] || continue`.
- If an image filename comes from an external API you don't control, the
  skill patches the path-construction expression at runtime rather than
  trying to rename files to match.

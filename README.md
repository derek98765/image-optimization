# image-optimization

A Claude Code skill that converts `.png` / `.jpg` / `.jpeg` images in
`public/images` and `src/assets` to `.webp`, downscaling oversized images
first to a safe size based on how large each one actually renders.

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

## What it does and why

Unoptimized PNG/JPG assets are one of the most common causes of slow page
loads. This skill automates the cleanup in one pass:

1. **Detects** every PNG/JPG under `public/images` **and** `src/assets`,
   with its pixel dimensions and file size.
2. **Finds every usage** of each image: string paths for `public/images`,
   and ES imports, `new URL(...)`, `require(...)` and CSS `url()` for
   `src/assets`. It records the largest size the image renders at across
   all breakpoints (including `object-cover` crops).
3. **Computes a safe resize target** instead of using fixed tiers:
   `required_px = largest rendered CSS px x density`, where density is 3x
   for small icons/avatars and images that fill the screen on mobile, and
   2x otherwise, rounded up to a multiple of 16.
4. **Resizes only when it is safe**:
   - never upscales;
   - skips images already within 1.5x of the target;
   - skips resizing when usage is unknown or dynamic;
   - leaves zoomable, downloadable and OG/social images at full size.
   A plan table is shown before anything changes.
5. **Converts to WebP** at quality 100, keeping the `.webp` only if it is
   smaller than the original.
6. **Updates every reference**: hardcoded paths, ES import specifiers, CSS
   `url()`, and dynamic paths built from API data via a runtime
   `.replace()` pattern.
7. **Deletes the converted originals**, checks no stale references remain,
   and runs a build/typecheck to catch dangling imports.

`.svg` files are never touched (already vector), and already-`.webp` files
need no action.

## How to invoke

```
/image-optimization
```

Or just ask in conversation, e.g. "optimize the images in this project".

## Example

Given `src/assets/images/homepage/hero.png` at 3200x1800px, rendered at most
1280 CSS px wide on desktop:

- **Required size:** 1280 x 2 = 2560px.
- **Guard check:** 3200 is not more than 1.5 x 2560 (= 3840), so it is not
  resized. It is only converted.
- **Convert:** `cwebp -q 100 hero.png -o hero.webp`, kept if smaller.
- **Update references:** `import hero from "@/assets/images/homepage/hero.png"`
  becomes `.../hero.webp`, then the build is run to confirm nothing dangles.

A 64px icon stored at 1400x1400 would get `required_px` = 64 x 3 = 192, and
since 1400 > 1.5 x 192 it is resized to 192px before conversion.

## Notes

- `sips -Z <n>` resizes the longest side while preserving aspect ratio.
- In zsh, `*.jpg` throws an error if no matches exist, so every JPEG loop is
  guarded with `[ -f "$f" ] || continue`.
- If an image filename comes from an external API you don't control, the
  skill patches the path-construction expression at runtime rather than
  trying to rename files.
- When in doubt it keeps more pixels: a larger result is preferred over a
  soft one.

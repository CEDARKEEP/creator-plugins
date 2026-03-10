# Image Quality Validation

Validation pipeline for all generated images. Run after every render.

## Validation Checklist

### 1. File Integrity

```bash
# Verify file exists and is a valid image
file {output-file}
# Expected: "PNG image data, 1280 x 720, 8-bit/color RGBA, non-interlaced"
```

- File must exist at the expected output path
- File must be a valid image (not a 0-byte or corrupt file)
- If render fails, check TypeScript compilation errors first

### 2. Dimensions

| Image Type | Width | Height | Aspect Ratio |
|-----------|-------|--------|-------------|
| YouTube Thumbnail | 1280 | 720 | 16:9 |
| Instagram Post | 1080 | 1080 | 1:1 |
| Twitter/X Header | 1500 | 500 | 3:1 |
| Facebook Cover | 820 | 312 | ~2.63:1 |
| OG Image | 1200 | 630 | ~1.9:1 |

Verify with:
```bash
# Using identify (ImageMagick)
identify -format "%wx%h" {output-file}

# Or check via file command
file {output-file} | grep -oP '\d+ x \d+'
```

### 3. File Size

| Platform | Max Size | Target |
|----------|----------|--------|
| YouTube Thumbnail | 2 MB | < 1.5 MB |
| General web | 5 MB | < 2 MB |

```bash
# Check file size
ls -lh {output-file}
# Or in bytes:
stat -c %s {output-file}
```

If file size exceeds limit:
- Switch from PNG to JPEG with `--image-format=jpeg --jpeg-quality=85`
- Reduce `--scale` if using scale > 1
- Simplify gradients or reduce image asset resolution

### 4. Format Verification

| Content Type | Recommended Format | Reason |
|-------------|-------------------|--------|
| Text-heavy, graphics | PNG | Preserves sharp edges, no compression artifacts |
| Photo-heavy | JPEG (quality 85-90) | Smaller file size, acceptable quality |
| Mixed (text + photos) | PNG | Prefer text clarity |
| Web-optimized | WebP | Smallest size, modern browser support |

### 5. Text Readability Test

The most critical validation for thumbnails. Text must be readable at **mobile preview size** (160×90 pixels).

**Mental simulation:**
- Imagine the thumbnail shrunk to the size of a postage stamp
- Can you still read the headline?
- Can you identify the subject/face?

**Common readability failures:**
- Font too small (< 60px at 1280×720)
- Font weight too light (< 700)
- Insufficient contrast between text and background
- Missing text stroke/outline
- Too many words (> 5)
- Text overlapping busy image area without overlay

### 6. Safe Zone Check

Verify no critical content in:
- Bottom-right corner (timestamp overlay)
- Bottom edge (title bar, progress bar)
- Extreme edges (may be cropped on some devices)

**Critical elements to keep in safe zone:**
- All text
- Faces (especially eyes)
- Key graphic elements
- Logos or branding

### 7. Color Contrast

- Text must have visible stroke AND shadow
- Subject must be visually separated from background
- Accent colors must be distinguishable from primary colors
- If using gradient text, it must still read clearly

## Validation Output Template

When presenting validation results to the user:

```
Output: {thumbnail-name}/{thumbnail-name}.png
Dimensions: 1280×720 (16:9) ✓
File size: {size}KB ({size}MB) ✓ (< 2MB)
Format: PNG ✓

Design summary:
- Archetype: {archetype name}
- Colors: {primary} + {secondary} + {accent}
- Font: {font name} at {size}px
- Layout: {brief layout description}
- Layers: {number} layers ({layer descriptions})

Readability: {assessment at mobile size}
Safe zones: {pass/fail}

Suggestions:
- {A/B variant idea}
- {optional refinement}
```

## Common Issues and Fixes

| Issue | Cause | Fix |
|-------|-------|-----|
| Blank/white image | Component error, no background set | Check console errors, add explicit background color |
| Missing text | Font not loaded before render | Ensure `loadFont()` is called at module level |
| Blurry text | Using JPEG format | Switch to PNG (`--image-format=png`) |
| Oversized file | High-res PNG with many layers | Use JPEG for photo-heavy, or reduce complexity |
| Missing images | Wrong path or failed download | Verify `staticFile()` path or URL accessibility |
| Layout overflow | Text too long for container | Use `fitText()` or reduce word count |
| Invisible text | Low contrast, no stroke | Add `-webkit-text-stroke` and `textShadow` |
| Timestamp collision | Text in bottom-right | Add bottom-right padding (150px wide, 50px tall) |

# Typography Fundamentals for Image Generation

Font and text layout knowledge for all image skills.

## Font Classification for Thumbnails

### Display / Impact Fonts (Headlines)
Best for: Primary headline text that needs to grab attention instantly.

- **Condensed sans-serif** (Bebas Neue, Anton, Oswald) — tall, narrow, maximizes text size in limited space
- **Black-weight sans-serif** (Montserrat Black, Poppins Black, Inter Black) — heavy, modern, clean
- **Novelty display** (Bangers, Permanent Marker, Black Ops One) — personality-driven, niche-specific

### Body / Support Fonts (Secondary Text)
Best for: Subtitles, descriptions, badges, supporting info.

- **Clean sans-serif** (Roboto, Open Sans, Lato) — highly legible, neutral
- **Medium-weight condensed** (Roboto Condensed, Barlow Condensed) — compact, professional

### Rule: Maximum 2 Fonts Per Image
- One **display/impact** font for the headline
- One **clean/neutral** font for secondary text (optional)
- Using more than 2 fonts creates visual clutter

## Font Weight Recommendations

| Use Case | Weight | CSS fontWeight |
|----------|--------|---------------|
| Primary headline | Extra Bold / Black | `800` or `900` |
| Secondary headline | Bold | `700` |
| Badge / label text | Bold or Semi-Bold | `600` or `700` |
| Body text (rare in thumbnails) | Regular | `400` |

**Key insight:** Font weight matters MORE than font size for thumbnail readability. An 80px bold font reads better than a 100px regular weight at small sizes.

## Font Size Guidelines

### YouTube Thumbnails (1280×720)

| Element | Size Range | Notes |
|---------|-----------|-------|
| Primary headline | 80–160px | Must read at 160×90 (mobile search) |
| Secondary text | 40–80px | Supporting info, subtitles |
| Badge/label | 30–50px | Corner badges, category labels |
| Never smaller than | 30px | Invisible at mobile preview size |

### Dynamic Sizing
Use `fitText()` from `@remotion/layout-utils` to automatically calculate the largest font size that fits within a given width. Always cap at a maximum (e.g., 160px) to prevent oversized text on short strings.

## Letter-Spacing and Line-Height

```tsx
// Tight letter-spacing for impact headlines
style={{ letterSpacing: '-0.02em' }}  // -2% — compact, punchy

// Normal letter-spacing for readability
style={{ letterSpacing: '0' }}

// Wide letter-spacing for small caps or labels
style={{ letterSpacing: '0.1em' }}   // +10% — spacious, elegant

// Line height for multi-line headlines
style={{ lineHeight: 1.0 }}   // Tight — for 2-3 word stacked headlines
style={{ lineHeight: 1.15 }}  // Normal — for longer text
```

## Text Hierarchy

Structure text elements with clear visual priority:

1. **Primary headline** — largest, boldest, highest contrast. This is what viewers read first.
2. **Secondary text** — smaller, lighter weight or different color. Supports the headline.
3. **Accent text** — badges, labels, numbers. Uses accent color for emphasis.

Example hierarchy:
```
[#1]  ← accent (badge, 40px, accent color, bold)
"MIND BLOWN"  ← primary (120px, white, black stroke, Bebas Neue)
"You won't believe this"  ← secondary (48px, off-white, no stroke, Inter)
```

## Text Transform

```tsx
// ALL CAPS — standard for thumbnail headlines (more impactful)
style={{ textTransform: 'uppercase' }}

// Title Case — for more refined/professional feel
// Apply manually or via CSS
style={{ textTransform: 'capitalize' }}
```

**Recommendation:** Use ALL CAPS for primary headlines in thumbnails. It maximizes visual weight and impact.

## CSS Text Effects

### Text Stroke (Outline) — Non-Negotiable for Thumbnails

Text MUST have stroke to ensure readability against any background:

```tsx
// Standard stroke (3-6px)
style={{
  WebkitTextStroke: '4px #000000',
  paintOrder: 'stroke fill',  // Renders stroke BEHIND the fill
  color: '#ffffff',
}}

// Thicker stroke for larger text
style={{
  WebkitTextStroke: '8px #000000',
  paintOrder: 'stroke fill',
  color: '#ffffff',
}}

// Colored stroke (matches palette)
style={{
  WebkitTextStroke: '5px #1e3a5f',
  paintOrder: 'stroke fill',
  color: '#f97316',
}}
```

**Stroke sizing rule:** `stroke-width = fontSize * 0.03` to `fontSize * 0.06`

### Text Shadow (Drop Shadow)

Add depth and separation:

```tsx
// Standard shadow
style={{ textShadow: '3px 3px 6px rgba(0,0,0,0.7)' }}

// Hard shadow (no blur — retro/bold feel)
style={{ textShadow: '4px 4px 0px rgba(0,0,0,0.9)' }}

// Multi-layer shadow (maximum depth)
style={{ textShadow: '2px 2px 0px #000, 4px 4px 0px rgba(0,0,0,0.5), 6px 6px 12px rgba(0,0,0,0.3)' }}

// Glow effect
style={{ textShadow: '0 0 10px #e94560, 0 0 30px #e94560, 0 0 60px rgba(233,69,96,0.5)' }}
```

### Gradient Text

```tsx
// Text with gradient fill (Chromium-only, works in Remotion)
style={{
  background: 'linear-gradient(to bottom, #ffffff 0%, #a0a0a0 100%)',
  WebkitBackgroundClip: 'text',
  WebkitTextFillColor: 'transparent',
  fontSize: 120,
  fontWeight: 900,
}}
```

**Note:** Gradient text does NOT work with `-webkit-text-stroke`. Use `text-shadow` for outline effect when using gradient fills.

### Combined Effect (Recommended Default)

```tsx
// The "YouTube thumbnail" text style
style={{
  fontSize: 120,
  fontWeight: 900,
  fontFamily: bebasNeue,
  color: '#ffffff',
  WebkitTextStroke: '5px #000000',
  paintOrder: 'stroke fill',
  textShadow: '4px 4px 8px rgba(0,0,0,0.7)',
  textTransform: 'uppercase',
  lineHeight: 1.0,
  letterSpacing: '-0.01em',
}}
```

## Text Placement Rules

### Safe Zones
- **10% margin** from all edges (128px horizontal, 72px vertical at 1280×720)
- **Avoid bottom-right 15%** — YouTube's timestamp overlay covers this area
- **Avoid bottom 10%** — title text and progress bar overlap here

### Alignment Patterns
- **Left-aligned, vertically centered** — most common for face+text thumbnails
- **Center-aligned, center** — bold text-only thumbnails
- **Left-aligned, bottom-left** — editorial/news style
- **Split layout** — text on one side, image on the other (50/50 or 40/60)

## Anti-Patterns

- **Never use thin/light font weights** — invisible at thumbnail size
- **Never use script/cursive fonts** — unreadable at small sizes
- **Never put text over busy image areas** without a shadow or overlay — becomes illegible
- **Never use more than 5 words** in the primary headline — viewers spend <2 seconds on thumbnails
- **Never skip text stroke** — even white text on dark backgrounds benefits from stroke for definition
- **Never use font sizes below 30px** — completely invisible in search results

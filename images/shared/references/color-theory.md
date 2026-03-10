# Color Theory for Image Generation

Color principles for all image skills — thumbnails, banners, social cards, and more.

## Complementary Color Pairs (Maximum Contrast)

Opposites on the color wheel — create the strongest visual pop:

| Pair | Hex Values | Effect |
|------|-----------|--------|
| Blue / Orange | `#2563eb` / `#f97316` | Trust + Energy (most popular in thumbnails) |
| Red / Cyan | `#ef4444` / `#06b6d4` | Urgency + Calm |
| Green / Magenta | `#22c55e` / `#d946ef` | Growth + Creativity |
| Purple / Yellow | `#8b5cf6` / `#eab308` | Luxury + Attention |
| Navy / Gold | `#1e3a5f` / `#f59e0b` | Authority + Premium |
| Dark Red / Teal | `#991b1b` / `#14b8a6` | Drama + Fresh |

## Color Psychology Mapping

Choose colors based on the emotional tone of the content:

| Color | Emotion/Association | Best For |
|-------|-------------------|----------|
| **Red** `#ef4444` | Urgency, excitement, danger, passion | Reaction videos, breaking news, challenges |
| **Orange** `#f97316` | Energy, warmth, enthusiasm, action | DIY, cooking, fitness, motivation |
| **Yellow** `#eab308` | Attention, optimism, happiness, caution | Tips, warnings, educational, kids content |
| **Green** `#22c55e` | Growth, money, nature, success, health | Finance, health, environment, tutorials |
| **Blue** `#3b82f6` | Trust, calm, professionalism, tech | Tech reviews, business, science, tutorials |
| **Purple** `#8b5cf6` | Luxury, creativity, mystery, wisdom | Premium content, art, mystery, music |
| **Pink** `#ec4899` | Fun, playful, feminine, romantic | Lifestyle, beauty, entertainment |
| **Black** `#000000` | Power, elegance, sophistication | Luxury, tech, drama, cinematic |
| **White** `#ffffff` | Clean, minimal, pure, modern | Minimalist, medical, tech, professional |

## The 60-30-10 Rule

Structure your color palette with clear hierarchy:

- **60% — Dominant color** (background, large areas). Sets the mood.
- **30% — Secondary color** (supporting elements, subject area). Creates contrast.
- **10% — Accent color** (text highlights, badges, call-to-action). Draws the eye.

Example for a tech thumbnail:
- 60%: Dark navy `#0f172a` (background)
- 30%: White `#ffffff` (text, subject outline)
- 10%: Electric blue `#3b82f6` (accent glow, badge)

## Contrast Requirements

### Text Readability

- **Minimum 30% brightness difference** between text and its immediate background
- **Always use text stroke + shadow** on thumbnails — text must read on any background region
- White text (`#ffffff`) on dark backgrounds (`< #444`) = safe default
- Dark text on light backgrounds needs less stroke but still benefits from shadow

### Subject-Background Separation

- The main subject should be at least **30% brighter or darker** than the background
- Use a colored outline/glow (5-10px) around subjects for additional separation
- If subject and background colors are similar, add a gradient overlay to create contrast

## Gradient Patterns

### CSS Gradient Syntax

```css
/* Linear gradient — directional transition */
background: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%);

/* Radial gradient — spotlight/focus effect */
background: radial-gradient(circle at 60% 50%, #e94560 0%, transparent 60%);

/* Conic gradient — sweep effect */
background: conic-gradient(from 45deg, #3b82f6, #8b5cf6, #ec4899, #3b82f6);

/* Text readability overlay */
background: linear-gradient(to right, rgba(0,0,0,0.85) 0%, rgba(0,0,0,0.4) 50%, transparent 100%);
```

### Common Gradient Patterns for Thumbnails

| Pattern | CSS | Use Case |
|---------|-----|----------|
| Dark vignette | `radial-gradient(ellipse, transparent 40%, rgba(0,0,0,0.7) 100%)` | Draws focus to center |
| Side fade | `linear-gradient(to right, rgba(0,0,0,0.9) 0%, transparent 60%)` | Text on left over image |
| Duotone | `linear-gradient(135deg, #667eea 0%, #764ba2 100%)` | Bold graphic style |
| Sunset warm | `linear-gradient(to bottom, #f97316 0%, #ef4444 50%, #991b1b 100%)` | Energy, excitement |
| Ocean cool | `linear-gradient(135deg, #0f172a 0%, #1e3a5f 50%, #0ea5e9 100%)` | Trust, tech, calm |
| Neon dark | `linear-gradient(135deg, #0a0a0a 0%, #1a0a2e 100%)` + glow overlay | Gaming, night, edgy |

## Niche-Specific Color Conventions

| Niche | Primary | Secondary | Accent | Background |
|-------|---------|-----------|--------|------------|
| **Tech/Programming** | Blue `#3b82f6` | White | Green/Orange | Dark `#0f172a` |
| **Gaming** | Neon Green `#22d3ee` | Purple `#a855f7` | Red | Black `#0a0a0a` |
| **Food/Cooking** | Warm Red `#dc2626` | Yellow `#facc15` | White | Warm neutral |
| **Finance/Business** | Green `#16a34a` | Navy `#1e3a5f` | Gold `#f59e0b` | Dark or white |
| **Fitness/Health** | Red `#ef4444` | Black | Yellow/Orange | Dark gradient |
| **Beauty/Lifestyle** | Pink `#ec4899` | Purple `#a855f7` | Gold | Light or pastel |
| **Education/Tutorial** | Blue `#2563eb` | White | Yellow `#eab308` | Clean light or dark |
| **Entertainment/Comedy** | Yellow `#eab308` | Red `#ef4444` | White | Bright or dark |
| **Music** | Purple `#7c3aed` | Pink `#ec4899` | White | Dark gradient |
| **Travel/Nature** | Teal `#14b8a6` | Orange `#f97316` | White | Photo-based |

## Color Palette Recipes

### High-Energy (Challenges, Reactions)
```
Background: #0f0f0f (near black)
Primary text: #ffffff (white)
Accent: #ef4444 (red) or #f97316 (orange)
Stroke: #000000
```

### Professional (Tutorials, Business)
```
Background: #1e293b (slate)
Primary text: #f8fafc (off-white)
Accent: #3b82f6 (blue)
Stroke: #0f172a (dark navy)
```

### Fun/Playful (Entertainment, Kids)
```
Background: #fef3c7 (warm cream) or #7c3aed (purple)
Primary text: #1e1b4b (dark indigo) or #ffffff
Accent: #f97316 (orange) or #22d3ee (cyan)
Stroke: match background darkness
```

### Luxe/Premium
```
Background: #0c0a09 (rich black)
Primary text: #fbbf24 (gold)
Accent: #a855f7 (purple)
Stroke: #44403c (warm dark)
```

## Anti-Patterns

- **Never use more than 3 colors** in a thumbnail — visual noise reduces readability
- **Avoid pastel-on-pastel** — low contrast fails at small sizes
- **Don't use red text on blue background** (or vice versa) — chromatic aberration makes it vibrate/blur
- **Avoid pure white backgrounds** — they blend into YouTube's light mode UI. Use off-white `#f8fafc` or add a subtle border
- **Don't match YouTube's red** `#ff0000` — your thumbnail merges with the UI instead of standing out

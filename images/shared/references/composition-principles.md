# Composition Principles for Image Generation

Visual layout and compositional knowledge for all image skills.

## Rule of Thirds

Divide the canvas into a 3×3 grid. Place key elements at **intersection points** for natural visual balance.

```
┌──────────┬──────────┬──────────┐
│          │          │          │
│     ╳    │          │    ╳     │  ← eyes/face here
│          │          │          │
├──────────┼──────────┼──────────┤
│          │          │          │
│     ╳    │          │    ╳     │  ← text anchor points
│          │          │          │
├──────────┼──────────┼──────────┤
│          │          │          │
│          │          │          │
│          │          │          │
└──────────┴──────────┴──────────┘
```

### Common Thumbnail Layouts Using Thirds

| Layout | Description |
|--------|------------|
| **Face-left, text-right** | Face fills left 1/3, headline in right 2/3 |
| **Face-right, text-left** | Face fills right 1/3, headline in left 2/3 |
| **Center subject, text below** | Subject centered, text in bottom 1/3 |
| **Full-bleed text** | Text centered across middle 1/3 row, background fills rest |
| **Split screen** | Two subjects or concepts, each in left/right halves with divider |

## Visual Hierarchy

Control what viewers see first, second, and third using these tools (in order of impact):

1. **Size** — Larger elements attract attention first
2. **Contrast** — High contrast elements pop against low contrast surroundings
3. **Color** — Saturated/warm colors draw the eye before muted/cool colors
4. **Position** — Upper-left gets seen first (Z-pattern for left-to-right readers)
5. **Isolation** — An element surrounded by negative space stands out
6. **Detail** — Complex/detailed areas attract more attention than simple areas

### Hierarchy Example for Thumbnails

```
1st: Large headline text (size + contrast + color)
2nd: Face/subject image (size + detail + position)
3rd: Badge/number overlay (color + isolation)
4th: Background (lowest priority, supports other elements)
```

## Reading Patterns

### Z-Pattern (Thumbnails, Single Images)
Viewers scan in a Z shape: top-left → top-right → bottom-left → bottom-right.

```
START ──────────→ ·
                 ╱
               ╱
             ╱
·  ──────────→ END
```

**Implication:** Place the most important element (headline) in the top-left or top-right. Place secondary elements along the Z path.

### F-Pattern (Text-Heavy Layouts)
Viewers scan the top, then move down the left side looking for anchor points.

**Implication:** Left-align text for multi-line layouts. Place the strongest visual hook at the top.

## Focal Point Placement

### Single Focal Point
One dominant element draws all attention. Everything else supports it.

- **Face thumbnails:** The face IS the focal point. Make it 40%+ of the frame.
- **Text thumbnails:** The headline IS the focal point. Make it the largest, boldest element.
- **Product thumbnails:** The product IS the focal point. Isolate it with negative space.

### Dual Focal Points
Two competing elements create tension (good for VS/comparison thumbnails).

- Equal visual weight on both sides
- Connected by a divider, "VS" text, or arrow
- Neither element should overpower the other

## Negative Space

Empty areas that give your key elements room to breathe.

### Guidelines
- **30-40% negative space** is a good starting point for thumbnails
- Negative space doesn't mean white — it can be a solid color, subtle gradient, or blurred background
- Place negative space around text to ensure readability
- Use negative space to create "visual breathing room" between elements

### Techniques
```tsx
// Padding creates negative space around content
style={{ padding: '60px 80px' }}

// Limiting content width creates side margins
style={{ maxWidth: '70%' }}

// Semi-transparent overlay creates negative space over busy images
style={{ background: 'rgba(0,0,0,0.6)' }}
```

## Layer Ordering

Standard layer structure for image compositions (bottom to top):

```
Layer 1 (back):  BACKGROUND
                 Solid color, gradient, blurred image, or styled photo

Layer 2:         SUBJECT / CUTOUT
                 Isolated person, product, or main visual element
                 Often has outline/glow for separation

Layer 3:         GRAPHIC ELEMENTS
                 Shapes, arrows, circles, badges, dividers, icons
                 Positioned to direct attention or add emphasis

Layer 4:         TEXT
                 Headline, secondary text, labels
                 Always has stroke + shadow for readability

Layer 5 (front): EFFECTS / OVERLAYS
                 Vignette, color grading, glow effects, borders
                 Subtle enhancements that unify the composition
```

### In Remotion

```tsx
<AbsoluteFill>
  <Background />      {/* Layer 1 */}
  <Subject />          {/* Layer 2 */}
  <Graphics />         {/* Layer 3 */}
  <TextOverlay />      {/* Layer 4 */}
  <Effects />          {/* Layer 5 */}
</AbsoluteFill>
```

## Safe Zones

Different platforms overlay UI elements on images. Keep critical content within safe zones.

### YouTube Thumbnail Safe Zones (1280×720)

```
┌─────────────────────────────────────────────┐
│  ┌─────────────────────────────────────┐    │
│  │                                     │    │
│  │         SAFE ZONE                   │    │
│  │         (center 70-80%)             │    │
│  │                                     │    │
│  │                                     │    │
│  └─────────────────────────────────────┘    │
│                                     ┌──────┐│
│                                     │TIMER ││
│                                     └──────┘│
└─────────────────────────────────────────────┘
```

- **Top margin:** 36px (5%)
- **Left/right margins:** 64px (5%)
- **Bottom margin:** 72px (10%) — progress bar + title
- **Bottom-right:** 150×50px — video duration timestamp
- **Critical content area:** center 60-70% of the canvas

### In Remotion

```tsx
// Safe zone padding container
<AbsoluteFill style={{
  padding: '48px 64px',
  paddingBottom: 80,
}}>
  {/* Place text and critical elements here */}
</AbsoluteFill>
```

## Balance and Symmetry

### Symmetrical Layout
- Centers everything on the vertical axis
- Feels formal, stable, authoritative
- Best for: announcements, text-only, centered subjects
- Risk: can feel static/boring

### Asymmetrical Layout
- Off-center placement creates dynamic tension
- Feels energetic, modern, engaging
- Best for: face+text, action shots, storytelling
- Most thumbnails use asymmetrical layouts

### Visual Weight Balancing
Heavier elements (large, dark, detailed, saturated) should be balanced by lighter elements (small, light, simple, muted) on the opposite side. A large face on the left is balanced by bold text on the right.

## Scale and Proportion

### Face Sizing
- **40-60% of frame** for face-centric thumbnails
- Eyes should align with upper-third intersections
- Close-up crops (head + shoulders) outperform medium or wide shots
- Exaggerated expressions (wide eyes, open mouth) increase engagement

### Text Sizing Relative to Canvas
- Primary text: **8-12% of canvas height** (57-86px at 720px height)
- Secondary text: **4-6% of canvas height** (29-43px at 720px height)
- Use `fitText()` for automatic sizing, but enforce min/max bounds

### Icon/Badge Sizing
- Corner badges: **60-100px** diameter
- Inline icons: match surrounding text height
- Arrow elements: **40-80px** width

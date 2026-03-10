# Remotion Patterns for Image Generation

Core Remotion knowledge for generating still images. All image skills use these patterns.

## Still Component

Use `<Still>` instead of `<Composition>` — no `fps` or `durationInFrames` needed:

```tsx
import { Still } from 'remotion';

// In Root.tsx
export const RemotionRoot: React.FC = () => (
  <>
    <Still
      id="Thumbnail"
      component={ThumbnailComp}
      width={1280}
      height={720}
      defaultProps={{ title: 'Default Title' }}
    />
  </>
);
```

## Entry Point

```tsx
// src/index.ts
import { registerRoot } from 'remotion';
import { RemotionRoot } from './Root';
registerRoot(RemotionRoot);
```

## Layering with AbsoluteFill

`<AbsoluteFill>` creates a full-size absolutely positioned container. **DOM order = z-order** (later = on top). No `z-index` needed.

```tsx
import { AbsoluteFill } from 'remotion';

export const MyImage: React.FC = () => (
  <AbsoluteFill>
    {/* Layer 1 (back): Background */}
    <AbsoluteFill style={{ backgroundColor: '#1a1a2e' }}>
      {/* gradient, image, or solid color */}
    </AbsoluteFill>

    {/* Layer 2: Subject/Image */}
    <AbsoluteFill style={{ justifyContent: 'flex-end', alignItems: 'center' }}>
      <Img src={staticFile('subject.png')} style={{ height: '100%' }} />
    </AbsoluteFill>

    {/* Layer 3: Graphics (shapes, arrows, badges) */}
    <AbsoluteFill>
      {/* SVG elements, positioned divs, etc. */}
    </AbsoluteFill>

    {/* Layer 4 (front): Text */}
    <AbsoluteFill style={{ justifyContent: 'center', padding: 80 }}>
      <h1 style={{ fontSize: 120, fontWeight: 900, color: 'white' }}>TITLE</h1>
    </AbsoluteFill>
  </AbsoluteFill>
);
```

## Image Loading

### `<Img>` Component — Render-Blocking

Blocks rendering until the image is fully loaded. **Always use this instead of `<img>`.**

```tsx
import { Img, staticFile } from 'remotion';

// Local file from public/ folder
<Img src={staticFile('background.png')} style={{ width: '100%', height: '100%', objectFit: 'cover' }} />

// Remote URL
<Img src="https://example.com/photo.jpg" style={{ width: '100%' }} />
```

Props:
- `maxRetries` — default 2, with exponential backoff
- `onError` — handle load failures (must unmount or change src)

### `staticFile()`

References files in the project's `public/` folder:

```tsx
import { staticFile } from 'remotion';
const url = staticFile('backgrounds/hero.png'); // → public/backgrounds/hero.png
```

## Google Fonts — @remotion/google-fonts

Type-safe, render-blocking font loading. **Import at module level, not inside components.**

```tsx
import { loadFont } from '@remotion/google-fonts/BebasNeue';
const { fontFamily } = loadFont();

// In component:
<h1 style={{ fontFamily, fontWeight: 400, fontSize: 120 }}>TITLE</h1>
```

### Available Thumbnail-Appropriate Fonts

| Font | Import Path | Style |
|------|------------|-------|
| Bebas Neue | `@remotion/google-fonts/BebasNeue` | Tall condensed, all-caps display |
| Anton | `@remotion/google-fonts/Anton` | Ultra-bold condensed |
| Oswald | `@remotion/google-fonts/Oswald` | Modern condensed sans-serif |
| Montserrat | `@remotion/google-fonts/Montserrat` | Geometric sans-serif (use weight 900) |
| Poppins | `@remotion/google-fonts/Poppins` | Rounded geometric (use weight 900) |
| Inter | `@remotion/google-fonts/Inter` | Clean sans-serif (use weight 900) |
| Roboto Condensed | `@remotion/google-fonts/RobotoCondensed` | Neutral condensed |
| Bangers | `@remotion/google-fonts/Bangers` | Comic/playful display |
| Black Ops One | `@remotion/google-fonts/BlackOpsOne` | Military/gaming stencil |
| Permanent Marker | `@remotion/google-fonts/PermanentMarker` | Handwritten marker |
| Lato | `@remotion/google-fonts/Lato` | Friendly sans-serif |
| Open Sans | `@remotion/google-fonts/OpenSans` | Highly legible sans-serif |

**Loading with specific weights:**
```tsx
import { loadFont } from '@remotion/google-fonts/Montserrat';
const { fontFamily } = loadFont('normal', {
  weights: ['700', '900'],
  subsets: ['latin'],
});
```

## Dynamic Text Sizing — @remotion/layout-utils

### fitText() — Fit text within a width

```tsx
import { fitText } from '@remotion/layout-utils';

const { fontSize } = fitText({
  text: title,
  withinWidth: 900, // max width in pixels
  fontFamily: 'Bebas Neue',
  fontWeight: '400',
});

<h1 style={{ fontSize, fontFamily: 'Bebas Neue' }}>{title}</h1>
```

### measureText() — Get pixel dimensions of text

```tsx
import { measureText } from '@remotion/layout-utils';

const { width, height } = measureText({
  text: 'Hello World',
  fontFamily: 'Inter',
  fontSize: 72,
  fontWeight: '900',
});
```

**Critical:** Only call `fitText()`/`measureText()` AFTER fonts are loaded. Use `validateFontIsLoaded: true` for safety.

## Project Scaffolding

### Manual Scaffold (Recommended)

**Do NOT use `npx create-video@latest`** — it launches an interactive prompt wizard that cannot be automated in non-interactive shells. Instead, scaffold manually:

```bash
mkdir -p {project-name}/src {project-name}/public
```

Write `package.json` with all image generation packages:

```json
{
  "name": "{project-name}",
  "private": true,
  "scripts": {
    "studio": "npx remotion studio src/index.ts"
  },
  "dependencies": {
    "@remotion/cli": "4.0.434",
    "@remotion/google-fonts": "4.0.434",
    "@remotion/layout-utils": "4.0.434",
    "@remotion/shapes": "4.0.434",
    "@remotion/noise": "4.0.434",
    "@remotion/motion-blur": "4.0.434",
    "@remotion/preload": "4.0.434",
    "@remotion/zod-types": "4.0.434",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "remotion": "4.0.434",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "@types/react": "^18.3.3",
    "typescript": "^5.5.3"
  }
}
```

> **Version note:** All `@remotion/*` and `remotion` packages MUST use the same version. Check latest with `npm view remotion dist-tags --json` and use the `latest` tag value.

Write `src/index.ts`:
```tsx
import { registerRoot } from 'remotion';
import { RemotionRoot } from './Root';
registerRoot(RemotionRoot);
```

Then `cd {project-name} && npm install`.

### Package Capabilities

| Package | What It Enables |
|---------|----------------|
| `@remotion/google-fonts` | 1500+ Google Fonts loaded at render time |
| `@remotion/layout-utils` | `fitText()` / `measureText()` for auto-sizing text |
| `@remotion/shapes` | SVG shapes: `<Triangle>`, `<Circle>`, `<Star>`, `<Polygon>`, `<Arrow>`, `<Pie>`, `<Rect>`, `<Ellipse>` |
| `@remotion/noise` | Perlin/simplex noise for procedural textures, grain, organic backgrounds |
| `@remotion/motion-blur` | Blur/trail effects via `<CameraMotionBlur>` |
| `@remotion/preload` | `preloadImage()` / `preloadFont()` to prevent render flicker |
| `@remotion/zod-types` | Type-safe props with `zColor()` for Studio color pickers |

### Using an Existing Remotion Project

If the user already has a Remotion project:

1. Verify `remotion` is in `package.json`
2. Install missing packages: `npm install @remotion/google-fonts @remotion/layout-utils @remotion/shapes @remotion/noise @remotion/preload`
3. Add a new `<Still>` composition to the existing `Root.tsx`
4. Create the component file (e.g., `src/Thumbnail.tsx`)

### What to Customize After Scaffolding

The scaffold gives you the boilerplate. You then:

1. **Replace `src/Root.tsx`** — swap the default `<Composition>` for a `<Still>` at 1280×720
2. **Create `src/Thumbnail.tsx`** (or whatever the component is) — this is where all the design lives
3. **Copy user assets** to `public/` if needed

## CLI Rendering

### Basic still render

```bash
cd {project-dir} && npx remotion still src/index.ts Thumbnail --output={name}.png
```

### With input props

```bash
npx remotion still src/index.ts Thumbnail --output=out.png --props='{"title":"MY TITLE","subtitle":"Watch Now"}'
```

### High-resolution output (2x scale)

```bash
npx remotion still src/index.ts Thumbnail --output=out.png --scale=2
```

### Output formats

```bash
# PNG (default, best for text-heavy thumbnails)
--image-format=png

# JPEG (smaller file size, good for photo-heavy)
--image-format=jpeg --jpeg-quality=90

# WebP (smallest, modern format)
--image-format=webp
```

## Shapes — @remotion/shapes

SVG shape components for decorative elements, dividers, badges, and visual accents.

```tsx
import { Triangle, Circle, Star, Polygon, Pie, Rect, Ellipse } from '@remotion/shapes';

// Accent arrow pointing right
<Triangle length={80} direction="right" fill="#e94560" />

// Star badge (e.g., "NEW" or "TOP 10")
<Star points={5} innerRadius={30} outerRadius={60} fill="#FFD700" />

// Decorative circle behind text
<Circle radius={200} fill="rgba(233, 69, 96, 0.3)" />

// Hexagonal badge
<Polygon points={6} radius={50} fill="#635BFF" />

// Progress indicator / pie chart
<Pie radius={40} progress={0.75} fill="#00d4aa" closePath rotation={-Math.PI / 2} />

// Rounded rectangle badge
<Rect width={200} height={60} cornerRadius={12} fill="#1a1a2e" />
```

All shapes accept `style`, `stroke`, `strokeWidth`, and standard SVG props.

## Noise — @remotion/noise

Procedural noise for organic backgrounds, grain effects, and texture.

```tsx
import { noise2D, noise3D } from '@remotion/noise';

// Generate a noisy gradient background
const NoiseBackground: React.FC = () => {
  const width = 1280;
  const height = 720;
  const scale = 0.005;

  return (
    <AbsoluteFill>
      {Array.from({ length: 20 }).map((_, i) =>
        Array.from({ length: 12 }).map((_, j) => {
          const n = noise2D('bg', i * scale * 64, j * scale * 60);
          const opacity = 0.3 + n * 0.2;
          return (
            <div key={`${i}-${j}`} style={{
              position: 'absolute',
              left: i * 64, top: j * 60,
              width: 64, height: 60,
              backgroundColor: `rgba(233, 69, 96, ${opacity})`,
            }} />
          );
        })
      )}
    </AbsoluteFill>
  );
};
```

**Common noise uses:**
- Film grain overlay with semi-transparent noise dots
- Organic color variation across a gradient
- Procedural texture for backgrounds (avoids needing stock images)
- Subtle displacement for hand-drawn aesthetic

## Preload — @remotion/preload

Preload assets to prevent render flicker or missing resources.

```tsx
import { preloadImage, preloadFont } from '@remotion/preload';

// Preload at module level (before render)
preloadImage(staticFile('logo.png'));
preloadImage('https://example.com/hero.jpg');
```

## CSS Patterns for Thumbnails

### Gradients

```tsx
// Linear gradient background
style={{ background: 'linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%)' }}

// Radial gradient (spotlight effect behind subject)
style={{ background: 'radial-gradient(circle at 60% 50%, #e94560 0%, transparent 60%)' }}

// Gradient overlay for text readability
style={{ background: 'linear-gradient(to right, rgba(0,0,0,0.8) 0%, rgba(0,0,0,0.4) 50%, transparent 100%)' }}
```

### Text Stroke (Outline)

```tsx
// Webkit text stroke — works in Remotion (Chromium)
style={{
  WebkitTextStroke: '4px #000000',
  color: 'white',
  fontSize: 120,
  fontWeight: 900,
}}

// For thicker outlines, use paint-order to render stroke behind fill
style={{
  WebkitTextStroke: '8px #000000',
  paintOrder: 'stroke fill',
  color: 'white',
}}
```

### Text Shadow (Drop Shadow)

```tsx
// Standard drop shadow
style={{
  textShadow: '4px 4px 8px rgba(0,0,0,0.8)',
}}

// Multiple shadows for depth
style={{
  textShadow: '2px 2px 0px #000, 4px 4px 8px rgba(0,0,0,0.5)',
}}

// Glow effect
style={{
  textShadow: '0 0 20px #e94560, 0 0 40px #e94560, 0 0 80px #e94560',
}}
```

### Subject Outline/Glow

```tsx
// Drop shadow on an image (creates outline-like effect)
style={{
  filter: 'drop-shadow(0 0 8px rgba(255,255,255,0.8))',
}}

// Multiple drop shadows for glow
style={{
  filter: 'drop-shadow(0 0 4px #e94560) drop-shadow(0 0 12px #e94560)',
}}
```

### Transforms

```tsx
// Slight rotation for dynamic feel
style={{ transform: 'rotate(-3deg)' }}

// Scale up for emphasis
style={{ transform: 'scale(1.1)' }}

// Perspective tilt
style={{ transform: 'perspective(800px) rotateY(-5deg)' }}
```

## Common Component Patterns

### Safe Zone Wrapper

```tsx
const SafeZone: React.FC<{ children: React.ReactNode }> = ({ children }) => (
  <AbsoluteFill
    style={{
      padding: '48px 64px', // ~5-7% margins
      paddingBottom: 80,     // extra bottom margin for title bar
    }}
  >
    {children}
  </AbsoluteFill>
);
```

### Responsive Text Block

```tsx
const TitleText: React.FC<{ text: string; maxWidth: number; fontFamily: string }> = ({
  text, maxWidth, fontFamily
}) => {
  const { fontSize } = fitText({
    text,
    withinWidth: maxWidth,
    fontFamily,
    fontWeight: '900',
  });

  return (
    <h1 style={{
      fontSize: Math.min(fontSize, 160), // cap at 160px
      fontFamily,
      fontWeight: 900,
      color: 'white',
      WebkitTextStroke: `${Math.max(3, fontSize * 0.04)}px #000`,
      paintOrder: 'stroke fill',
      textShadow: '4px 4px 8px rgba(0,0,0,0.7)',
      lineHeight: 1.1,
      textTransform: 'uppercase',
    }}>
      {text}
    </h1>
  );
};
```

### Background Image with Overlay

```tsx
<AbsoluteFill>
  <Img src={backgroundUrl} style={{ width: '100%', height: '100%', objectFit: 'cover' }} />
  <AbsoluteFill style={{
    background: 'linear-gradient(to right, rgba(0,0,0,0.85) 0%, rgba(0,0,0,0.3) 60%, transparent 100%)',
  }} />
</AbsoluteFill>
```

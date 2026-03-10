# Image Iteration Core

Refinement workflow for all image skills. Mirrors the audio iteration pattern: surgical edits over full regeneration.

## Core Principle

Modify the existing TSX component and re-render rather than re-scaffolding with `npx create-video@latest`. The project was already scaffolded and dependencies installed — only the component code in `src/` needs to change.

## Iteration Workflow

### Step 1: Identify What to Change

Map the user's request to specific code locations:

| Request Type | Where to Edit | What to Change |
|-------------|---------------|----------------|
| Text content | `Thumbnail.tsx` props/JSX | String values in JSX elements |
| Font change | `Thumbnail.tsx` imports + styles | `loadFont` import, `fontFamily` in styles |
| Color change | `Thumbnail.tsx` styles | `color`, `backgroundColor`, gradient values |
| Layout change | `Thumbnail.tsx` structure | Flexbox properties, padding, element order |
| Add/remove element | `Thumbnail.tsx` JSX | Add/remove JSX elements and their layers |
| Image change | `Thumbnail.tsx` + `public/` | `src` prop on `<Img>`, copy new file to `public/` |
| Size/scale | `Root.tsx` or styles | `width`/`height` on `<Still>`, or `fontSize`/`transform` |

### Step 2: Show Proposed Changes

Before editing, present a summary:
```
Proposed changes:
- Change headline from "MIND BLOWN" → "GAME CHANGER"
- Swap accent color from #ef4444 (red) → #3b82f6 (blue)
- Increase text stroke from 4px → 6px
```

### Step 3: Make Surgical Edit

Edit ONLY the identified parameters in `Thumbnail.tsx`. Do not rewrite the entire file.

### Step 4: Re-Render

```bash
cd {thumbnail-name} && npx remotion still src/index.ts Thumbnail --output={thumbnail-name}_v2.png
```

### Step 5: Version Output

- NEVER overwrite the original file
- Version outputs: `_v2.png`, `_v3.png`, etc.
- Keep all versions for A/B comparison

## Preserve vs Regenerate

### ALWAYS Preserve
- Scaffolded project structure (package.json, tsconfig, node_modules — from `npx create-video@latest`)
- Entry point (src/index.ts)
- Root composition (src/Root.tsx) — unless dimensions change
- Layer structure — unless user explicitly asks for a "completely different" layout
- Font loading imports — unless font is being changed

### Regenerate When
- User says "completely different", "start over", "new design"
- More than 5 parameters would change
- The archetype itself needs to change (e.g., Reaction → Tutorial)
- Layout structure is fundamentally wrong

### Partial Regeneration
- Different colors only: Change style values, keep everything else
- Different text only: Change string content, may need `fitText` recalculation
- Different font only: Change import + fontFamily, may affect sizing
- Different layout: Restructure JSX, keep color/font/text choices
- Add image: Copy file to `public/`, add `<Img>` layer in appropriate position

## Common Refinement Mappings

| User Says | Action |
|-----------|--------|
| "bolder" / "more impactful" | Increase `fontWeight` to 900, increase stroke width by 2px, increase `fontSize` by 20% |
| "cleaner" / "simpler" | Remove decorative elements, increase padding, reduce effects, use fewer colors |
| "more eye-catching" | Increase color contrast, add glow effect, enlarge text, add accent elements |
| "different colors" | Swap entire color palette, recalculate text stroke color to complement new bg |
| "change the text" | Update text strings, re-run `fitText()` if text length changed significantly |
| "make text bigger" | Increase `fontSize`, may need to reduce word count or line breaks |
| "darker" / "moodier" | Darken background, add dark overlay, reduce brightness of accent colors |
| "brighter" / "lighter" | Lighten background, increase text color vibrancy, add glow effects |
| "more professional" | Switch to clean sans-serif, muted colors, increase whitespace, remove playful elements |
| "more fun" / "playful" | Switch to Bangers/novelty font, bright saturated colors, add emoji/stickers |
| "move text to [position]" | Change flexbox alignment/justification, adjust padding |
| "add a face/image" | Copy image to `public/`, add `<Img>` layer, adjust existing layout to accommodate |
| "remove the image" | Remove `<Img>` element, expand text area to fill space |
| "different font" | Change `loadFont` import, update `fontFamily` in styles, recheck sizing |
| "more contrast" | Increase difference between bg and text colors, thicken stroke, add shadow |
| "add a border" | Add `border` or `outline` CSS to the outermost container |
| "add a glow" | Add `filter: drop-shadow()` or `textShadow` glow values |
| "mobile friendly" | Increase font size, simplify layout, test at 160×90 scale |

## Quality Check After Each Iteration

After every re-render:
1. Verify file exists and is valid PNG
2. Check file size < 2MB
3. Visually confirm the change matches the request
4. Ensure no regression in text readability or layout
5. Confirm safe zones are still respected

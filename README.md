# creator-plugins

A curated collection of Claude Code plugins for creators and creative workflows.

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) version 1.0.33 or later (run `claude --version` to check)

## Installation

### Step 1: Add the marketplace

From within Claude Code, run:

```
/plugin marketplace add CEDARKEEP/creator-plugins
```

This registers the plugin catalog — no plugins are installed yet.

### Step 2: Install plugins

Option A — Command line:

```
/plugin install audio@creator-plugins
/plugin install images@creator-plugins
/plugin install copy@creator-plugins
/plugin install brand@creator-plugins
```

Option B — Interactive UI:

1. Run `/plugin` to open the plugin manager
2. Go to the Discover tab
3. Select a plugin and choose an installation scope:
   - User — available across all your projects
   - Project — shared with collaborators via `.claude/settings.json`
   - Local — only for you in the current repo

### Step 3: Use the skills

After installing, skills are available as slash commands:

```
/create-song a dreamy lo-fi beat with warm keys
/create-sfx laser blast impact
/create-soundscape rain on a tin roof at night
/create-thumbnail epic coding tutorial reaction
/transform-image resize logo.png to 200x200
/fetch-brand-assets Stripe
/create-title python tutorial for beginners YouTube
/create-description my latest cooking vlog Instagram
/create-hashtags fitness workout TikTok
```

### Updating

To pull the latest plugin versions:

```
/plugin marketplace update creator-plugins
```

Or enable auto-updates in the Marketplaces tab of `/plugin`.

### Uninstalling

```
/plugin uninstall audio@creator-plugins
/plugin marketplace remove creator-plugins
```

## Available Plugins

### Audio

Music production, sound effects, and audio synthesis skills.

| Command | Description |
|---------|-------------|
| `/create-song` | Generate original music as .wav from a vibe, genre, or mood description. 24+ genres, world music patterns, advanced synthesis, professional mastering, and parallel clip-based rendering for faster generation. |
| `/create-sfx` | Generate synthesized sound effects — UI sounds, combat, impacts, whooshes, transitions, foley, and game audio. Researches real-world acoustics before synthesizing for realistic results. |
| `/create-soundscape` | Generate ambient soundscapes and environmental audio — nature, meditation, spatial environments, atmospheric textures. Researches real environments for immersive, layered synthesis with parallel chunk rendering. |

Each audio command creates an organized project folder:

```
{name}/
├── scripts/    # Generated Python synthesis code
├── sounds/     # Individual section/chunk clips (songs & soundscapes)
└── {name}.wav  # Final output
```

### Images

Image generation and transformation skills — thumbnails, banners, social media graphics, resize, convert. Requires Node.js 18+ (for thumbnail generation).

| Command | Description |
|---------|-------------|
| `/create-thumbnail` | Generate YouTube thumbnail images as PNG files using Remotion. Designs high-CTR thumbnails with researched color palettes, typography, and proven archetype layouts. Supports face photos, product images, or pure typography designs. |
| `/transform-image` | Resize, convert, crop, enhance, and batch-process images. Generates a Python script using Pillow. Supports social media kits, favicon generation, format conversion, padding, trimming, and bulk operations. |

Thumbnail commands create a self-contained Remotion project:

```
{name}/
├── src/           # Remotion composition code (TypeScript/React)
├── public/        # User-provided assets (images, logos)
└── {name}.png     # Final output
```

### Brand

Brand asset sourcing — fetch official logos, icons, and visual identity assets from trusted sources.

| Command | Description |
|---------|-------------|
| `/fetch-brand-assets` | Fetch official product logos, brand marks, icons, and visual identity assets from trusted sources (press kits, brand portals, official CDNs). Downloads and organizes assets locally for use by other skills. |

### Copy

Copywriting skills for titles, descriptions, and hashtags — optimized for YouTube, Instagram, TikTok, Twitter/X, and LinkedIn.

| Command | Description |
|---------|-------------|
| `/create-title` | Generate optimized titles for YouTube videos and social media posts. Researches your niche, then produces 4 distinct title variations using proven formulas (How-To, Listicle, Question, Contrarian, etc.) with character counts and above-fold analysis. |
| `/create-description` | Generate optimized descriptions and captions for YouTube, Instagram, LinkedIn, TikTok, and Twitter/X. Produces 4 variations (comprehensive, hook-heavy, SEO-optimized, storytelling) with timestamps, CTAs, and above-fold previews. |
| `/create-hashtags` | Generate optimized hashtag sets using the pyramid strategy (broad + medium + niche). Produces 4 variations per platform with PascalCase formatting, copy-paste lines, and platform-specific count limits. |

Copy skills output pure text — no files or projects are created. Use all three in sequence for a complete copy package:
```
/create-title → pick a title → /create-description → pick a description → /create-hashtags
```

## Project Structure

```
creator-plugins/
├── .claude-plugin/
│   └── marketplace.json
├── audio/
│   ├── .claude-plugin/
│   │   └── plugin.json
│   ├── shared/
│   │   └── references/                      # Generic DSP/audio (shared by all skills)
│   │       ├── dsp-core.md                  # PolyBLEP oscillators, filters, envelopes
│   │       ├── effects.md                   # Freeverb, delay, chorus, compression
│   │       ├── synthesis-techniques.md      # Granular, physical modeling, modal, formant
│   │       ├── spectral-processing.md       # STFT, phase vocoder, Paulstretch
│   │       ├── production-techniques.md     # Multiband comp, transient shaper, mid/side
│   │       ├── sfx-synthesized.md           # Risers, impacts, glitch, tape stop, Doppler
│   │       ├── environmental-and-vocal.md   # Rain, wind, ocean, auto-tune, body perc
│   │       ├── mastering-and-export.md      # Pedalboard master chain, soundfile export
│   │       ├── studio-production.md         # Studio production techniques
│   │       ├── advanced-synthesis-dsp.md    # Advanced DSP and synthesis
│   │       ├── psychoacoustics.md           # Psychoacoustic principles
│   │       ├── mixing-core.md              # Stereo pipeline, panning, sidechain, EQ, master chain
│   │       ├── energy-framework.md         # 5-dimension energy system
│   │       ├── iteration-core.md           # Refinement workflow, version management
│   │       ├── automation-core.md          # LFO system, filter sweeps, modulation
│   │       └── quality-validation.md       # WAV validation, peak/clipping checks
│   └── skills/
│       ├── create-song/
│       │   ├── SKILL.md
│       │   ├── examples/
│       │   └── references/                  # Music-specific references
│       ├── create-sfx/
│       │   ├── SKILL.md
│       │   └── references/                  # SFX-specific references
│       └── create-soundscape/
│           ├── SKILL.md
│           └── references/                  # Soundscape-specific references
├── images/
│   ├── .claude-plugin/
│   │   └── plugin.json
│   ├── shared/
│   │   └── references/                      # Generic image/design (shared by all skills)
│   │       ├── remotion-patterns.md          # Still components, fonts, layout-utils, CLI
│   │       ├── color-theory.md               # Color pairs, psychology, gradients, palettes
│   │       ├── typography-fundamentals.md    # Font classification, sizing, text effects
│   │       ├── composition-principles.md     # Rule of thirds, hierarchy, safe zones
│   │       ├── iteration-core-images.md      # Image refinement workflow
│   │       └── quality-validation-images.md  # Dimension, size, readability checks
│   └── skills/
│       ├── create-thumbnail/
│       │   ├── SKILL.md
│       │   ├── examples/                      # Example thumbnail outputs
│       │   └── references/                    # Thumbnail-specific references
│       └── transform-image/
│           ├── SKILL.md
│           ├── examples/                      # Example transform outputs
│           └── references/                    # Transform-specific references
├── brand/
│   ├── .claude-plugin/
│   │   └── plugin.json
│   └── skills/
│       └── fetch-brand-assets/
│           ├── SKILL.md
│           ├── examples/                      # Example fetch outputs
│           └── references/                    # Brand asset references
├── copy/
│   ├── .claude-plugin/
│   │   └── plugin.json
│   ├── shared/
│   │   └── references/                      # Copywriting fundamentals (shared by all skills)
│   │       ├── platform-specs.md             # Character limits, above-fold, formatting rules
│   │       ├── copywriting-frameworks.md     # AIDA, PAS, Hook-Value-CTA, power words
│   │       ├── iteration-core-copy.md        # Copy refinement workflow
│   │       └── quality-validation-copy.md    # Char counts, readability, CTA checks
│   └── skills/
│       ├── create-title/
│       │   ├── SKILL.md
│       │   └── references/                    # Title-specific references
│       ├── create-description/
│       │   ├── SKILL.md
│       │   └── references/                    # Description-specific references
│       └── create-hashtags/
│           ├── SKILL.md
│           └── references/                    # Hashtag-specific references
├── .gitignore
├── LICENSE
└── README.md
```

## Contributing

To add a new skill:

1. Create a topic directory at the repo root if it doesn't exist (e.g. `audio/`)
2. Add your skill under `<topic>/skills/<skill-name>/` with a `SKILL.md` and any supporting files
3. Place reusable references in `<topic>/shared/references/` so other skills can share them
4. Register the plugin in `.claude-plugin/marketplace.json` by adding an entry to the `plugins` array
5. Submit a pull request

## License

MIT - see [LICENSE](LICENSE) for details.

# creator-plugins

A curated collection of Claude Code plugins for creators and creative workflows.

## Quick Start

```
/plugin marketplace add Cedarkeep/creator-plugins
```

## Available Plugins

### Music

Music production, sound effects, and audio synthesis skills.

| Skill | Description |
|-------|-------------|
| [create-song](music/skills/create-song/) | Generate original music as .wav from a vibe, genre, or mood description. 24+ genres, world music patterns, advanced synthesis, and professional mastering. |
| [create-sfx](music/skills/create-sfx/) | Generate synthesized sound effects — UI sounds, combat, impacts, whooshes, transitions, foley, and game audio. |
| [create-soundscape](music/skills/create-soundscape/) | Generate ambient soundscapes and environmental audio — nature, meditation, spatial environments, atmospheric textures. |

## Project Structure

```
creator-plugins/
├── .claude-plugin/
│   └── marketplace.json
├── music/
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
├── .gitignore
├── LICENSE
└── README.md
```

## Contributing

To add a new skill:

1. Create a topic directory at the repo root if it doesn't exist (e.g. `music/`)
2. Add your skill under `<topic>/skills/<skill-name>/` with a `SKILL.md` and any supporting files
3. Place reusable references in `<topic>/shared/references/` so other skills can share them
4. Register the plugin in `.claude-plugin/marketplace.json` by adding an entry to the `plugins` array
5. Submit a pull request

## License

MIT - see [LICENSE](LICENSE) for details.

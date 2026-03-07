# creator-plugins

A curated collection of Claude Code plugins for creators and creative workflows.

## Quick Start

```
/plugin marketplace add Cedarkeep/creator-plugins
```

## Available Plugins

### Music

| Skill | Description |
|-------|-------------|
| [create-song](music/skills/create-song/) | Generate original music as a .wav audio file from a vibe, genre, or mood description. Supports 24+ genres, world music patterns, advanced synthesis, SFX, and professional mastering via pedalboard. |

## Project Structure

```
creator-plugins/
├── .claude-plugin/
│   └── marketplace.json
├── music/
│   ├── .claude-plugin/
│   │   └── plugin.json
│   └── skills/
│       └── create-song/
│           ├── SKILL.md
│           ├── examples/
│           │   ├── atmospheric-xx.md
│           │   └── lofi-house.md
│           └── references/
│               ├── dsp-core.md              # PolyBLEP oscillators, filters, envelopes
│               ├── effects.md               # Freeverb, delay, chorus, compression
│               ├── instruments.md           # Drum + synth instrument recipes
│               ├── music-theory.md          # Scales, chords, voice leading, melody
│               ├── rhythm-and-groove.md     # Genre drum patterns, swing, humanization
│               ├── genre-guide.md           # BPM/mode/scale quick-reference table
│               ├── drum-patterns-world.md   # Latin, Afro-Cuban, Brazilian, ME, Indian
│               ├── drum-patterns-breaks.md  # Breakbeats, ghost notes, fills
│               ├── chord-progressions.md    # Neo-soul, gospel, film, game, bass, arps
│               ├── melody-and-structure.md  # Melody data, song templates, JSON schemas
│               ├── synthesis-techniques.md  # Granular, physical modeling, modal, formant
│               ├── spectral-processing.md   # STFT, phase vocoder, Paulstretch
│               ├── production-techniques.md # Multiband comp, transient shaper, mid/side
│               ├── sfx-synthesized.md       # Risers, impacts, glitch, tape stop, Doppler
│               ├── environmental-and-vocal.md # Rain, wind, ocean, auto-tune, body perc
│               ├── mixing.md                # Stereo pipeline, EQ, arrangement
│               ├── mastering-and-export.md  # Pedalboard master chain, soundfile export
│               └── mixing-and-mastering.md  # Combined mixing & mastering reference
├── .gitignore
├── LICENSE
└── README.md
```

## Contributing

To add a new skill:

1. Create a topic directory at the repo root if it doesn't exist (e.g. `music/`)
2. Add your skill under `<topic>/skills/<skill-name>/` with a `SKILL.md` and any supporting files
3. Register the plugin in `.claude-plugin/marketplace.json` by adding an entry to the `plugins` array
4. Submit a pull request

## License

MIT - see [LICENSE](LICENSE) for details.

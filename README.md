<!-- readme.md -->
# Visage UI Cookbook

Tips for pairing [Visage](https://github.com/VitalAudio/visage) — a lightweight, GPU-accelerated C++ UI framework — with [JUCE](https://github.com/juce-framework/JUCE) audio plugins.

> **Note**: This cookbook predates the [`juce-visage` Claude skill](https://github.com/danielraffel/generous-corp-marketplace/tree/master/skills/juce-visage) and the [`juce-dev` plugin](https://github.com/danielraffel/generous-corp-marketplace/tree/master/plugins/juce-dev), which are actively maintained and more accurate. **Use those instead for new projects.**

## Recommended Resources

| Resource | What it covers | Status |
|----------|---------------|--------|
| **[`juce-visage` skill](https://github.com/danielraffel/generous-corp-marketplace/tree/master/skills/juce-visage)** | Production-tested JUCE+Visage integration patterns: Metal embedding, event bridging, DAW keyboard handling, destruction ordering, popup/modal/dropdown systems, required Visage patches | **Actively maintained** |
| **[`juce-dev` plugin](https://github.com/danielraffel/generous-corp-marketplace/tree/master/plugins/juce-dev)** | Automates creating new JUCE plugin projects from a template, with optional Visage integration | **Actively maintained** |
| **[`danielraffel/visage` fork](https://github.com/danielraffel/visage)** | Visage with production patches applied (performKeyEquivalent for plugin text editing, Cmd+Q propagation, 60 FPS cap, popup overflow fix) | **Actively maintained** |
| **`visage_prompt.md`** (this repo) | Early migration guide for JUCE to Visage | **Outdated — see warning below** |

## About `visage_prompt.md`

`visage_prompt.md` was an early attempt at a comprehensive JUCE-to-Visage migration guide, generated before the skill and plugin existed. **It has known accuracy issues:**

- Many code examples use API calls that don't exist in Visage (`canvas.save()`, `canvas.translate()`, `canvas.clipRect()`, etc.)
- Effect class names are wrong (`BlurEffect` should be `BlurPostEffect`, etc.)
- Several "effect" classes are fabricated (`DropShadowEffect`, `FilmGrainEffect`, `ChromaticAberrationEffect`)
- Critical production patterns are missing (destruction ordering, Visage patches, focus management, AU/VST3 startup optimization)

The general concepts (Frame vs Component, draw vs paint, migration priorities) are valid, but **don't trust the code examples without verifying against the [actual Visage source](https://github.com/VitalAudio/visage)**.

## Contributing

MIT licensed. If you'd like to help create an accurate API reference by verifying examples against the Visage source code, PRs are welcome.

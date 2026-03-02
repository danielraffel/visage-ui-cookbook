<!-- readme.md -->
# Visage UI Cookbook

API reference and migration guide for [Visage](https://github.com/VitalAudio/visage) — a lightweight, GPU-accelerated C++ UI framework — with [JUCE](https://github.com/juce-framework/JUCE) audio plugins.

> All code examples in `visage_prompt.md` have been verified against the actual Visage source code ([danielraffel/visage](https://github.com/danielraffel/visage) patched fork).

## Which Resource Should You Use?

**`visage_prompt.md`** (this repo) is an **AI-agnostic API reference** — it works with any AI assistant, IDE, or as a standalone developer reference. Use it to look up Visage API signatures, understand the architecture, or translate JUCE patterns to Visage.

If you use **Claude Code**, two additional resources provide deeper automation:

| Resource | What it is | Who it's for |
|----------|-----------|--------------|
| **`visage_prompt.md`** (this repo) | Verified Visage API reference and JUCE migration guide | Any developer or AI assistant |
| **[`juce-visage` skill](https://github.com/danielraffel/generous-corp-marketplace/tree/master/skills/juce-visage)** | Claude Code skill: Metal embedding, event bridging, DAW keyboard handling, destruction ordering, popup/modal/dropdown systems | Claude Code users building JUCE plugins with Visage |
| **[JUCE-Plugin-Starter](https://github.com/danielraffel/JUCE-Plugin-Starter)** | Claude Code-friendly project template: build automation, code signing, notarization, auto-versioning, installer generation | Claude Code users starting a new JUCE plugin from scratch |
| **[`danielraffel/visage` fork](https://github.com/danielraffel/visage)** | Visage with production patches (plugin text editing, Cmd+Q, 60 FPS cap, popup fix) | Anyone using Visage in DAW plugins |

**TL;DR**: Start here for API reference. Add `juce-visage` skill for Claude Code JUCE+Visage work. Use JUCE-Plugin-Starter for new plugin projects with Claude Code.

## What's in `visage_prompt.md`

- Architecture overview (Frame tree, Canvas drawing, GPU rendering, Palette/Theme)
- Complete Frame API reference (lifecycle, mouse, keyboard, children, bounds, layout, effects)
- Complete Canvas API reference (shapes, text, images, shaders, color/brush, state)
- Color, Brush & Theme system
- Font & Text system
- PostEffect system (Blur, Bloom, Shader — only the real ones)
- Widget classes (Button, PopupMenu, ScrollableFrame, TextEditor)
- Window & Application (standalone and plugin embedding)
- Event system (MouseEvent, KeyEvent, EventTimer)
- Dimension system (_px, _vw, _vh, _vmin, _vmax)
- JUCE-to-Visage migration patterns with side-by-side tables
- Plugin integration essentials (Metal embedding, destruction ordering, DAW keyboard handling)
- Build system (CMake, file embedding)

## Contributing

MIT licensed. PRs welcome.

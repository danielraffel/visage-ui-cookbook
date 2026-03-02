<!-- readme.md -->
# Visage UI Cookbook

API reference and migration guide for [Visage](https://github.com/VitalAudio/visage) — a lightweight, GPU-accelerated C++ UI framework — with [JUCE](https://github.com/juce-framework/JUCE) audio plugins.

> All code examples in `visage_prompt.md` have been verified against the actual Visage source code ([danielraffel/visage](https://github.com/danielraffel/visage) patched fork).

## Which Resource Should You Use?

| Resource | What it is | Who it's for |
|----------|-----------|--------------|
| **[`visage_prompt.md`](visage_prompt.md)** | AI-agnostic Visage API reference and JUCE migration guide | Any developer or AI assistant working with Visage |
| **[`juce-visage` skill](https://github.com/danielraffel/generous-corp-marketplace/tree/master/skills/juce-visage)** | Claude Code skill: Metal embedding, event bridging, DAW keyboard handling, destruction ordering, popup/modal/dropdown systems | Claude Code users building macOS JUCE plugins with Visage |
| **[JUCE-Plugin-Starter](https://github.com/danielraffel/JUCE-Plugin-Starter)** | Claude Code-friendly project template: build automation, code signing, notarization, auto-versioning, installer generation | Claude Code users starting a new JUCE plugin from scratch |
| **[`danielraffel/visage` fork](https://github.com/danielraffel/visage)** | Visage with macOS-specific patches for JUCE plugin hosting (see PRs below) | Anyone embedding Visage in a macOS JUCE plugin |

### Patches in the `danielraffel/visage` fork

These are macOS-specific fixes for running Visage inside DAW-hosted JUCE plugins (AU/VST3):

- [#81 — Fix keyboard handling when Visage is hosted inside a plugin](https://github.com/VitalAudio/visage/pull/81) — `performKeyEquivalent:` intercepts Cmd+A/C/V/X/Z so DAW menus don't steal them from text fields
- [#83 — Cap MTKView to 60 FPS](https://github.com/VitalAudio/visage/pull/83) — prevents excessive CPU/GPU usage from uncapped Metal render loop
- [#79 — Fix popup menu positioning when menu overflows below window](https://github.com/VitalAudio/visage/pull/79) — positions overflow menus above the source frame instead of at screen top
- [#80 — Only call setAlwaysOnTop in showWindow when explicitly enabled](https://github.com/VitalAudio/visage/pull/80) — fixes window stacking issues in plugin hosts
- [#82 — Navigate to start/end on Up/Down in single-line text fields](https://github.com/VitalAudio/visage/pull/82) — Home/End-style navigation via arrow keys
- [#84 — Log instead of assert in InstanceCounter destructor](https://github.com/VitalAudio/visage/pull/84) — prevents debug crashes during plugin teardown
- [#51 — Enable touch and pinch gesture support on macOS](https://github.com/VitalAudio/visage/pull/51) — trackpad gesture handling

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

# Visage UI Cookbook

A collection of _lightly_ field-tested tips and guidance for working with [Visage](https://github.com/VitalAudio/visage) — a C++ UI toolkit optimized for real-time audio and plugin UIs — especially when used alongside [JUCE]( https://github.com/juce-framework/JUCE) AI tools like Claude Code.

## Why Visage?

Visage is a lightweight, retained-mode graphics framework built for high-performance rendering and flexibility in plugin UI design. It's particularly useful when:

- You want to **separate rendering from interaction logic**
- You need **OpenGL-backed performance** for scalable visuals
- You’re building **custom, modern UIs** beyond JUCE’s traditional look
- You want clean **layout control and animation support**

By integrating Visage with JUCE, you can take advantage of JUCE’s audio engine, plugin wrapper support, and platform abstractions — while using Visage for advanced UI rendering and layout.

## What's Inside

This repo contains a curated markdown file for developers exploring or integrating Visage into their C++ projects. Each file addresses specific areas, like:

- Fixing build warnings (e.g., integer truncation)
- Integrating Visage inside JUCE components
- Best practices for layout, animation, and draw loops
- Debugging and profiling tips

### Files

- [visage_prompt.md](./visage_prompt.md): Guidance for an AI integrating the Visage codebase with JUCE.

## License

MIT — use freely and contribute if you’d like.

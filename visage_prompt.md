# Visage UI Framework Migration Guide for AI Assistants

*A comprehensive guide for AI assistants helping developers migrate from JUCE to Visage or create new Visage UIs*

## Overview

Visage is a modern GPU-accelerated UI framework using bgfx for cross-platform rendering. This guide helps AI assistants understand key differences from JUCE and provide effective migration assistance.

## Quick Start - Minimal Working Example

```cpp
// MinimalVisagePlugin.h - Start here!
class MinimalVisagePlugin : public juce::AudioProcessor {
public:
    juce::AudioProcessorEditor* createEditor() override {
        return new MinimalVisageEditor(*this);
    }
};

// MinimalVisageEditor.h
class MinimalVisageEditor : public juce::AudioProcessorEditor {
    class VisageComponent : public juce::Component {
        std::unique_ptr<visage::ApplicationWindow> visageWindow;
        
    public:
        VisageComponent() {
            setOpaque(true);
        }
        
        void paint(juce::Graphics& g) override {
            // Step 1: Get native window handle
            if (auto* peer = getPeer()) {
                void* nativeHandle = peer->getNativeHandle();
                
                // Step 2: Initialize Visage if needed
                if (!visageWindow) {
                    visageWindow = createPluginWindow(getWidth(), getHeight(), nativeHandle);
                    visage::Renderer::instance().checkInitialization(
                        visageWindow->initWindow(), 
                        visageWindow->globalDisplay()
                    );
                }
                
                // Step 3: Simple Visage rendering
                visage::Canvas canvas;
                canvas.pairToWindow(visageWindow->nativeHandle(), getWidth(), getHeight());
                canvas.setColor(0xff0080ff); // Blue
                canvas.fill(0, 0, getWidth(), getHeight());
                canvas.submit();
            }
        }
        
        void resized() override {
            if (visageWindow) {
                // Update Visage window size
                visageWindow->resize(getWidth(), getHeight());
            }
        }
    };
    
    VisageComponent visageComponent;
    
public:
    MinimalVisageEditor(juce::AudioProcessor& p) 
        : AudioProcessorEditor(p) {
        addAndMakeVisible(visageComponent);
        setSize(400, 300);
    }
    
    void resized() override {
        visageComponent.setBounds(getLocalBounds());
    }
};
```

## Core Architecture Differences

### Component Hierarchy
- **JUCE**: `Component` base class with `addAndMakeVisible()`
- **Visage**: `Frame` base class with `addChild()` or `addSubFrame()`
- **Key**: Visage uses smart pointers and explicit parent-child relationships

```cpp
// JUCE
addAndMakeVisible(button);

// Visage
button = std::make_unique<visage::Button>();
addChild(button.get());
// or
addChild(std::move(button));
```

### Drawing System
- **JUCE**: `paint(Graphics& g)` method
- **Visage**: `draw(Canvas& canvas)` method with GPU acceleration
- **Key**: Canvas uses immediate mode rendering with state management

```cpp
// JUCE
void paint(Graphics& g) override {
    g.setColour(Colours::blue);
    g.fillRect(bounds);
}

// Visage
void draw(Canvas& canvas) override {
    canvas.setColor(0xff0000ff);
    canvas.fill(0, 0, width(), height());
}
```

### Event Handling
- **JUCE**: Virtual methods (`mouseDown`, `keyPressed`) and listener interfaces
- **Visage**: Lambda callbacks and virtual methods
- **Key**: Visage callbacks are more functional programming oriented

```cpp
// JUCE
button.onClick = [this] { handleClick(); };

// Visage
button->onMouseDown() += [this](const MouseEvent& e) { handleClick(); };
```

## Critical Plugin Integration Patterns

### Window Embedding (Most Important)
For plugins, **never** use `ApplicationWindow` - use the bridge pattern:

```cpp
// CORRECT: Plugin window embedding
auto window = createPluginWindow(width, height, hostParentHandle);
visage::Renderer::instance().checkInitialization(window->initWindow(), window->globalDisplay());
canvas.pairToWindow(window->nativeHandle(), width, height);

// WRONG: Using ApplicationWindow in plugins
auto appWindow = std::make_unique<visage::ApplicationWindow>();
appWindow->show(hostParentHandle); // This will fail
```

### Platform-Specific Handles
- **Windows**: `HWND` with `WS_CHILD` flag
- **macOS**: `NSView*` added as subview to parent
- **Linux**: X11 window with parent window ID

### Core Integration Bridge

```cpp
// JuceVisageBridge.h
class JuceVisageBridge : public juce::Component,
                        public visage::Frame {
public:
    JuceVisageBridge() {
        // Initialize Visage graphics context
        visage_canvas_ = std::make_unique<visage::Canvas>();
        
        // Set up proper lifecycle coordination
        setOpaque(true);
        setBufferedToImage(false); // Avoid JUCE's software buffering
    }
    
    // JUCE Component overrides
    void paint(juce::Graphics& g) override {
        // Delegate to Visage rendering
        renderVisageContent(g);
    }
    
    void resized() override {
        // Coordinate sizing between JUCE and Visage
        auto bounds = getLocalBounds();
        visage_canvas_->setDimensions(bounds.getWidth(), bounds.getHeight());
        setBounds(bounds.getX(), bounds.getY(), bounds.getWidth(), bounds.getHeight());
        
        // Trigger Visage layout
        computeLayout();
    }
    
    // Visage Frame overrides
    void draw(visage::Canvas& canvas) override {
        // Your Visage UI rendering here
        drawVisageUI(canvas);
    }
    
private:
    void renderVisageContent(juce::Graphics& juceGraphics) {
        // Get the native graphics context from JUCE
        auto nativeContext = extractNativeContext(juceGraphics);
        
        // Initialize Visage renderer with native context
        if (!visage_initialized_) {
            initializeVisageRenderer(nativeContext);
            visage_initialized_ = true;
        }
        
        // Submit Visage rendering commands
        int submit_pass = 0;
        submit_pass = visage_canvas_->submit(submit_pass);
    }
    
    std::unique_ptr<visage::Canvas> visage_canvas_;
    bool visage_initialized_ = false;
};
```

## Missing Visage Components (Create These)

### Label Component
Visage doesn't have a built-in Label. Create one:

```cpp
class VisageLabel : public visage::Frame {
public:
    void setText(const std::string& text) { text_ = text; redraw(); }
    void draw(visage::Canvas& canvas) override {
        visage::String visageText(text_.c_str());
        canvas.text(visageText, *font_, justification_, 0, 0, width(), height());
    }
private:
    std::string text_;
    std::unique_ptr<visage::Font> font_;
    visage::Font::Justification justification_ = visage::Font::kLeft;
};
```

### Production-Ready Knob Component

```cpp
// VisageKnob.h - Production-ready knob component
class VisageKnob : public visage::Frame, 
                   public juce::Timer {
private:
    float value_ = 0.0f;
    float targetValue_ = 0.0f;
    float animationSpeed_ = 0.1f;
    
    // Visual state
    bool isHovered_ = false;
    bool isDragging_ = false;
    float glowAmount_ = 0.0f;
    
    // Drag handling
    visage::Point dragStartPos_;
    float dragStartValue_ = 0.0f;
    
    // Callbacks
    std::function<void(float)> onValueChanged_;
    
public:
    VisageKnob() {
        startTimerHz(60); // 60 FPS animation
    }
    
    void setValue(float newValue, bool animate = true) {
        targetValue_ = std::clamp(newValue, 0.0f, 1.0f);
        if (!animate) {
            value_ = targetValue_;
            if (onValueChanged_) onValueChanged_(value_);
        }
    }
    
    void draw(visage::Canvas& canvas) override {
        const float centerX = width() / 2.0f;
        const float centerY = height() / 2.0f;
        const float radius = std::min(width(), height()) * 0.4f;
        
        // Animated glow effect
        if (isHovered_ || isDragging_) {
            canvas.pushEffect(std::make_unique<visage::BloomEffect>(glowAmount_));
        }
        
        // Background circle
        canvas.setColor(0xff202020);
        canvas.circle(centerX, centerY, radius);
        
        // Value arc
        const float startAngle = -135.0f * (M_PI / 180.0f);
        const float endAngle = 135.0f * (M_PI / 180.0f);
        const float valueAngle = startAngle + (endAngle - startAngle) * value_;
        
        canvas.setColor(0xff00a0ff);
        canvas.setLineWidth(3.0f);
        canvas.arc(centerX, centerY, radius * 0.8f, startAngle, valueAngle);
        
        // Center dot
        canvas.setColor(isDragging_ ? 0xffffffff : 0xffa0a0a0);
        canvas.circle(centerX, centerY, radius * 0.15f);
        
        // Value text
        auto* font = FontManager::instance().getFont(
            FontManager::FontType::Regular, 12.0f);
        std::string valueText = std::to_string(static_cast<int>(value_ * 100)) + "%";
        visage::String visageText(valueText.c_str());
        canvas.setColor(0xffffffff);
        canvas.text(visageText, *font, visage::Font::kCenter, 
                   0, height() - 20, width(), 20);
        
        if (isHovered_ || isDragging_) {
            canvas.popEffect();
        }
    }
    
    void onMouseEnter(const visage::MouseEvent& e) override {
        isHovered_ = true;
        redraw();
    }
    
    void onMouseExit(const visage::MouseEvent& e) override {
        isHovered_ = false;
        redraw();
    }
    
    void onMouseDown(const visage::MouseEvent& e) override {
        isDragging_ = true;
        dragStartPos_ = e.position;
        dragStartValue_ = value_;
        redraw();
    }
    
    void onMouseDrag(const visage::MouseEvent& e) override {
        if (isDragging_) {
            // Vertical drag for value change
            float deltaY = (dragStartPos_.y - e.position.y) / 100.0f;
            setValue(dragStartValue_ + deltaY);
            redraw();
        }
    }
    
    void onMouseUp(const visage::MouseEvent& e) override {
        isDragging_ = false;
        redraw();
    }
    
    void timerCallback() override {
        // Smooth value animation
        if (std::abs(value_ - targetValue_) > 0.001f) {
            value_ += (targetValue_ - value_) * animationSpeed_;
            if (onValueChanged_) onValueChanged_(value_);
            redraw();
        }
        
        // Glow animation
        float targetGlow = (isHovered_ || isDragging_) ? 0.5f : 0.0f;
        if (std::abs(glowAmount_ - targetGlow) > 0.001f) {
            glowAmount_ += (targetGlow - glowAmount_) * 0.1f;
            redraw();
        }
    }
};
```

### Drag and Drop System
Visage has basic drag/drop but needs custom implementation for complex scenarios:

```cpp
class VisageDragDropManager {
public:
    void startDrag(Frame* source, const std::string& data);
    void updateDragPosition(Point position);
    void finishDrag(Frame* target);
private:
    Frame* dragSource_ = nullptr;
    std::unique_ptr<Frame> dragPreview_;
    std::string dragData_;
};
```

## API Translation Patterns

### Bounds and Positioning
```cpp
// JUCE
auto bounds = getLocalBounds();
setBounds(x, y, width, height);

// Visage
auto bounds = localBounds(); // or use width()/height()
setBounds(x, y, width, height); // Same signature
```

### Colors and Theming
```cpp
// JUCE
g.setColour(Colours::blue);

// Visage
canvas.setColor(0xff0000ff); // ARGB hex
// or use theme system
canvas.setColor(theme.button.normal);
```

### Mouse Events
```cpp
// JUCE MouseEvent
event.getMouseDownX(), event.getMouseDownY()
event.mods.isLeftButtonDown()

// Visage MouseEvent  
event.position.x, event.position.y
event.button_state & kMouseButtonLeft // Note: button_state not buttons
```

### Text and Fonts
```cpp
// JUCE
g.drawText(text, bounds, Justification::centred);

// Visage
visage::String visageText(text.c_str());
canvas.text(visageText, *font, visage::Font::kCenter, x, y, width, height);
```

## Font System Implementation

### Font Embedding Strategy
1. Convert TTF files to C++ headers using Python script
2. Create font manager with size-based caching
3. Use Visage's Font constructor with embedded data

```cpp
// Font loading
auto font = std::make_unique<visage::Font>(size, fontData, dataSize);

// Text rendering with proper string conversion
visage::String visageText(text.c_str());
canvas.text(visageText, *font, visage::Font::kCenter, x, y, width, height);
```

### Font Embedding Process

#### Step 1: Convert TTF/OTF to C++ Headers
```python
# font_converter.py
import os
import sys

def convert_font_to_header(font_path, output_path):
    """Convert font file to C++ header with binary data"""
    font_name = os.path.splitext(os.path.basename(font_path))[0]
    
    with open(font_path, 'rb') as f:
        font_data = f.read()
    
    with open(output_path, 'w') as f:
        f.write(f"// Auto-generated from {font_path}\n")
        f.write(f"#pragma once\n\n")
        f.write(f"static const unsigned char {font_name}_data[] = {{\n")
        
        # Write bytes in hex format
        for i, byte in enumerate(font_data):
            if i % 16 == 0:
                f.write("    ")
            f.write(f"0x{byte:02x}, ")
            if (i + 1) % 16 == 0:
                f.write("\n")
        
        f.write("\n};\n\n")
        f.write(f"static const size_t {font_name}_size = {len(font_data)};\n")
```

#### Step 2: Font Manager Implementation
```cpp
// FontManager.h
class FontManager {
public:
    static FontManager& instance() {
        static FontManager instance;
        return instance;
    }
    
    // Cache fonts by size to avoid recreation
    visage::Font* getFont(FontType type, float size) {
        FontKey key{type, size};
        
        auto it = fontCache_.find(key);
        if (it != fontCache_.end()) {
            return it->second.get();
        }
        
        // Create new font
        auto font = createFont(type, size);
        auto* fontPtr = font.get();
        fontCache_[key] = std::move(font);
        return fontPtr;
    }
    
    // Support for different font weights/styles
    enum class FontType {
        Regular,
        Bold,
        Italic,
        Light,
        Medium,
        // Icon fonts
        MaterialIcons,
        FontAwesome
    };
    
private:
    struct FontKey {
        FontType type;
        float size;
        
        bool operator<(const FontKey& other) const {
            return std::tie(type, size) < std::tie(other.type, other.size);
        }
    };
    
    std::map<FontKey, std::unique_ptr<visage::Font>> fontCache_;
    
    std::unique_ptr<visage::Font> createFont(FontType type, float size) {
        switch (type) {
            case FontType::Regular:
                return std::make_unique<visage::Font>(size, 
                    roboto_regular_data, roboto_regular_size);
            case FontType::Bold:
                return std::make_unique<visage::Font>(size, 
                    roboto_bold_data, roboto_bold_size);
            case FontType::MaterialIcons:
                return std::make_unique<visage::Font>(size, 
                    material_icons_data, material_icons_size);
            // ... other font types
        }
    }
};
```

### Text Rendering Patterns

#### Basic Text Rendering
```cpp
void drawText(visage::Canvas& canvas, const std::string& text, 
              float x, float y, float width, float height) {
    // CRITICAL: Always convert std::string to visage::String
    visage::String visageText(text.c_str());
    
    auto* font = FontManager::instance().getFont(
        FontManager::FontType::Regular, 14.0f);
    
    canvas.text(visageText, *font, 
        visage::Font::kCenter,  // Justification
        x, y, width, height);   // Bounds
}
```

#### Advanced Text Rendering with State
```cpp
class TextRenderer {
public:
    struct TextStyle {
        FontManager::FontType fontType = FontManager::FontType::Regular;
        float fontSize = 14.0f;
        uint32_t color = 0xffffffff;
        visage::Font::Justification justification = visage::Font::kLeft;
        float lineHeight = 1.2f;
        float letterSpacing = 0.0f;
    };
    
    void drawStyledText(visage::Canvas& canvas, 
                       const std::string& text,
                       const TextStyle& style,
                       float x, float y, float width, float height) {
        auto* font = FontManager::instance().getFont(style.fontType, style.fontSize);
        
        canvas.save();
        canvas.setColor(style.color);
        
        // Handle multi-line text manually if needed
        if (text.find('\n') != std::string::npos) {
            drawMultilineText(canvas, text, style, font, x, y, width, height);
        } else {
            visage::String visageText(text.c_str());
            canvas.text(visageText, *font, style.justification, x, y, width, height);
        }
        
        canvas.restore();
    }
    
private:
    void drawMultilineText(visage::Canvas& canvas,
                          const std::string& text,
                          const TextStyle& style,
                          visage::Font* font,
                          float x, float y, float width, float height) {
        std::istringstream stream(text);
        std::string line;
        float lineY = y;
        float lineHeight = style.fontSize * style.lineHeight;
        
        while (std::getline(stream, line)) {
            visage::String visageLine(line.c_str());
            canvas.text(visageLine, *font, style.justification, 
                       x, lineY, width, lineHeight);
            lineY += lineHeight;
        }
    }
};
```

### Icon Font Support
```cpp
// MaterialIconsHelper.h
namespace MaterialIcons {
    // Define icon constants as UTF-8 strings
    constexpr const char* SETTINGS = "\xEE\x8B\x8D";
    constexpr const char* PLAY_ARROW = "\xEE\x80\xB7";
    constexpr const char* PAUSE = "\xEE\x80\xB8";
    constexpr const char* STOP = "\xEE\x80\xC7";
    
    void drawIcon(visage::Canvas& canvas, const char* icon, 
                  float x, float y, float size, uint32_t color) {
        auto* font = FontManager::instance().getFont(
            FontManager::FontType::MaterialIcons, size);
        
        canvas.setColor(color);
        visage::String iconString(icon);
        canvas.text(iconString, *font, visage::Font::kCenter,
                   x, y, size, size);
    }
}
```

### Font Metrics and Layout
```cpp
class TextLayoutHelper {
public:
    struct TextMetrics {
        float width;
        float height;
        float ascent;
        float descent;
    };
    
    // Since Visage doesn't provide text metrics, approximate them
    static TextMetrics measureText(const std::string& text, 
                                  visage::Font* font,
                                  float fontSize) {
        // Approximations based on font size
        // These ratios work for most fonts
        TextMetrics metrics;
        metrics.height = fontSize;
        metrics.ascent = fontSize * 0.8f;
        metrics.descent = fontSize * 0.2f;
        
        // Approximate width (monospace assumption)
        // For proportional fonts, you'd need a character width table
        float avgCharWidth = fontSize * 0.6f;
        metrics.width = text.length() * avgCharWidth;
        
        return metrics;
    }
    
    // Handle text truncation with ellipsis
    static std::string truncateText(const std::string& text,
                                   float maxWidth,
                                   visage::Font* font,
                                   float fontSize) {
        auto metrics = measureText(text, font, fontSize);
        if (metrics.width <= maxWidth) {
            return text;
        }
        
        const std::string ellipsis = "...";
        auto ellipsisMetrics = measureText(ellipsis, font, fontSize);
        float availableWidth = maxWidth - ellipsisMetrics.width;
        
        // Binary search for the right truncation point
        size_t low = 0, high = text.length();
        while (low < high) {
            size_t mid = (low + high + 1) / 2;
            auto truncated = text.substr(0, mid);
            auto truncMetrics = measureText(truncated, font, fontSize);
            
            if (truncMetrics.width <= availableWidth) {
                low = mid;
            } else {
                high = mid - 1;
            }
        }
        
        return text.substr(0, low) + ellipsis;
    }
};
```

### Font Loading Best Practices

```cpp
// FontLoader.h - Lazy loading pattern
class FontLoader {
public:
    // Load fonts on-demand to reduce memory usage
    static void preloadEssentialFonts() {
        // Only preload the most commonly used font/size combinations
        FontManager::instance().getFont(FontManager::FontType::Regular, 14.0f);
        FontManager::instance().getFont(FontManager::FontType::Bold, 16.0f);
    }
    
    // For development: hot-reload fonts
    #if DEBUG
    static void reloadFonts() {
        // Clear font cache and reload from disk
        // Useful during development for testing different fonts
    }
    #endif
};
```

## Theme System Best Practices

### JSON-Based Themes
Create extensible theme system with hot-reload for development:

```cpp
// Theme structure
struct ColorDef {
    Color normal, hover, active, disabled;
};

// Usage
auto& theme = getThemeManager().getCurrentTheme();
canvas.setColor(isMouseOver ? theme.button.hover : theme.button.normal);

// Hot reload in debug builds
#if DEBUG
getThemeManager().enableHotReload(true);
#endif
```

### State-Based Colors
Support multiple color formats in JSON:
```json
{
  "button": {
    "normal": "#333333",
    "hover": "#3d3d3d",
    "active": "#2a2a2a", 
    "disabled": "rgba(51, 51, 51, 0.5)"
  }
}
```

## Common Migration Pitfalls

### 1. Namespace Conflicts
- Remove Cocoa.h imports that conflict with visage::Point/String
- Use forward declarations for platform types

### 2. String Types
- `visage::String` is not std::string - needs conversion
- TextEditor returns visage::String, not std::u32string

### 3. Event API Differences
- MouseEvent: `button_state` not `buttons`
- MouseEvent: `wheel_delta_x/y` not `wheelDelta`
- KeyEvent constructor: `(key, mods, is_down, repeat)`

### 4. Initialization Order
```cpp
// CORRECT: Initialize before setting palette
rootFrame = std::make_unique<visage::Frame>();
rootFrame->init();
rootFrame->setPalette(&getDefaultPalette());

// WRONG: Setting palette before init
rootFrame->setPalette(&getDefaultPalette());
rootFrame->init(); // May crash
```

### 5. Missing Methods
- No `setRootFrame()` on ApplicationWindow
- No `getLocalBounds()` - use `bounds()` or `width()/height()`
- No `toFront()` - use `setOnTop(true)`

### 6. Effect-Related Pitfalls

#### Effect Memory Management
```cpp
// WRONG: Creating effects every frame
void draw(visage::Canvas& canvas) override {
    canvas.pushEffect(std::make_unique<visage::BlurEffect>(5.0f)); // Memory allocation!
    drawContent(canvas);
    canvas.popEffect();
}

// CORRECT: Reuse effect instances
class MyComponent : public visage::Frame {
    std::unique_ptr<visage::BlurEffect> blur_effect_;
    
    MyComponent() {
        blur_effect_ = std::make_unique<visage::BlurEffect>(5.0f);
    }
    
    void draw(visage::Canvas& canvas) override {
        canvas.pushEffect(blur_effect_.get());
        drawContent(canvas);
        canvas.popEffect();
    }
};
```

#### Effect Stacking Order
```cpp
// WRONG: Effects applied in wrong order
canvas.pushEffect(blur_effect_.get());    // Blur first
canvas.pushEffect(shadow_effect_.get());  // Then shadow - shadow gets blurred!

// CORRECT: Logical effect order
canvas.pushEffect(shadow_effect_.get());  // Shadow first
canvas.pushEffect(blur_effect_.get());    // Then blur content only
```

#### Platform-Specific Effect Support
```cpp
// WRONG: Assuming all effects work on all platforms
auto effect = std::make_unique<ComputeShaderEffect>();
canvas.pushEffect(effect.get()); // May fail on older GPUs

// CORRECT: Check platform capabilities
if (visage::Renderer::instance().supportsComputeShaders()) {
    canvas.pushEffect(compute_effect_.get());
} else {
    canvas.pushEffect(fallback_effect_.get());
}
```

#### Effect Performance Assumptions
```cpp
// WRONG: Too many high-cost effects
void draw(visage::Canvas& canvas) override {
    canvas.pushEffect(blur_effect_.get());        // 2ms
    canvas.pushEffect(bloom_effect_.get());       // 3ms
    canvas.pushEffect(distortion_effect_.get());  // 2ms
    canvas.pushEffect(color_grade_effect_.get()); // 1ms
    // Total: 8ms just for effects!
}

// CORRECT: Profile and limit effects
void draw(visage::Canvas& canvas) override {
    if (high_quality_mode_) {
        canvas.pushEffect(bloom_effect_.get());
        canvas.pushEffect(color_grade_effect_.get());
    } else {
        canvas.pushEffect(color_grade_effect_.get()); // Just color grading
    }
    drawContent(canvas);
    // Pop effects...
}
```

#### Effect State Management
```cpp
// WRONG: Not saving/restoring canvas state
void draw(visage::Canvas& canvas) override {
    canvas.setBlendMode(visage::BlendMode::Add);
    canvas.pushEffect(glow_effect_.get());
    drawContent(canvas);
    canvas.popEffect();
    // Blend mode still set to Add!
}

// CORRECT: Save and restore state
void draw(visage::Canvas& canvas) override {
    canvas.save();
    canvas.setBlendMode(visage::BlendMode::Add);
    canvas.pushEffect(glow_effect_.get());
    drawContent(canvas);
    canvas.popEffect();
    canvas.restore(); // Blend mode restored
}
```

#### JUCE Graphics Context Mixing
```cpp
// WRONG: Trying to use JUCE Graphics with Visage effects
void paint(juce::Graphics& g) override {
    visage::Canvas canvas;
    canvas.pushEffect(blur_effect_.get());
    g.fillRect(bounds); // Won't get the effect!
}

// CORRECT: Use either JUCE or Visage, not both
void draw(visage::Canvas& canvas) override {
    canvas.pushEffect(blur_effect_.get());
    canvas.fill(0, 0, width(), height()); // Proper Visage drawing
    canvas.popEffect();
}
```

#### Effect Animation Pitfalls
```cpp
// WRONG: Updating effect parameters without checking cost
void timerCallback() override {
    float radius = std::sin(time_) * 20.0f;
    blur_effect_->setBlurRadius(radius); // Might rebuild internal buffers!
}

// CORRECT: Cache expensive effect configurations
void timerCallback() override {
    int radius_step = static_cast<int>(std::sin(time_) * 5.0f) * 4.0f;
    if (radius_step != last_radius_step_) {
        blur_effect_->setBlurRadius(radius_step);
        last_radius_step_ = radius_step;
    }
}
```

#### HDR and Color Space Issues
```cpp
// WRONG: Not considering HDR when using bloom
auto bloom = std::make_unique<visage::BloomEffect>();
bloom->setIntensity(1.0f); // May blow out on HDR displays

// CORRECT: Configure for display capabilities
auto bloom = std::make_unique<visage::BloomEffect>();
if (visage::Renderer::instance().isHDREnabled()) {
    bloom->setHDR(true);
    bloom->setIntensity(0.6f); // Lower for HDR
    bloom->setToneMapping(true);
} else {
    bloom->setIntensity(1.0f);
}
```

## Animation and GPU Optimization

### GPU-Accelerated Animations
Create base class for animatable components:

```cpp
class AnimatableFrame : public visage::Frame {
protected:
    void drawContent(visage::Canvas& canvas) override {
        // Custom drawing implementation
    }
    
    void draw(visage::Canvas& canvas) override {
        // Apply transforms
        canvas.save();
        canvas.translate(animationOffset_);
        canvas.scale(animationScale_);
        drawContent(canvas);
        canvas.restore();
    }
};
```

### Performance Patterns
- Use `redraw()` only when needed
- Batch similar drawing operations
- Cache complex calculations
- Use transforms instead of repainting

## Visage Effects System

### Understanding Visage's GPU-Accelerated Effects

Visage provides a comprehensive effects system built on GPU shaders. Unlike JUCE where effects require custom Graphics code, Visage effects are first-class citizens designed for real-time performance.

### Effect Categories & Performance Characteristics

#### Core Effect Categories

```cpp
// Source/UI/Visage/Effects/EffectCategories.h

namespace VisageEffects {
    
    // 1. BLUR EFFECTS
    namespace Blur {
        // Standard Gaussian Blur - visage::BlurPostEffect
        // - Real-time variable radius blur
        // - Optimized separable implementation
        // - Typical range: 0.5 - 20.0 pixels
        
        class GaussianBlur : public visage::BlurPostEffect {
            // Inherited: setBlurSize(), setBlurAmount()
            // GPU Cost: Medium (two-pass separable)
        };
        
        // Box Blur - Optimized for large radii
        class BoxBlur : public visage::PostEffect {
            // Constant time regardless of radius
            // Less smooth than Gaussian but faster
            // GPU Cost: Low
        };
        
        // Directional Blur (Motion Blur)
        class DirectionalBlur : public visage::PostEffect {
            // Blur along specific angle
            // Variable distance and angle
            // GPU Cost: Medium
        };
    }
    
    // 2. BLOOM EFFECTS
    namespace Bloom {
        // Standard Bloom - visage::BloomPostEffect
        // - Multi-pass implementation
        // - Threshold-based glow
        // - HDR support
        
        class StandardBloom : public visage::BloomPostEffect {
            // Inherited: setBloomIntensity(), setBloomSize()
            // setHdr(), setThreshold()
            // GPU Cost: High (multiple passes)
        };
        
        // Lens Flare Bloom
        class LensFlareBloom : public visage::BloomPostEffect {
            // Adds chromatic streaks
            // Simulates camera lens artifacts
            // GPU Cost: High
        };
    }
    
    // 3. DISTORTION EFFECTS
    namespace Distortion {
        // Wave/Ripple distortions
        class WaveDistortion : public visage::PostEffect {
            // Sine wave displacement
            // Animated or static
            // GPU Cost: Low
        };
        
        // Lens distortions
        class LensDistortion : public visage::PostEffect {
            // Barrel/Pincushion distortion
            // Chromatic aberration
            // GPU Cost: Medium
        };
        
        // Heat/Refraction distortion
        class RefractionDistortion : public visage::PostEffect {
            // Simulates heat haze or glass
            // Uses normal maps
            // GPU Cost: Medium
        };
    }
    
    // 4. COLOR MANIPULATION
    namespace Color {
        // HSL adjustments
        class HSLAdjustment : public visage::PostEffect {
            // Hue shift: -180 to +180 degrees
            // Saturation: 0 to 2x
            // Lightness: -1 to +1
            // GPU Cost: Low
        };
        
        // Color grading with LUTs
        class ColorGrading : public visage::PostEffect {
            // 3D LUT support
            // Film emulation
            // GPU Cost: Low
        };
        
        // Channel effects
        class ChannelEffects : public visage::PostEffect {
            // Channel swap/isolation
            // Chromatic aberration
            // GPU Cost: Low
        };
    }
    
    // 5. STYLIZATION EFFECTS
    namespace Stylize {
        // Edge detection/outline
        class EdgeDetection : public visage::PostEffect {
            // Sobel/Canny edge detection
            // Variable thickness
            // GPU Cost: Medium
        };
        
        // Posterization
        class Posterize : public visage::PostEffect {
            // Reduce color levels
            // Comic book effect
            // GPU Cost: Low
        };
        
        // Pixelation
        class Pixelate : public visage::PostEffect {
            // Variable pixel size
            // Maintains aspect ratio
            // GPU Cost: Low
        };
    }
}
```

### Effect Quick Reference

| Effect | GPU Cost | Use Case | Key Parameters |
|--------|----------|----------|----------------|
| Gaussian Blur | Medium | UI softening, backgrounds | radius (0.5-20px) |
| Bloom | High | Glow effects, highlights | intensity, threshold |
| Wave Distortion | Low | Animation, water effects | amplitude, frequency |
| Chromatic Aberration | Low | Retro/glitch look | R/B offset |
| Film Grain | Low | Vintage/film look | amount, size |
| Drop Shadow | Medium | Depth, elevation | offset, blur, color |
| Scanlines | Low | CRT monitor effect | density, opacity |
| Hue Shift | Low | Color variations | shift (-180 to 180) |
| Inner Shadow | Medium | Inset depth | offset, blur, color |
| Pixelate | Low | Retro/privacy effect | pixel size |

### Basic Effect Usage

```cpp
// Simple effect application
void draw(visage::Canvas& canvas) override {
    // Single effect
    canvas.pushEffect(std::make_unique<visage::BlurEffect>(5.0f));
    drawContent(canvas);
    canvas.popEffect();
    
    // Layered effects
    canvas.pushEffect(bloomEffect_.get());
    canvas.pushEffect(blurEffect_.get());
    drawContent(canvas);
    canvas.popEffect(); // blur
    canvas.popEffect(); // bloom
}
```

### Migrating JUCE Effects to Visage

#### Shadow Effects
```cpp
// JUCE
void paint(Graphics& g) override {
    // Manual shadow with multiple draws
    g.setColour(Colours::black.withAlpha(0.3f));
    g.fillRect(bounds.translated(2, 2));
    g.setColour(componentColor);
    g.fillRect(bounds);
}

// Visage
void draw(visage::Canvas& canvas) override {
    auto shadow = std::make_unique<DropShadowEffect>();
    shadow->setOffset(2.0f, 2.0f);
    shadow->setBlurRadius(4.0f);
    shadow->setShadowColor(0x4D000000);
    
    canvas.pushEffect(shadow.get());
    canvas.setColor(componentColor);
    canvas.fill(0, 0, width(), height());
    canvas.popEffect();
}
```

#### Blur Effects
```cpp
// JUCE (requires custom implementation)
class JUCEBlurEffect {
    void applyBlur(Image& img, int radius) {
        // Complex CPU-based convolution
        for (int y = 0; y < img.getHeight(); ++y) {
            for (int x = 0; x < img.getWidth(); ++x) {
                // Expensive per-pixel operations
            }
        }
    }
};

// Visage (built-in GPU acceleration)
canvas.pushEffect(std::make_unique<visage::BlurEffect>(10.0f));
drawContent(canvas);
canvas.popEffect();
```

#### Glow/Bloom Effects
```cpp
// JUCE (manual implementation)
void createGlow(Graphics& g, Rectangle<int> bounds) {
    Image glow(Image::ARGB, bounds.getWidth() * 2, bounds.getHeight() * 2, true);
    Graphics glowG(glow);
    // Draw enlarged blurred version
    // Composite back with transparency
    // Very expensive operation
}

// Visage (optimized GPU bloom)
auto bloom = std::make_unique<visage::BloomEffect>();
bloom->setIntensity(0.6f);
bloom->setThreshold(0.8f);
canvas.pushEffect(bloom.get());
drawContent(canvas);
canvas.popEffect();
```

### Effect Performance Guidelines

```cpp
class EffectPerformanceGuide {
public:
    enum class GPUCost {
        Low,      // < 0.5ms per frame
        Medium,   // 0.5-2ms per frame  
        High      // > 2ms per frame
    };
    
    struct EffectProfile {
        std::string name;
        GPUCost cost;
        bool requires_multiple_passes;
        bool supports_batching;
        size_t texture_memory_bytes;
    };
    
    // Effect stacking recommendations
    static int getMaxRecommendedEffects(float target_fps) {
        if (target_fps >= 120) return 1;  // High refresh rate
        if (target_fps >= 60) return 3;   // Standard 60fps
        if (target_fps >= 30) return 5;   // Lower performance
        return 2; // Mobile/low-end
    }
    
    // Check if effect combination is viable
    static bool isViableCombination(const std::vector<EffectProfile>& effects) {
        float total_ms = 0.0f;
        for (const auto& effect : effects) {
            switch (effect.cost) {
                case GPUCost::Low: total_ms += 0.3f; break;
                case GPUCost::Medium: total_ms += 1.0f; break;
                case GPUCost::High: total_ms += 2.5f; break;
            }
        }
        return total_ms < 16.0f; // Under 16ms for 60fps
    }
};
```

### Effect Combination Patterns

```cpp
// Recommended effect combinations
namespace EffectPresets {
    std::vector<std::unique_ptr<visage::PostEffect>> createCinematicPreset() {
        std::vector<std::unique_ptr<visage::PostEffect>> effects;
        
        // Subtle chromatic aberration
        auto chroma = std::make_unique<ChromaticAberrationEffect>();
        chroma->setRedOffset(0.001f, 0.0f);
        chroma->setBlueOffset(-0.001f, 0.0f);
        effects.push_back(std::move(chroma));
        
        // Soft bloom
        auto bloom = std::make_unique<visage::BloomEffect>();
        bloom->setIntensity(0.3f);
        bloom->setThreshold(0.85f);
        effects.push_back(std::move(bloom));
        
        // Film grain
        auto grain = std::make_unique<FilmGrainEffect>();
        grain->setAmount(0.02f);
        effects.push_back(std::move(grain));
        
        return effects;
    }
    
    std::vector<std::unique_ptr<visage::PostEffect>> createRetroPreset() {
        std::vector<std::unique_ptr<visage::PostEffect>> effects;
        
        // CRT curvature
        auto barrel = std::make_unique<BarrelDistortionEffect>();
        barrel->setStrength(0.02f);
        effects.push_back(std::move(barrel));
        
        // Scanlines
        auto scanlines = std::make_unique<ScanlinesEffect>();
        scanlines->setDensity(600.0f);
        scanlines->setOpacity(0.15f);
        effects.push_back(std::move(scanlines));
        
        // Chromatic aberration
        auto chroma = std::make_unique<ChromaticAberrationEffect>();
        chroma->setRedOffset(0.003f, 0.0f);
        chroma->setBlueOffset(-0.003f, 0.0f);
        effects.push_back(std::move(chroma));
        
        // Bloom for CRT glow
        auto bloom = std::make_unique<visage::BloomEffect>();
        bloom->setIntensity(0.4f);
        effects.push_back(std::move(bloom));
        
        return effects;
    }
}
```

### Effect Feature Flags

```cmake
# In CMakeLists.txt
option(ENABLE_BASIC_EFFECTS "Basic blur/bloom effects" ON)
option(ENABLE_DISTORTION_EFFECTS "Wave/ripple/fisheye effects" OFF)
option(ENABLE_STYLIZATION_EFFECTS "Artistic effects" OFF)
option(ENABLE_ADVANCED_SHADERS "Custom shader effects" OFF)
```

```cpp
// Conditional effect compilation
std::unique_ptr<visage::PostEffect> createEffect(const std::string& type) {
    if (type == "blur") {
#ifdef ENABLE_BASIC_EFFECTS
        return std::make_unique<visage::BlurEffect>();
#else
        return nullptr;
#endif
    }
    
    if (type == "wave") {
#ifdef ENABLE_DISTORTION_EFFECTS
        return std::make_unique<WaveDistortionEffect>();
#else
        DBG("Distortion effects not enabled in build");
        return nullptr;
#endif
    }
    
    if (type == "pixelate") {
#ifdef ENABLE_STYLIZATION_EFFECTS
        return std::make_unique<PixelateEffect>();
#else
        return nullptr;
#endif
    }
    
    return nullptr;
}
```

### Platform-Specific Effect Optimizations

```cpp
namespace PlatformEffects {
#ifdef _WIN32
    // DirectX 11/12 optimizations
    void optimizeForDirectX(visage::PostEffect* effect) {
        // Use compute shaders for blur effects
        if (auto* blur = dynamic_cast<visage::BlurEffect*>(effect)) {
            blur->setUseComputeShader(true);
        }
    }
#endif
    
#ifdef __APPLE__
    // Metal optimizations
    void optimizeForMetal(visage::PostEffect* effect) {
        // Use Metal Performance Shaders where available
        if (@available(macOS 10.13, *)) {
            effect->setUseNativeOptimization(true);
        }
    }
#endif
    
#ifdef __linux__
    // Vulkan optimizations
    void optimizeForVulkan(visage::PostEffect* effect) {
        // Enable Vulkan-specific features
        effect->setMultiGPU(true);
    }
#endif
}
```

### Effect Profiling and Debugging

```cpp
class EffectProfiler {
public:
    struct EffectMetrics {
        std::string effect_name;
        float gpu_time_ms = 0.0f;
        float cpu_time_ms = 0.0f;
        size_t texture_memory_mb = 0;
        int draw_calls = 0;
    };
    
    static EffectMetrics profileEffect(visage::PostEffect* effect, 
                                      visage::Frame* testContent) {
        EffectMetrics metrics;
        metrics.effect_name = typeid(*effect).name();
        
        // GPU timing
        auto gpu_start = visage::Renderer::instance().getGPUTime();
        
        visage::Canvas canvas;
        canvas.pushEffect(effect);
        testContent->draw(canvas);
        canvas.popEffect();
        int submit_pass = canvas.submit();
        
        // Wait for GPU completion
        visage::Renderer::instance().frame();
        
        auto gpu_end = visage::Renderer::instance().getGPUTime();
        metrics.gpu_time_ms = gpu_end - gpu_start;
        
        // Memory usage
        metrics.texture_memory_mb = effect->getTextureMemoryUsage() / (1024 * 1024);
        
        return metrics;
    }
    
    static void generateEffectReport(const std::vector<EffectMetrics>& metrics) {
        DBG("=== Effect Performance Report ===");
        for (const auto& m : metrics) {
            DBG(m.effect_name << ":");
            DBG("  GPU Time: " << m.gpu_time_ms << "ms");
            DBG("  Memory: " << m.texture_memory_mb << "MB");
            DBG("  Draw Calls: " << m.draw_calls);
        }
    }
};
```

### Common GPU-Accelerated UI Patterns

#### Window/Panel Animation Effects

```cpp
// Slide-in/out animations for panels
class SlidePanel : public visage::Frame {
private:
    enum SlideDirection { Left, Right, Top, Bottom };
    float animationProgress_ = 0.0f;
    bool isShowing_ = false;
    SlideDirection direction_ = Right;
    
public:
    void show() {
        isShowing_ = true;
        startAnimation();
    }
    
    void hide() {
        isShowing_ = false;
        startAnimation();
    }
    
    void draw(visage::Canvas& canvas) override {
        canvas.save();
        
        // Calculate offset based on animation progress
        float offset = (1.0f - animationProgress_) * width();
        if (!isShowing_) offset = -offset;
        
        switch (direction_) {
            case Right:
                canvas.translate(offset, 0);
                break;
            case Left:
                canvas.translate(-offset, 0);
                break;
            case Top:
                canvas.translate(0, -offset);
                break;
            case Bottom:
                canvas.translate(0, offset);
                break;
        }
        
        // Optional: Add shadow for depth
        if (animationProgress_ > 0.01f) {
            auto shadow = std::make_unique<DropShadowEffect>();
            shadow->setOffset(5.0f, 5.0f);
            shadow->setBlurRadius(20.0f);
            shadow->setShadowColor(0x40000000);
            canvas.pushEffect(shadow.get());
        }
        
        drawContent(canvas);
        
        if (animationProgress_ > 0.01f) {
            canvas.popEffect();
        }
        
        canvas.restore();
    }
};
```

#### Background Blur/Focus Effects

```cpp
// Modal/Dialog background blur effect
class ModalBackground : public visage::Frame {
private:
    float blurAmount_ = 0.0f;
    float targetBlur_ = 20.0f;
    float opacity_ = 0.0f;
    float targetOpacity_ = 0.7f;
    std::unique_ptr<visage::BlurEffect> blurEffect_;
    
public:
    ModalBackground() {
        blurEffect_ = std::make_unique<visage::BlurEffect>(0.0f);
    }
    
    void show() {
        targetBlur_ = 20.0f;
        targetOpacity_ = 0.7f;
        startTimer(16); // 60fps animation
    }
    
    void hide() {
        targetBlur_ = 0.0f;
        targetOpacity_ = 0.0f;
    }
    
    void draw(visage::Canvas& canvas) override {
        // Animate blur
        blurAmount_ += (targetBlur_ - blurAmount_) * 0.1f;
        opacity_ += (targetOpacity_ - opacity_) * 0.1f;
        
        blurEffect_->setBlurSize(blurAmount_);
        
        // Render background content with blur
        canvas.pushEffect(blurEffect_.get());
        
        // Render parent content here or use render-to-texture
        if (auto* parent = getParentComponent()) {
            parent->draw(canvas);
        }
        
        canvas.popEffect();
        
        // Dark overlay
        canvas.setColor(0x000000 | (uint32_t(opacity_ * 255) << 24));
        canvas.fill(0, 0, width(), height());
    }
    
    void timerCallback() override {
        if (std::abs(blurAmount_ - targetBlur_) < 0.1f &&
            std::abs(opacity_ - targetOpacity_) < 0.01f) {
            stopTimer();
        }
        redraw();
    }
};
```

#### 3D-Style Flip Animations

```cpp
// Card flip effect
class FlipCard : public visage::Frame {
private:
    float rotation_ = 0.0f;
    float targetRotation_ = 0.0f;
    bool showingFront_ = true;
    
public:
    void flip() {
        targetRotation_ += M_PI;
        showingFront_ = !showingFront_;
        startTimer(16);
    }
    
    void draw(visage::Canvas& canvas) override {
        rotation_ += (targetRotation_ - rotation_) * 0.15f;
        
        canvas.save();
        
        // Apply 3D rotation transform
        float perspective = 1000.0f;
        float centerX = width() / 2.0f;
        float centerY = height() / 2.0f;
        
        // Simple 3D projection
        canvas.translate(centerX, centerY);
        float scale = perspective / (perspective + 200.0f * std::sin(rotation_));
        canvas.scale(std::cos(rotation_) * scale, scale);
        canvas.translate(-centerX, -centerY);
        
        // Draw front or back based on rotation
        if (std::cos(rotation_) > 0) {
            drawFrontContent(canvas);
        } else {
            canvas.scale(-1.0f, 1.0f); // Flip horizontally
            canvas.translate(-width(), 0);
            drawBackContent(canvas);
        }
        
        canvas.restore();
    }
};
```

#### Material Design Ripple Effect

```cpp
class RippleButton : public visage::Button {
private:
    struct Ripple {
        float x, y;
        float radius = 0.0f;
        float maxRadius;
        float opacity = 0.5f;
        bool expanding = true;
    };
    
    std::vector<Ripple> ripples_;
    
public:
    void onMouseDown(const visage::MouseEvent& e) override {
        // Create new ripple at click position
        Ripple ripple;
        ripple.x = e.position.x;
        ripple.y = e.position.y;
        ripple.maxRadius = std::sqrt(width() * width() + height() * height());
        ripples_.push_back(ripple);
        
        startTimer(16);
    }
    
    void draw(visage::Canvas& canvas) override {
        // Draw button background
        canvas.setColor(0xff2196f3);
        canvas.roundedRect(0, 0, width(), height(), 4.0f);
        
        // Draw ripples
        canvas.save();
        canvas.clipRect(0, 0, width(), height());
        
        for (auto& ripple : ripples_) {
            uint32_t rippleColor = 0xffffff | (uint32_t(ripple.opacity * 100) << 24);
            canvas.setColor(rippleColor);
            canvas.circle(ripple.x, ripple.y, ripple.radius);
        }
        
        canvas.restore();
        
        // Draw button text
        drawButtonText(canvas);
    }
    
    void timerCallback() override {
        bool hasActiveRipples = false;
        
        for (auto it = ripples_.begin(); it != ripples_.end();) {
            it->radius += 8.0f;
            it->opacity -= 0.02f;
            
            if (it->opacity <= 0.0f || it->radius > it->maxRadius) {
                it = ripples_.erase(it);
            } else {
                ++it;
                hasActiveRipples = true;
            }
        }
        
        if (!hasActiveRipples) {
            stopTimer();
        }
        
        redraw();
    }
};
```

#### Elastic/Spring Animations

```cpp
class ElasticButton : public visage::Button {
private:
    float scale_ = 1.0f;
    float scaleVelocity_ = 0.0f;
    float targetScale_ = 1.0f;
    const float springStiffness_ = 300.0f;
    const float springDamping_ = 20.0f;
    
public:
    void onMouseDown(const visage::MouseEvent& e) override {
        targetScale_ = 0.95f;
        startTimer(16);
    }
    
    void onMouseUp(const visage::MouseEvent& e) override {
        targetScale_ = 1.1f; // Overshoot
        juce::Timer::callAfterDelay(100, [this]() {
            targetScale_ = 1.0f;
        });
    }
    
    void draw(visage::Canvas& canvas) override {
        canvas.save();
        
        float centerX = width() / 2.0f;
        float centerY = height() / 2.0f;
        
        canvas.translate(centerX, centerY);
        canvas.scale(scale_, scale_);
        canvas.translate(-centerX, -centerY);
        
        // Draw button
        canvas.setColor(0xff4CAF50);
        canvas.roundedRect(0, 0, width(), height(), 8.0f);
        
        canvas.restore();
    }
    
    void timerCallback() override {
        // Spring physics
        float force = (targetScale_ - scale_) * springStiffness_;
        scaleVelocity_ += force / 60.0f; // 60fps
        scaleVelocity_ *= (1.0f - springDamping_ / 60.0f);
        scale_ += scaleVelocity_ / 60.0f;
        
        if (std::abs(scale_ - targetScale_) < 0.001f && 
            std::abs(scaleVelocity_) < 0.001f) {
            scale_ = targetScale_;
            stopTimer();
        }
        
        redraw();
    }
};
```

#### Morph/Shape Transitions

```cpp
class MorphingShape : public visage::Frame {
private:
    enum Shape { Circle, Square, Triangle, Hexagon };
    Shape currentShape_ = Circle;
    Shape targetShape_ = Circle;
    float morphProgress_ = 0.0f;
    
    struct ControlPoint {
        float x, y;
        float handleX1, handleY1;
        float handleX2, handleY2;
    };
    
    std::vector<ControlPoint> currentPoints_;
    std::vector<ControlPoint> targetPoints_;
    
public:
    void morphTo(Shape shape) {
        targetShape_ = shape;
        currentPoints_ = getCurrentShapePoints();
        targetPoints_ = getShapePoints(shape);
        morphProgress_ = 0.0f;
        startTimer(16);
    }
    
    void draw(visage::Canvas& canvas) override {
        canvas.save();
        
        // Interpolate between shapes
        std::vector<ControlPoint> interpolatedPoints;
        for (size_t i = 0; i < currentPoints_.size(); ++i) {
            ControlPoint p;
            p.x = lerp(currentPoints_[i].x, targetPoints_[i].x, morphProgress_);
            p.y = lerp(currentPoints_[i].y, targetPoints_[i].y, morphProgress_);
            p.handleX1 = lerp(currentPoints_[i].handleX1, targetPoints_[i].handleX1, morphProgress_);
            p.handleY1 = lerp(currentPoints_[i].handleY1, targetPoints_[i].handleY1, morphProgress_);
            p.handleX2 = lerp(currentPoints_[i].handleX2, targetPoints_[i].handleX2, morphProgress_);
            p.handleY2 = lerp(currentPoints_[i].handleY2, targetPoints_[i].handleY2, morphProgress_);
            interpolatedPoints.push_back(p);
        }
        
        // Draw morphed shape
        canvas.setColor(0xff9C27B0);
        drawBezierPath(canvas, interpolatedPoints);
        
        canvas.restore();
    }
    
    void timerCallback() override {
        morphProgress_ += 0.03f;
        
        if (morphProgress_ >= 1.0f) {
            morphProgress_ = 1.0f;
            currentShape_ = targetShape_;
            stopTimer();
        }
        
        redraw();
    }
    
private:
    float lerp(float a, float b, float t) {
        return a + (b - a) * t;
    }
};
```

#### Parallax Scrolling

```cpp
class ParallaxBackground : public visage::Frame {
private:
    struct Layer {
        std::unique_ptr<visage::Image> image;
        float scrollSpeed;
        float xOffset = 0.0f;
        float yOffset = 0.0f;
        float scale = 1.0f;
    };
    
    std::vector<Layer> layers_;
    float scrollX_ = 0.0f;
    float scrollY_ = 0.0f;
    
public:
    void addLayer(std::unique_ptr<visage::Image> image, float speed, float scale = 1.0f) {
        layers_.push_back({std::move(image), speed, 0.0f, 0.0f, scale});
    }
    
    void setScrollPosition(float x, float y) {
        scrollX_ = x;
        scrollY_ = y;
        
        for (auto& layer : layers_) {
            layer.xOffset = -x * layer.scrollSpeed;
            layer.yOffset = -y * layer.scrollSpeed;
        }
        redraw();
    }
    
    void draw(visage::Canvas& canvas) override {
        for (const auto& layer : layers_) {
            canvas.save();
            
            // Apply parallax offset and scale
            float scaledWidth = layer.image->width() * layer.scale;
            float scaledHeight = layer.image->height() * layer.scale;
            
            canvas.translate(layer.xOffset, layer.yOffset);
            canvas.scale(layer.scale, layer.scale);
            
            // Tile the image if needed
            float startX = std::fmod(layer.xOffset, scaledWidth);
            float startY = std::fmod(layer.yOffset, scaledHeight);
            
            for (float x = startX - scaledWidth; x < width(); x += scaledWidth) {
                for (float y = startY - scaledHeight; y < height(); y += scaledHeight) {
                    canvas.drawImage(*layer.image, x, y);
                }
            }
            
            canvas.restore();
        }
    }
};
```

#### Glassmorphism Effect

```cpp
class GlassmorphicPanel : public visage::Frame {
private:
    std::unique_ptr<visage::BlurEffect> blurEffect_;
    float glassOpacity_ = 0.85f;
    float borderOpacity_ = 0.3f;
    
public:
    GlassmorphicPanel() {
        blurEffect_ = std::make_unique<visage::BlurEffect>(15.0f);
    }
    
    void draw(visage::Canvas& canvas) override {
        canvas.save();
        
        // Clip to rounded rect
        canvas.clipRoundedRect(0, 0, width(), height(), 16.0f);
        
        // Blur background
        canvas.pushEffect(blurEffect_.get());
        renderBackgroundContent(canvas);
        canvas.popEffect();
        
        // Glass overlay with gradient
        auto gradient = canvas.createLinearGradient(0, 0, 0, height());
        gradient.addColorStop(0.0f, 0xffffff | (uint32_t(glassOpacity_ * 40) << 24));
        gradient.addColorStop(1.0f, 0xffffff | (uint32_t(glassOpacity_ * 20) << 24));
        canvas.setGradient(gradient);
        canvas.roundedRect(0, 0, width(), height(), 16.0f);
        
        // Inner glow
        canvas.setColor(0xffffff | (uint32_t(glassOpacity_ * 60) << 24));
        canvas.strokeRoundedRect(1, 1, width() - 2, height() - 2, 15.0f, 1.0f);
        
        // Outer border
        canvas.setColor(0xffffff | (uint32_t(borderOpacity_ * 255) << 24));
        canvas.strokeRoundedRect(0, 0, width(), height(), 16.0f, 1.0f);
        
        canvas.restore();
        
        // Draw content
        drawPanelContent(canvas);
    }
};
```

#### Neumorphism (Soft UI)

```cpp
class NeumorphicButton : public visage::Button {
private:
    bool isPressed_ = false;
    float pressedAmount_ = 0.0f;
    
public:
    void onMouseDown(const visage::MouseEvent& e) override {
        isPressed_ = true;
        startTimer(16);
    }
    
    void onMouseUp(const visage::MouseEvent& e) override {
        isPressed_ = false;
    }
    
    void draw(visage::Canvas& canvas) override {
        // Animate pressed state
        float targetPressed = isPressed_ ? 1.0f : 0.0f;
        pressedAmount_ += (targetPressed - pressedAmount_) * 0.2f;
        
        // Background color
        uint32_t bgColor = 0xffe0e5ec;
        canvas.setColor(bgColor);
        canvas.fill(0, 0, width(), height());
        
        // Calculate shadow parameters
        float shadowOffset = 4.0f * (1.0f - pressedAmount_);
        float shadowBlur = 8.0f * (1.0f - pressedAmount_);
        float innerShadowOffset = 2.0f * pressedAmount_;
        
        // Outer shadows (only when not pressed)
        if (pressedAmount_ < 0.9f) {
            // Light shadow
            auto lightShadow = std::make_unique<DropShadowEffect>();
            lightShadow->setOffset(-shadowOffset, -shadowOffset);
            lightShadow->setBlurRadius(shadowBlur);
            lightShadow->setShadowColor(0x40ffffff);
            
            // Dark shadow
            auto darkShadow = std::make_unique<DropShadowEffect>();
            darkShadow->setOffset(shadowOffset, shadowOffset);
            darkShadow->setBlurRadius(shadowBlur);
            darkShadow->setShadowColor(0x20000000);
            
            canvas.pushEffect(lightShadow.get());
            canvas.pushEffect(darkShadow.get());
        }
        
        // Button shape
        canvas.setColor(bgColor);
        canvas.roundedRect(4, 4, width() - 8, height() - 8, 12.0f);
        
        // Inner shadow when pressed
        if (pressedAmount_ > 0.1f) {
            auto innerShadow = std::make_unique<InnerShadowEffect>();
            innerShadow->setOffset(innerShadowOffset, innerShadowOffset);
            innerShadow->setBlurRadius(4.0f * pressedAmount_);
            innerShadow->setShadowColor(0x30000000 | (uint32_t(pressedAmount_ * 255) << 24));
            
            canvas.pushEffect(innerShadow.get());
            canvas.roundedRect(4, 4, width() - 8, height() - 8, 12.0f);
            canvas.popEffect();
        }
        
        // Pop outer shadows
        if (pressedAmount_ < 0.9f) {
            canvas.popEffect();
            canvas.popEffect();
        }
        
        // Draw content
        drawButtonContent(canvas);
    }
    
    void timerCallback() override {
        if (std::abs(pressedAmount_ - (isPressed_ ? 1.0f : 0.0f)) < 0.01f) {
            stopTimer();
        }
        redraw();
    }
};
```

#### Particle Effects

```cpp
class ParticleSystem : public visage::Frame, public juce::Timer {
private:
    struct Particle {
        float x, y;
        float vx, vy;
        float life;
        float maxLife;
        float size;
        float rotation;
        float rotationSpeed;
        uint32_t color;
        float gravity;
    };
    
    std::vector<Particle> particles_;
    std::default_random_engine rng_;
    
    // Emitter properties
    float emitterX_ = 0.0f;
    float emitterY_ = 0.0f;
    float emissionRate_ = 10.0f;
    float emissionAngle_ = 0.0f;
    float emissionSpread_ = M_PI / 4.0f;
    
public:
    ParticleSystem() {
        startTimerHz(60);
    }
    
    void setEmitterPosition(float x, float y) {
        emitterX_ = x;
        emitterY_ = y;
    }
    
    void burst(int count) {
        for (int i = 0; i < count; ++i) {
            emitParticle();
        }
    }
    
    void draw(visage::Canvas& canvas) override {
        // Optional bloom for glow
        auto bloom = std::make_unique<visage::BloomEffect>();
        bloom->setIntensity(0.6f);
        bloom->setThreshold(0.5f);
        canvas.pushEffect(bloom.get());
        
        for (const auto& p : particles_) {
            canvas.save();
            
            // Apply particle transform
            canvas.translate(p.x, p.y);
            canvas.rotate(p.rotation);
            
            // Fade based on life
            float alpha = p.life / p.maxLife;
            uint32_t color = (p.color & 0x00ffffff) | (uint32_t(alpha * 255) << 24);
            canvas.setColor(color);
            
            // Draw particle (can be customized)
            float currentSize = p.size * (0.5f + 0.5f * alpha); // Shrink as it fades
            canvas.circle(0, 0, currentSize);
            
            canvas.restore();
        }
        
        canvas.popEffect();
    }
    
    void timerCallback() override {
        // Emit new particles
        static float emissionAccumulator = 0.0f;
        emissionAccumulator += emissionRate_ / 60.0f;
        while (emissionAccumulator >= 1.0f) {
            emitParticle();
            emissionAccumulator -= 1.0f;
        }
        
        // Update existing particles
        for (auto it = particles_.begin(); it != particles_.end();) {
            // Physics update
            it->vy += it->gravity;
            it->x += it->vx;
            it->y += it->vy;
            it->rotation += it->rotationSpeed;
            it->life -= 1.0f / 60.0f;
            
            // Remove dead particles
            if (it->life <= 0 || it->y > height() + 50) {
                it = particles_.erase(it);
            } else {
                ++it;
            }
        }
        
        redraw();
    }
    
private:
    void emitParticle() {
        std::uniform_real_distribution<float> angleDist(
            emissionAngle_ - emissionSpread_,
            emissionAngle_ + emissionSpread_
        );
        std::uniform_real_distribution<float> speedDist(2.0f, 5.0f);
        std::uniform_real_distribution<float> sizeDist(2.0f, 8.0f);
        std::uniform_real_distribution<float> lifeDist(1.0f, 3.0f);
        std::uniform_real_distribution<float> rotSpeedDist(-0.1f, 0.1f);
        
        Particle p;
        p.x = emitterX_;
        p.y = emitterY_;
        
        float angle = angleDist(rng_);
        float speed = speedDist(rng_);
        p.vx = std::cos(angle) * speed;
        p.vy = std::sin(angle) * speed;
        
        p.maxLife = lifeDist(rng_);
        p.life = p.maxLife;
        p.size = sizeDist(rng_);
        p.rotation = 0.0f;
        p.rotationSpeed = rotSpeedDist(rng_);
        p.color = 0xff00a0ff; // Can be randomized
        p.gravity = 0.1f;
        
        particles_.push_back(p);
    }
};
```

#### Skeleton Loading Animation

```cpp
class SkeletonLoader : public visage::Frame {
private:
    float shimmerPosition_ = -0.3f;
    
public:
    SkeletonLoader() {
        startTimer(16);
    }
    
    void draw(visage::Canvas& canvas) override {
        // Base skeleton color
        uint32_t baseColor = 0xffe0e0e0;
        uint32_t shimmerColor = 0xfff0f0f0;
        
        // Draw skeleton shapes
        canvas.setColor(baseColor);
        
        // Title skeleton
        canvas.roundedRect(20, 20, width() * 0.6f, 20, 4.0f);
        
        // Subtitle skeleton
        canvas.roundedRect(20, 50, width() * 0.4f, 16, 4.0f);
        
        // Content skeletons
        for (int i = 0; i < 3; ++i) {
            float y = 90 + i * 30;
            canvas.roundedRect(20, y, width() - 40, 12, 4.0f);
        }
        
        // Animated shimmer effect
        canvas.save();
        canvas.clipRect(0, 0, width(), height());
        
        // Create gradient for shimmer
        float shimmerX = width() * shimmerPosition_;
        auto gradient = canvas.createLinearGradient(
            shimmerX - 100, 0,
            shimmerX + 100, 0
        );
        gradient.addColorStop(0.0f, baseColor);
        gradient.addColorStop(0.5f, shimmerColor);
        gradient.addColorStop(1.0f, baseColor);
        
        canvas.setGradient(gradient);
        canvas.fill(0, 0, width(), height());
        
        canvas.restore();
    }
    
    void timerCallback() override {
        shimmerPosition_ += 0.01f;
        if (shimmerPosition_ > 1.3f) {
            shimmerPosition_ = -0.3f;
        }
        redraw();
    }
};
```

#### Liquid/Metaball Effect

```cpp
class MetaballEffect : public visage::Frame {
private:
    struct Ball {
        float x, y;
        float vx, vy;
        float radius;
    };
    
    std::vector<Ball> balls_;
    std::unique_ptr<visage::BlurEffect> blurEffect_;
    
public:
    MetaballEffect() {
        blurEffect_ = std::make_unique<visage::BlurEffect>(8.0f);
        
        // Initialize some metaballs
        for (int i = 0; i < 5; ++i) {
            Ball ball;
            ball.x = random.nextFloat() * width();
            ball.y = random.nextFloat() * height();
            ball.vx = (random.nextFloat() - 0.5f) * 2.0f;
            ball.vy = (random.nextFloat() - 0.5f) * 2.0f;
            ball.radius = 30.0f + random.nextFloat() * 20.0f;
            balls_.push_back(ball);
        }
        
        startTimer(16);
    }
    
    void draw(visage::Canvas& canvas) override {
        // Draw metaballs with blur to create liquid effect
        canvas.pushEffect(blurEffect_.get());
        
        // Draw balls
        for (const auto& ball : balls_) {
            canvas.setColor(0xff0080ff);
            canvas.circle(ball.x, ball.y, ball.radius);
        }
        
        canvas.popEffect();
        
        // Apply threshold to create sharp edges
        auto thresholdEffect = std::make_unique<ThresholdEffect>();
        thresholdEffect->setThreshold(0.5f);
        canvas.pushEffect(thresholdEffect.get());
        
        // Redraw to apply threshold
        canvas.fill(0, 0, width(), height());
        
        canvas.popEffect();
    }
    
    void timerCallback() override {
        // Update ball positions
        for (auto& ball : balls_) {
            ball.x += ball.vx;
            ball.y += ball.vy;
            
            // Bounce off walls
            if (ball.x - ball.radius < 0 || ball.x + ball.radius > width()) {
                ball.vx = -ball.vx;
            }
            if (ball.y - ball.radius < 0 || ball.y + ball.radius > height()) {
                ball.vy = -ball.vy;
            }
            
            // Keep in bounds
            ball.x = std::clamp(ball.x, ball.radius, width() - ball.radius);
            ball.y = std::clamp(ball.y, ball.radius, height() - ball.radius);
        }
        
        redraw();
    }
};
```

#### Accordion/Collapse Animation

```cpp
class AccordionPanel : public visage::Frame {
private:
    struct Section {
        std::string title;
        std::unique_ptr<visage::Frame> content;
        float height;
        float currentHeight = 0.0f;
        bool isExpanded = false;
    };
    
    std::vector<Section> sections_;
    float headerHeight_ = 40.0f;
    
public:
    void addSection(const std::string& title, std::unique_ptr<visage::Frame> content) {
        Section section;
        section.title = title;
        section.height = content->height();
        section.content = std::move(content);
        sections_.push_back(std::move(section));
    }
    
    void draw(visage::Canvas& canvas) override {
        float y = 0;
        
        for (size_t i = 0; i < sections_.size(); ++i) {
            auto& section = sections_[i];
            
            // Draw header
            canvas.setColor(0xff3f51b5);
            canvas.fill(0, y, width(), headerHeight_);
            
            // Draw title
            auto* font = FontManager::instance().getFont(
                FontManager::FontType::Bold, 16.0f);
            canvas.setColor(0xffffffff);
            visage::String titleText(section.title.c_str());
            canvas.text(titleText, *font, visage::Font::kLeft,
                       20, y, width() - 40, headerHeight_);
            
            // Draw expand/collapse indicator
            float indicatorX = width() - 30;
            float indicatorY = y + headerHeight_ / 2;
            float rotation = section.isExpanded ? M_PI / 2 : 0;
            
            canvas.save();
            canvas.translate(indicatorX, indicatorY);
            canvas.rotate(rotation);
            drawArrow(canvas);
            canvas.restore();
            
            y += headerHeight_;
            
            // Draw content if expanded
            if (section.currentHeight > 0.1f) {
                canvas.save();
                canvas.clipRect(0, y, width(), section.currentHeight);
                
                section.content->setBounds(0, y, width(), section.height);
                section.content->draw(canvas);
                
                canvas.restore();
                
                y += section.currentHeight;
            }
            
            // Separator
            canvas.setColor(0x20000000);
            canvas.fill(0, y - 1, width(), 1);
        }
    }
    
    void onMouseDown(const visage::MouseEvent& e) override {
        float y = 0;
        
        for (auto& section : sections_) {
            if (e.position.y >= y && e.position.y < y + headerHeight_) {
                section.isExpanded = !section.isExpanded;
                startTimer(16);
                break;
            }
            
            y += headerHeight_ + section.currentHeight;
        }
    }
    
    void timerCallback() override {
        bool animating = false;
        
        for (auto& section : sections_) {
            float targetHeight = section.isExpanded ? section.height : 0.0f;
            float diff = targetHeight - section.currentHeight;
            
            if (std::abs(diff) > 0.1f) {
                section.currentHeight += diff * 0.2f;
                animating = true;
            } else {
                section.currentHeight = targetHeight;
            }
        }
        
        if (!animating) {
            stopTimer();
        }
        
        redraw();
    }
    
private:
    void drawArrow(visage::Canvas& canvas) {
        canvas.setColor(0xffffffff);
        canvas.moveTo(-5, -3);
        canvas.lineTo(5, 0);
        canvas.lineTo(-5, 3);
        canvas.closePath();
        canvas.fill();
    }
};
```

#### Progress/Loading Animations

```cpp
class CircularProgress : public visage::Frame {
private:
    float progress_ = 0.0f;
    float rotation_ = 0.0f;
    bool isIndeterminate_ = true;
    
public:
    CircularProgress() {
        startTimer(16);
    }
    
    void setProgress(float progress) {
        progress_ = std::clamp(progress, 0.0f, 1.0f);
        isIndeterminate_ = false;
    }
    
    void setIndeterminate(bool indeterminate) {
        isIndeterminate_ = indeterminate;
    }
    
    void draw(visage::Canvas& canvas) override {
        float centerX = width() / 2.0f;
        float centerY = height() / 2.0f;
        float radius = std::min(width(), height()) / 2.0f - 4.0f;
        
        // Background circle
        canvas.setColor(0x20000000);
        canvas.setLineWidth(4.0f);
        canvas.strokeCircle(centerX, centerY, radius);
        
        // Progress arc
        canvas.setColor(0xff2196f3);
        canvas.setLineWidth(4.0f);
        
        if (isIndeterminate_) {
            // Rotating arc for indeterminate progress
            float arcLength = M_PI / 2;
            float startAngle = rotation_;
            float endAngle = rotation_ + arcLength;
            
            canvas.arc(centerX, centerY, radius, startAngle, endAngle);
            canvas.stroke();
            
            // Secondary arc
            float secondaryStart = rotation_ + M_PI;
            float secondaryEnd = secondaryStart + arcLength / 2;
            canvas.setColor(0x802196f3);
            canvas.arc(centerX, centerY, radius, secondaryStart, secondaryEnd);
            canvas.stroke();
        } else {
            // Fixed arc for determinate progress
            float startAngle = -M_PI / 2;
            float endAngle = startAngle + (2 * M_PI * progress_);
            
            canvas.arc(centerX, centerY, radius, startAngle, endAngle);
            canvas.stroke();
            
            // Percentage text
            if (progress_ > 0.0f) {
                auto* font = FontManager::instance().getFont(
                    FontManager::FontType::Regular, 14.0f);
                std::string percentText = std::to_string(int(progress_ * 100)) + "%";
                visage::String visageText(percentText.c_str());
                canvas.setColor(0xff333333);
                canvas.text(visageText, *font, visage::Font::kCenter,
                           0, 0, width(), height());
            }
        }
    }
    
    void timerCallback() override {
        if (isIndeterminate_) {
            rotation_ += 0.05f;
            if (rotation_ > 2 * M_PI) {
                rotation_ -= 2 * M_PI;
            }
            redraw();
        }
    }
};
```

### Performance Tips for Common Effects

#### Effect Performance Best Practices

```cpp
// 1. Cache effect instances instead of creating new ones
class OptimizedEffectComponent : public visage::Frame {
private:
    // Good: Reuse effect instances
    std::unique_ptr<visage::BlurEffect> cachedBlur_;
    std::unique_ptr<visage::BloomEffect> cachedBloom_;
    
    // Bad: Creating effects in draw()
    void badDraw(visage::Canvas& canvas) {
        canvas.pushEffect(std::make_unique<visage::BlurEffect>(10.0f)); // Allocates every frame!
    }
    
    // Good: Reuse cached effects
    void goodDraw(visage::Canvas& canvas) {
        canvas.pushEffect(cachedBlur_.get());
    }
};

// 2. Use level-of-detail for effects
class LODEffectComponent : public visage::Frame {
    void draw(visage::Canvas& canvas) override {
        float quality = getQualityLevel();
        
        if (quality >= 1.0f) {
            // High quality: All effects
            canvas.pushEffect(shadowEffect_.get());
            canvas.pushEffect(blurEffect_.get());
            canvas.pushEffect(bloomEffect_.get());
        } else if (quality >= 0.5f) {
            // Medium: Skip expensive effects
            canvas.pushEffect(shadowEffect_.get());
        }
        // Low quality: No effects
        
        drawContent(canvas);
        
        // Pop effects based on quality level
    }
};

// 3. Batch similar effects
class BatchedEffects : public visage::Frame {
    void draw(visage::Canvas& canvas) override {
        // Good: Apply blur once to multiple elements
        canvas.pushEffect(blurEffect_.get());
        drawAllBlurredElements(canvas);
        canvas.popEffect();
        
        // Bad: Applying blur to each element separately
        for (auto& element : elements_) {
            canvas.pushEffect(blurEffect_.get());
            element->draw(canvas);
            canvas.popEffect();
        }
    }
};

// 4. Use render-to-texture for complex static content
class CachedComplexContent : public visage::Frame {
private:
    std::unique_ptr<visage::RenderTexture> cachedTexture_;
    bool needsRedraw_ = true;
    
public:
    void invalidateCache() {
        needsRedraw_ = true;
    }
    
    void draw(visage::Canvas& canvas) override {
        if (needsRedraw_) {
            // Render to texture
            cachedTexture_ = std::make_unique<visage::RenderTexture>(width(), height());
            visage::Canvas textureCanvas;
            textureCanvas.setRenderTarget(cachedTexture_.get());
            
            // Draw complex content once
            drawComplexContent(textureCanvas);
            
            textureCanvas.submit();
            needsRedraw_ = false;
        }
        
        // Draw cached texture
        canvas.drawTexture(*cachedTexture_, 0, 0);
    }
};

// 5. Optimize animation frame rates
class AdaptiveAnimation : public visage::Frame, public juce::Timer {
private:
    std::chrono::steady_clock::time_point lastFrame_;
    float targetFPS_ = 60.0f;
    float currentFPS_ = 60.0f;
    
public:
    void startAdaptiveAnimation() {
        lastFrame_ = std::chrono::steady_clock::now();
        startTimer(16); // Start at 60fps
    }
    
    void timerCallback() override {
        auto now = std::chrono::steady_clock::now();
        auto frameTime = std::chrono::duration<float, std::milli>(now - lastFrame_).count();
        lastFrame_ = now;
        
        // Calculate actual FPS
        currentFPS_ = 1000.0f / frameTime;
        
        // Adapt animation complexity based on performance
        if (currentFPS_ < targetFPS_ * 0.9f) {
            // Reduce quality or skip frames
            reduceAnimationQuality();
        } else if (currentFPS_ > targetFPS_ * 0.95f) {
            // Can increase quality
            increaseAnimationQuality();
        }
        
        updateAnimation(frameTime);
        redraw();
    }
};
```

### Learning from Popular Libraries

#### Dear ImGui Pattern: Immediate Mode

```cpp
// Immediate mode inspired pattern for Visage
class ImmediateModeUI {
    static bool Button(visage::Canvas& canvas, const std::string& label, 
                      float x, float y, float w, float h) {
        static std::map<std::string, bool> buttonStates;
        bool& isHovered = buttonStates[label + "_hover"];
        bool& isPressed = buttonStates[label + "_press"];
        
        // Check mouse interaction
        auto mousePos = getMousePosition();
        bool inBounds = mousePos.x >= x && mousePos.x <= x + w &&
                       mousePos.y >= y && mousePos.y <= y + h;
        
        isHovered = inBounds;
        if (inBounds && isMousePressed()) {
            isPressed = true;
        } else if (!isMousePressed()) {
            isPressed = false;
        }
        
        // Draw button
        uint32_t color = isPressed ? 0xff1976d2 : (isHovered ? 0xff2196f3 : 0xff42a5f5);
        canvas.setColor(color);
        canvas.roundedRect(x, y, w, h, 4.0f);
        
        // Draw label
        auto* font = FontManager::instance().getFont(FontManager::FontType::Regular, 14.0f);
        canvas.setColor(0xffffffff);
        visage::String text(label.c_str());
        canvas.text(text, *font, visage::Font::kCenter, x, y, w, h);
        
        return isPressed && !isMousePressed(); // Click on release
    }
};
```

#### Flutter Pattern: Widget Composition

```cpp
// Flutter-inspired widget composition
class VisageWidget {
public:
    virtual void build(visage::Canvas& canvas) = 0;
};

class Container : public VisageWidget {
private:
    std::unique_ptr<VisageWidget> child_;
    float padding_ = 0.0f;
    uint32_t color_ = 0x00000000;
    float borderRadius_ = 0.0f;
    
public:
    Container& padding(float p) { padding_ = p; return *this; }
    Container& color(uint32_t c) { color_ = c; return *this; }
    Container& borderRadius(float r) { borderRadius_ = r; return *this; }
    Container& child(std::unique_ptr<VisageWidget> c) { 
        child_ = std::move(c); 
        return *this; 
    }
    
    void build(visage::Canvas& canvas) override {
        if (color_ != 0x00000000) {
            canvas.setColor(color_);
            if (borderRadius_ > 0) {
                canvas.roundedRect(0, 0, width(), height(), borderRadius_);
            } else {
                canvas.fill(0, 0, width(), height());
            }
        }
        
        if (child_) {
            canvas.save();
            canvas.translate(padding_, padding_);
            child_->build(canvas);
            canvas.restore();
        }
    }
};
```

#### Three.js Pattern: Effect Composer

```cpp
// Three.js inspired effect composer
class EffectComposer {
private:
    std::vector<std::unique_ptr<visage::PostEffect>> effectChain_;
    std::unique_ptr<visage::RenderTexture> pingTexture_;
    std::unique_ptr<visage::RenderTexture> pongTexture_;
    
public:
    void addEffect(std::unique_ptr<visage::PostEffect> effect) {
        effectChain_.push_back(std::move(effect));
    }
    
    void render(visage::Canvas& canvas, visage::Frame* content) {
        // Initialize render textures if needed
        if (!pingTexture_) {
            pingTexture_ = std::make_unique<visage::RenderTexture>(
                content->width(), content->height());
            pongTexture_ = std::make_unique<visage::RenderTexture>(
                content->width(), content->height());
        }
        
        // Render initial content to first texture
        visage::Canvas tempCanvas;
        tempCanvas.setRenderTarget(pingTexture_.get());
        content->draw(tempCanvas);
        tempCanvas.submit();
        
        // Apply effects in sequence
        bool usePing = false;
        for (size_t i = 0; i < effectChain_.size(); ++i) {
            auto& effect = effectChain_[i];
            
            tempCanvas.setRenderTarget(usePing ? pongTexture_.get() : pingTexture_.get());
            tempCanvas.pushEffect(effect.get());
            tempCanvas.drawTexture(*(usePing ? pingTexture_ : pongTexture_), 0, 0);
            tempCanvas.popEffect();
            tempCanvas.submit();
            
            usePing = !usePing;
        }
        
        // Draw final result
        canvas.drawTexture(*(usePing ? pingTexture_ : pongTexture_), 0, 0);
    }
};
```

#### Skia Pattern: Path Effects

```cpp
// Skia-inspired path effects
class PathEffect {
public:
    virtual void applyToPath(std::vector<visage::Point>& points) = 0;
};

class DashPathEffect : public PathEffect {
private:
    float dashLength_ = 10.0f;
    float gapLength_ = 5.0f;
    
public:
    void applyToPath(std::vector<visage::Point>& points) override {
        std::vector<visage::Point> dashedPoints;
        float distance = 0.0f;
        bool drawing = true;
        
        for (size_t i = 1; i < points.size(); ++i) {
            visage::Point p1 = points[i - 1];
            visage::Point p2 = points[i];
            
            float segmentLength = std::sqrt(
                (p2.x - p1.x) * (p2.x - p1.x) + 
                (p2.y - p1.y) * (p2.y - p1.y)
            );
            
            float traveled = 0.0f;
            while (traveled < segmentLength) {
                float remaining = drawing ? dashLength_ : gapLength_;
                float step = std::min(remaining - distance, segmentLength - traveled);
                
                float t = (traveled + step) / segmentLength;
                visage::Point interpolated{
                    p1.x + (p2.x - p1.x) * t,
                    p1.y + (p2.y - p1.y) * t
                };
                
                if (drawing) {
                    dashedPoints.push_back(interpolated);
                }
                
                distance += step;
                traveled += step;
                
                if (distance >= remaining) {
                    drawing = !drawing;
                    distance = 0.0f;
                }
            }
        }
        
        points = dashedPoints;
    }
};
```


## Build System Integration

### CMake Configuration
```cmake
option(USE_VISAGE_UI "Use Visage UI instead of JUCE" ON)

if(USE_VISAGE_UI)
    add_subdirectory(visage)
    target_link_libraries(${PROJECT_NAME} PRIVATE visage)
    target_compile_definitions(${PROJECT_NAME} PRIVATE USE_VISAGE_UI=1)
endif()
```

### Conditional Compilation
```cpp
#if USE_VISAGE_UI
    #include "VisagePluginEditor.h"
    using EditorType = VisagePluginEditor;
#else
    #include "JUCEPluginEditor.h" 
    using EditorType = JUCEPluginEditor;
#endif
```

## Development Workflow

### Migration Decision Trees

When to Use Pure Visage vs Hybrid Approach:

**High Priority** (Migrate First):
- Visualizers (waveforms, spectrums) - Huge GPU benefit
- Animated elements - Smoother with Visage
- Complex custom graphics - Easier in Visage

**Medium Priority**:
- Standard controls (knobs, sliders) - Good learning projects
- Meters and displays - Performance gains

**Low Priority** (Keep in JUCE):
- File browsers - JUCE's is battle-tested
- Menu bars - Platform-specific behavior
- Tooltips - JUCE handles accessibility better

### 1. Infrastructure First
- Set up window embedding/bridge
- Implement theme system
- Create font management
- Test basic rendering

### 2. Component Migration Order
1. Simple components (buttons, labels)
2. Layout containers 
3. Complex widgets (combo boxes, text editors)
4. Custom drawing components
5. Overlays and dialogs

### 3. Testing Strategy
- Build frequently during migration
- Test in multiple DAWs
- Verify all keyboard shortcuts work
- Check memory management
- Profile performance vs JUCE

## Feature Flag Migration Strategy

### Phase 1: Foundation
**Goal**: Establish infrastructure without user-facing changes

```cmake
# Phase 1 Feature Flags
set(USE_VISAGE_UI ON)
set(ENABLE_VISAGE_MIGRATION ON)
set(MIGRATE_KNOBS_TO_VISAGE OFF)  # All component migrations OFF
set(ENABLE_AB_TESTING ON)
set(VISAGE_DEBUG_RENDERING ON)
```

**Tasks**:
- Set up build system with feature flags
- Implement JuceVisageBridge
- Create wrapper templates
- Establish performance monitoring
- No visual changes to users

### Phase 2: Low-Risk Components
**Goal**: Migrate simple, non-critical components

```cmake
# Phase 2 Feature Flags
set(MIGRATE_BUTTONS_TO_VISAGE ON)  # Start with buttons
set(AB_TEST_BUTTON_RATIO 0.1)      # 10% of users
```

**Rollout Strategy**:
```cpp
// Gradual rollout configuration
class MigrationSchedule {
    static constexpr float getButtonRolloutPercentage(int days_since_launch) {
        if (days_since_launch < 3) return 0.1f;      // 10% for 3 days
        if (days_since_launch < 7) return 0.25f;     // 25% for next 4 days
        if (days_since_launch < 14) return 0.5f;     // 50% for week 2
        return 1.0f;                                  // 100% after 2 weeks
    }
};
```

### Phase 3: Performance-Critical Components
**Goal**: Migrate components that benefit most from GPU acceleration

```cmake
# Phase 3 Feature Flags
set(MIGRATE_KNOBS_TO_VISAGE ON)
set(MIGRATE_SLIDERS_TO_VISAGE ON)
set(MIGRATE_DISPLAYS_TO_VISAGE ON)  # Waveforms, meters, etc.
```

### Phase 4: Visual Effects
**Goal**: Enable GPU effects for enhanced visuals

```cmake
# Phase 4 Feature Flags
set(MIGRATE_EFFECTS_TO_VISAGE ON)
set(ENABLE_ADVANCED_EFFECTS ON)
```

### Phase 5: Complete Migration
**Goal**: Remove JUCE UI dependencies

```cmake
# Phase 5 - Final configuration
set(VISAGE_MIGRATION_LEVEL "COMPLETE")
set(ENABLE_JUCE_FALLBACK OFF)  # Remove JUCE UI code
```

### Rollback Procedures

```cpp
// Emergency rollback system
class MigrationRollback {
public:
    enum class RollbackLevel {
        None,
        CurrentPhase,    // Roll back current phase only
        LastStable,      // Roll back to last stable phase
        Complete         // Roll back entire migration
    };
    
    static void executeRollback(RollbackLevel level) {
        switch (level) {
            case RollbackLevel::CurrentPhase:
                // Disable current phase flags via environment
                setenv("MIGRATE_CURRENT_PHASE", "OFF", 1);
                break;
                
            case RollbackLevel::LastStable:
                // Revert to previous phase configuration
                loadConfiguration("last_stable_phase.cmake");
                break;
                
            case RollbackLevel::Complete:
                // Disable all Visage features
                setenv("USE_VISAGE_UI", "OFF", 1);
                setenv("FORCE_JUCE_UI", "ON", 1);
                break;
        }
    }
};
```

### Monitoring & Success Metrics

```cpp
class MigrationMetrics {
    struct PhaseMetrics {
        // Performance metrics
        float avg_frame_time_ms;
        float p99_frame_time_ms;
        int gpu_memory_mb;
        float cpu_usage_percent;
        
        // Stability metrics
        int crash_count;
        int error_count;
        float uptime_hours;
        
        // User engagement metrics
        int user_interactions;
        float session_duration;
        int feature_usage_count;
        
        // Quality metrics
        int visual_glitch_reports;
        int performance_complaints;
        float user_satisfaction_score;
    };
    
    static bool shouldProceedToNextPhase(const PhaseMetrics& current, 
                                        const PhaseMetrics& baseline) {
        // Performance must not degrade more than 5%
        if (current.avg_frame_time_ms > baseline.avg_frame_time_ms * 1.05f) {
            return false;
        }
        
        // Crash rate must not increase
        if (current.crash_count > baseline.crash_count) {
            return false;
        }
        
        // User engagement must remain stable
        if (current.user_interactions < baseline.user_interactions * 0.95f) {
            return false;
        }
        
        return true;
    }
};
```

## Debugging Common Issues

### Black/Empty Interface
- Check renderer initialization with proper window handle
- Verify canvas.pairToWindow() called with valid handles
- Ensure swap chain support available
- Add logging to track rendering pipeline

### Debug Overlay for Development
```cpp
// VisageDebugLayer.h
class VisageDebugLayer {
public:
    static void enableDebugMode() {
        #if DEBUG
        // Enable GPU timing
        visage::Renderer::instance().enableGPUTiming(true);
        
        // Add debug overlay
        auto debugOverlay = std::make_unique<DebugOverlay>();
        getRootFrame()->addChild(debugOverlay.get());
        #endif
    }
    
    class DebugOverlay : public visage::Frame {
        struct FrameStats {
            float fps = 0.0f;
            float gpuTime = 0.0f;
            int drawCalls = 0;
            size_t memoryUsage = 0;
        };
        
        FrameStats stats_;
        
    public:
        void draw(visage::Canvas& canvas) override {
            // Update stats
            updateStats();
            
            // Draw debug info
            canvas.save();
            canvas.setColor(0x80000000); // Semi-transparent black
            canvas.fill(10, 10, 200, 100);
            
            auto* font = FontManager::instance().getFont(
                FontManager::FontType::Regular, 12.0f);
            canvas.setColor(0xff00ff00); // Green text
            
            drawDebugText(canvas, font, 15, 25, 
                "FPS: " + std::to_string(static_cast<int>(stats_.fps)));
            drawDebugText(canvas, font, 15, 40, 
                "GPU: " + std::to_string(stats_.gpuTime) + "ms");
            drawDebugText(canvas, font, 15, 55, 
                "Draw Calls: " + std::to_string(stats_.drawCalls));
            drawDebugText(canvas, font, 15, 70, 
                "Memory: " + formatBytes(stats_.memoryUsage));
            
            canvas.restore();
        }
    };
};
```

### Crashes on Startup
- Check palette/theme initialization order
- Verify Frame::init() called before setPalette()
- Check for static initialization order issues
- Validate window handle before passing to Visage

### Events Not Working
- Verify bridge translates coordinates correctly
- Check modifier key mapping between frameworks
- Ensure focus management works between JUCE/Visage
- Test mouse capture and release

### Memory Leaks in Plugin Unload
**Solution**:
```cpp
~VisagePluginEditor() {
    // Proper cleanup order is critical
    stopTimer();
    
    // Remove all Visage children first
    if (rootVisageFrame) {
        rootVisageFrame->removeAllChildren();
    }
    
    // Clear any cached resources
    ResourceManager::instance().clearAll();
    
    // Finally destroy the Visage context
    visageWindow.reset();
}
```

## Platform-Specific Considerations

### macOS
- Use NSView* for parent handle
- Handle Retina scaling properly
- Metal rendering backend

### Windows  
- Use HWND for parent handle
- Set WS_CHILD flag for plugin windows
- DirectX 11/12 rendering backend

### Linux
- Use X11 window ID for parent handle
- Handle DPI scaling
- Vulkan rendering backend

## Testing & Validation

### Unit Testing Components
[existing unit test content]

### Effect-Specific Testing

#### Effect Performance Tests
```cpp
class EffectPerformanceTests : public juce::UnitTest {
public:
    EffectPerformanceTests() : UnitTest("Visage Effect Performance") {}
    
    void runTest() override {
        beginTest("Single Effect Performance");
        {
            auto testFrame = createTestFrame();
            
            // Test each effect type
            testEffectPerformance<visage::BlurEffect>("Blur", 5.0f);
            testEffectPerformance<visage::BloomEffect>("Bloom");
            testEffectPerformance<ChromaticAberrationEffect>("Chromatic");
            
            // Ensure no effect takes more than 2ms
            for (const auto& [name, time] : effect_times_) {
                expect(time < 2.0f, name + " took " + String(time) + "ms");
            }
        }
        
        beginTest("Effect Stacking Performance");
        {
            auto canvas = createTestCanvas();
            auto testFrame = createTestFrame();
            
            // Test multiple effects
            auto blur = std::make_unique<visage::BlurEffect>(5.0f);
            auto bloom = std::make_unique<visage::BloomEffect>();
            auto grain = std::make_unique<FilmGrainEffect>();
            
            auto start = std::chrono::high_resolution_clock::now();
            
            canvas.pushEffect(blur.get());
            canvas.pushEffect(bloom.get());
            canvas.pushEffect(grain.get());
            testFrame->draw(canvas);
            canvas.popEffect();
            canvas.popEffect();
            canvas.popEffect();
            canvas.submit();
            
            auto end = std::chrono::high_resolution_clock::now();
            auto duration = std::chrono::duration<float, std::milli>(end - start).count();
            
            // Three effects should complete in under 5ms
            expect(duration < 5.0f, "Stacked effects took " + String(duration) + "ms");
        }
        
        beginTest("Effect Memory Usage");
        {
            size_t initial_memory = getGPUMemoryUsage();
            
            // Create many effects
            std::vector<std::unique_ptr<visage::PostEffect>> effects;
            for (int i = 0; i < 100; ++i) {
                effects.push_back(std::make_unique<visage::BlurEffect>(10.0f));
            }
            
            size_t peak_memory = getGPUMemoryUsage();
            size_t memory_per_effect = (peak_memory - initial_memory) / 100;
            
            // Each blur effect should use less than 1MB
            expect(memory_per_effect < 1024 * 1024, 
                   "Blur effect uses " + String(memory_per_effect / 1024) + "KB");
        }
    }
    
private:
    std::map<std::string, float> effect_times_;
    
    template<typename EffectType, typename... Args>
    void testEffectPerformance(const std::string& name, Args&&... args) {
        auto effect = std::make_unique<EffectType>(std::forward<Args>(args)...);
        auto testFrame = createTestFrame();
        visage::Canvas canvas;
        
        // Warm up
        for (int i = 0; i < 5; ++i) {
            canvas.pushEffect(effect.get());
            testFrame->draw(canvas);
            canvas.popEffect();
            canvas.submit();
        }
        
        // Measure
        auto start = std::chrono::high_resolution_clock::now();
        
        canvas.pushEffect(effect.get());
        testFrame->draw(canvas);
        canvas.popEffect();
        canvas.submit();
        visage::Renderer::instance().frame(); // Wait for GPU
        
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration<float, std::milli>(end - start).count();
        
        effect_times_[name] = duration;
    }
    
    std::unique_ptr<visage::Frame> createTestFrame() {
        auto frame = std::make_unique<TestContentFrame>();
        frame->setBounds(0, 0, 800, 600);
        return frame;
    }
};
```

#### Effect Visual Regression Tests
```cpp
class EffectVisualTests : public juce::UnitTest {
public:
    EffectVisualTests() : UnitTest("Visage Effect Visual Tests") {}
    
    void runTest() override {
        beginTest("Effect Output Consistency");
        {
            // Render same content with same effect parameters
            auto hash1 = renderAndHash<visage::BlurEffect>(5.0f);
            auto hash2 = renderAndHash<visage::BlurEffect>(5.0f);
            
            expectEquals(hash1, hash2, "Blur effect not deterministic");
        }
        
        beginTest("Effect Parameter Validation");
        {
            auto blur = std::make_unique<visage::BlurEffect>();
            
            // Test parameter clamping
            blur->setBlurSize(-5.0f);
            expect(blur->getBlurSize() >= 0.0f, "Negative blur size not clamped");
            
            blur->setBlurSize(1000.0f);
            expect(blur->getBlurSize() <= 100.0f, "Excessive blur size not clamped");
        }
        
        beginTest("Effect Combination Rendering");
        {
            // Test that problematic combinations don't crash
            auto testFrame = createTestFrame();
            visage::Canvas canvas;
            
            // Extreme parameters
            auto blur = std::make_unique<visage::BlurEffect>(50.0f);
            auto bloom = std::make_unique<visage::BloomEffect>();
            bloom->setIntensity(10.0f);
            
            // Should not crash or hang
            bool completed = false;
            std::thread render_thread([&]() {
                canvas.pushEffect(blur.get());
                canvas.pushEffect(bloom.get());
                testFrame->draw(canvas);
                canvas.popEffect();
                canvas.popEffect();
                canvas.submit();
                completed = true;
            });
            
            render_thread.join();
            expect(completed, "Effect combination caused hang");
        }
    }
    
private:
    template<typename EffectType, typename... Args>
    uint64_t renderAndHash(Args&&... args) {
        auto effect = std::make_unique<EffectType>(std::forward<Args>(args)...);
        auto testFrame = createTestFrame();
        
        // Render to texture
        visage::RenderTarget target(256, 256);
        visage::Canvas canvas;
        canvas.setRenderTarget(&target);
        
        canvas.pushEffect(effect.get());
        testFrame->draw(canvas);
        canvas.popEffect();
        canvas.submit();
        
        // Get pixel data and compute hash
        auto pixels = target.readPixels();
        return computeHash(pixels);
    }
    
    uint64_t computeHash(const std::vector<uint8_t>& pixels) {
        // Simple hash for regression testing
        uint64_t hash = 0;
        for (size_t i = 0; i < pixels.size(); i += 4) {
            hash ^= (uint64_t(pixels[i]) << ((i % 8) * 8));
        }
        return hash;
    }
};
```

#### Effect Platform Compatibility Tests
```cpp
class EffectCompatibilityTests : public juce::UnitTest {
public:
    EffectCompatibilityTests() : UnitTest("Effect Platform Compatibility") {}
    
    void runTest() override {
        beginTest("Platform Feature Detection");
        {
            auto& renderer = visage::Renderer::instance();
            
            // Log platform capabilities
            DBG("Platform: " << getPlatformName());
            DBG("GPU: " << renderer.getGPUName());
            DBG("Max Texture Size: " << renderer.getMaxTextureSize());
            DBG("Compute Shaders: " << renderer.supportsComputeShaders());
            DBG("HDR: " << renderer.isHDREnabled());
            
            // Ensure minimum capabilities
            expect(renderer.getMaxTextureSize() >= 2048, 
                   "GPU max texture size below minimum");
        }
        
        beginTest("Effect Fallback Testing");
        {
            // Test that effects have appropriate fallbacks
            std::vector<std::unique_ptr<visage::PostEffect>> effects;
            
            effects.push_back(createEffectWithFallback("AdvancedBloom"));
            effects.push_back(createEffectWithFallback("ComputeBlur"));
            effects.push_back(createEffectWithFallback("TessellationWarp"));
            
            for (const auto& effect : effects) {
                expect(effect != nullptr, "Effect creation failed without fallback");
                
                // Test that effect renders without crashing
                testEffectRender(effect.get());
            }
        }
        
        beginTest("Multi-GPU Consistency");
        {
#ifdef ENABLE_MULTI_GPU_TESTING
            auto& renderer = visage::Renderer::instance();
            int gpu_count = renderer.getGPUCount();
            
            if (gpu_count > 1) {
                // Test rendering on each GPU
                for (int i = 0; i < gpu_count; ++i) {
                    renderer.setActiveGPU(i);
                    
                    auto hash = renderTestPattern();
                    DBG("GPU " << i << " render hash: " << hash);
                    
                    // Hashes should be similar (allowing for minor precision differences)
                    if (i > 0) {
                        expect(hashesAreSimilar(previous_hash_, hash),
                               "GPU rendering inconsistency");
                    }
                    previous_hash_ = hash;
                }
            }
#endif
        }
    }
    
private:
    uint64_t previous_hash_ = 0;
    
    std::unique_ptr<visage::PostEffect> createEffectWithFallback(const std::string& type) {
        try {
            return EffectFactory::create(type);
        } catch (const std::exception& e) {
            DBG("Effect " << type << " not available: " << e.what());
            DBG("Using fallback");
            return EffectFactory::createFallback(type);
        }
    }
    
    void testEffectRender(visage::PostEffect* effect) {
        auto testFrame = createTestFrame();
        visage::Canvas canvas;
        
        bool success = false;
        try {
            canvas.pushEffect(effect);
            testFrame->draw(canvas);
            canvas.popEffect();
            canvas.submit();
            success = true;
        } catch (const std::exception& e) {
            DBG("Effect render failed: " << e.what());
        }
        
        expect(success, "Effect rendering failed");
    }
    
    bool hashesAreSimilar(uint64_t a, uint64_t b) {
        // Allow up to 5% difference for GPU precision variations
        uint64_t diff = (a > b) ? (a - b) : (b - a);
        return diff < (a / 20);
    }
};
```

#### Effect Stress Tests
```cpp
class EffectStressTests : public juce::UnitTest {
public:
    EffectStressTests() : UnitTest("Effect Stress Testing") {}
    
    void runTest() override {
        beginTest("Rapid Effect Switching");
        {
            auto testFrame = createTestFrame();
            visage::Canvas canvas;
            
            // Create effect pool
            std::vector<std::unique_ptr<visage::PostEffect>> effects;
            effects.push_back(std::make_unique<visage::BlurEffect>(5.0f));
            effects.push_back(std::make_unique<visage::BloomEffect>());
            effects.push_back(std::make_unique<ChromaticAberrationEffect>());
            
            // Rapidly switch effects
            auto start = std::chrono::steady_clock::now();
            int frame_count = 0;
            
            while (std::chrono::steady_clock::now() - start < std::chrono::seconds(5)) {
                auto& effect = effects[frame_count % effects.size()];
                
                canvas.pushEffect(effect.get());
                testFrame->draw(canvas);
                canvas.popEffect();
                canvas.submit();
                
                frame_count++;
            }
            
            float fps = frame_count / 5.0f;
            expect(fps > 30.0f, "Effect switching performance below 30fps: " + String(fps));
        }
        
        beginTest("Maximum Effect Stack");
        {
            auto testFrame = createTestFrame();
            visage::Canvas canvas;
            
            // Stack many effects
            std::vector<std::unique_ptr<visage::PostEffect>> effect_stack;
            int max_effects = 0;
            
            try {
                for (int i = 0; i < 50; ++i) {
                    auto effect = std::make_unique<visage::BlurEffect>(1.0f);
                    canvas.pushEffect(effect.get());
                    effect_stack.push_back(std::move(effect));
                    max_effects = i + 1;
                }
                
                testFrame->draw(canvas);
                
                // Pop all effects
                for (int i = 0; i < max_effects; ++i) {
                    canvas.popEffect();
                }
                
                canvas.submit();
            } catch (const std::exception& e) {
                DBG("Effect stack limit reached at " << max_effects << ": " << e.what());
            }
            
            expect(max_effects >= 8, "Effect stack limit too low: " + String(max_effects));
        }
        
        beginTest("Effect Memory Pressure");
        {
            size_t initial_memory = getGPUMemoryUsage();
            std::vector<std::unique_ptr<visage::PostEffect>> effects;
            
            // Create effects until memory pressure
            try {
                for (int i = 0; i < 1000; ++i) {
                    // High memory effect - large blur
                    auto blur = std::make_unique<visage::BlurEffect>(50.0f);
                    effects.push_back(std::move(blur));
                    
                    if (i % 100 == 0) {
                        size_t current_memory = getGPUMemoryUsage();
                        DBG("Created " << i << " effects, memory: " << 
                            (current_memory - initial_memory) / (1024 * 1024) << "MB");
                    }
                }
            } catch (const std::exception& e) {
                DBG("Memory limit reached: " << e.what());
            }
            
            // Should handle memory pressure gracefully
            expect(effects.size() > 50, "Too few effects before memory limit");
        }
    }
};
```

### Integration Testing Checklist
- [ ] Test in all major DAWs (Ableton, Logic, Reaper, etc.)
- [ ] Verify window resizing behavior
- [ ] Check preset loading/saving with UI state
- [ ] Test automation recording/playback
- [ ] Verify CPU/GPU usage under stress
- [ ] Check for memory leaks during long sessions
- [ ] **Test effect combinations in real-world scenarios**
- [ ] **Verify effect parameter automation**
- [ ] **Check effect performance on minimum spec hardware**
- [ ] **Test effect behavior during audio dropouts**
- [ ] **Verify effect state persistence across sessions**

### Effect-Specific DAW Testing
```cpp
class EffectDAWIntegrationTests {
    void testEffectsInDAW(const std::string& daw_name) {
        // Test that effects work correctly in each DAW
        
        // 1. Load plugin with effects enabled
        // 2. Automate effect parameters
        // 3. Save and reload project
        // 4. Verify effects still work
        // 5. Test during high CPU load
        // 6. Check GPU acceleration is active
        // 7. Verify no conflicts with DAW's graphics
    }
    
    void testSpecificScenarios() {
        // Ableton Live: Test with multiple plugin instances
        // Logic Pro: Test with plugin window resizing
        // Pro Tools: Test with HDX hardware acceleration
        // Reaper: Test with custom themes
        // Studio One: Test with high DPI displays
    }
};
```

## Success Metrics

When migration is complete, you should have:
- ✅ Pixel-perfect visual parity with JUCE version
- ✅ All functionality working (drag/drop, keyboard shortcuts, etc.)
- ✅ Better performance due to GPU acceleration  
- ✅ Cleaner code with modern C++ patterns
- ✅ Hot-reloadable themes for easy customization
- ✅ No memory leaks or crashes
- ✅ Works in all target DAWs

## Detailed Implementation Guidelines for Key Visage Principles

### 1. Visage is Shader-First

**What this means for AI implementation:**

Visage converts all drawing operations to GPU shaders using signed distance fields (SDFs). This fundamental difference affects how you approach visual problems.

#### Implementation Implications:

```cpp
// JUCE Approach (CPU-based, won't work well in Visage):
void drawComplexClippingPath(Graphics& g) {
    Path complexPath;
    complexPath.addStar(Point(50, 50), 7, 20, 40);
    g.reduceClipRegion(complexPath);
    g.fillAll(Colours::blue);
}

// Visage Approach (Shader-friendly):
class StarShape : public visage::Frame {
    void draw(visage::Canvas& canvas) override {
        // Use GPU-friendly primitives
        canvas.pushMask();
        drawStarUsingSDF(canvas); // Draw star as SDF
        canvas.popMask();
        
        canvas.setColor(0xff0000ff);
        canvas.fill(0, 0, width(), height());
    }
    
    void drawStarUsingSDF(visage::Canvas& canvas) {
        // Implement star using triangles or a custom shader
        // SDFs work best with mathematical descriptions
        float centerX = width() / 2;
        float centerY = height() / 2;
        
        // Draw star using triangle fan
        std::vector<Point> points = generateStarPoints(centerX, centerY, 7, 20, 40);
        for (size_t i = 1; i < points.size(); ++i) {
            canvas.triangle(centerX, centerY,
                          points[i-1].x, points[i-1].y,
                          points[i].x, points[i].y);
        }
    }
};
```

**Key Strategies:**
- Think in terms of mathematical functions that can be evaluated per-pixel
- Complex shapes should be broken down into GPU-friendly primitives
- Use masking and blending instead of complex clipping paths
- Leverage the GPU's parallel processing for effects

```cpp
// Custom shader effect example
class WaveDistortionEffect : public visage::Effect {
    std::string getFragmentShader() override {
        return R"(
            uniform float time;
            uniform float amplitude;
            
            vec4 effect(vec4 color, vec2 uv) {
                float wave = sin(uv.x * 10.0 + time) * amplitude;
                uv.y += wave;
                return texture(inputTexture, uv) * color;
            }
        )";
    }
};
```

### 2. Minimal by Design

**What this means for AI implementation:**

Visage provides only core rendering primitives. You must build higher-level components yourself, which gives you complete control but requires more implementation work.

#### Implementation Strategy:

```cpp
// Build a component library gradually
namespace VisageComponents {
    
    // Base class for all custom components
    class Component : public visage::Frame {
    public:
        // Add common functionality all components need
        void setEnabled(bool enabled) { 
            enabled_ = enabled; 
            updateAppearance();
        }
        
        virtual void updateAppearance() = 0;
        
    protected:
        bool enabled_ = true;
        bool mouseOver_ = false;
        bool mouseDown_ = false;
    };
    
    // Example: Building a ComboBox from scratch
    class ComboBox : public Component {
    private:
        std::vector<std::string> items_;
        int selectedIndex_ = -1;
        bool isOpen_ = false;
        std::unique_ptr<PopupMenu> popup_;
        
    public:
        void addItem(const std::string& item) {
            items_.push_back(item);
        }
        
        void draw(visage::Canvas& canvas) override {
            // Draw main button
            drawButton(canvas);
            
            // Draw dropdown arrow
            drawDropdownArrow(canvas);
            
            // Draw selected text
            if (selectedIndex_ >= 0) {
                drawText(canvas, items_[selectedIndex_]);
            }
        }
        
        void onMouseDown(const visage::MouseEvent& e) override {
            if (!enabled_) return;
            
            if (!isOpen_) {
                showPopup();
            } else {
                hidePopup();
            }
        }
        
    private:
        void showPopup() {
            popup_ = std::make_unique<PopupMenu>();
            for (size_t i = 0; i < items_.size(); ++i) {
                popup_->addItem(items_[i], [this, i]() {
                    selectedIndex_ = i;
                    hidePopup();
                    if (onChange) onChange(selectedIndex_);
                });
            }
            
            // Position below this component
            auto globalPos = localToGlobal(Point(0, height()));
            popup_->show(globalPos.x, globalPos.y);
            isOpen_ = true;
        }
    };
}
```

**Component Building Checklist:**
- Start with basic interaction (mouse, keyboard)
- Add visual states (normal, hover, pressed, disabled)
- Implement accessibility hooks
- Add animation support
- Create consistent API patterns
- Document usage examples

### 3. Effects Are First-Class

**What this means for AI implementation:**

Unlike JUCE where effects are complex to implement, Visage makes GPU effects a core feature. Design your components to take advantage of this.

#### Implementation Patterns:

```cpp
// Effect-based component design
class GlowButton : public visage::Button {
private:
    float glowIntensity_ = 0.0f;
    std::unique_ptr<visage::BloomEffect> bloomEffect_;
    
public:
    GlowButton() {
        bloomEffect_ = std::make_unique<visage::BloomEffect>(0.0f);
    }
    
    void draw(visage::Canvas& canvas) override {
        // Animate glow on hover
        float targetGlow = mouseOver_ ? 1.0f : 0.0f;
        glowIntensity_ += (targetGlow - glowIntensity_) * 0.1f;
        
        bloomEffect_->setIntensity(glowIntensity_ * 0.5f);
        
        // Apply effect to button rendering
        canvas.pushEffect(bloomEffect_.get());
        drawButtonContent(canvas);
        canvas.popEffect();
    }
};

// Layered effects for complex visuals
class FuturisticPanel : public visage::Frame {
    void draw(visage::Canvas& canvas) override {
        // Layer 1: Blurred background
        canvas.pushEffect(std::make_unique<visage::BlurEffect>(20.0f));
        canvas.setColor(0x40000000);
        canvas.fill(0, 0, width(), height());
        canvas.popEffect();
        
        // Layer 2: Main content with subtle glow
        canvas.pushEffect(std::make_unique<visage::BloomEffect>(0.2f));
        drawContent(canvas);
        canvas.popEffect();
        
        // Layer 3: Scanline effect overlay
        canvas.pushEffect(customScanlineEffect_.get());
        canvas.setColor(0x10ffffff);
        canvas.fill(0, 0, width(), height());
        canvas.popEffect();
    }
};
```

**Effect Optimization Strategies:**
```cpp
// Pool effects to avoid allocation
class EffectPool {
    std::vector<std::unique_ptr<visage::BlurEffect>> blurPool_;
    
public:
    visage::BlurEffect* getBlur(float radius) {
        // Find or create blur effect with this radius
        for (auto& blur : blurPool_) {
            if (blur->getRadius() == radius) {
                return blur.get();
            }
        }
        
        blurPool_.push_back(std::make_unique<visage::BlurEffect>(radius));
        return blurPool_.back().get();
    }
};
```

### 4. Modern C++ Throughout

**What this means for AI implementation:**

Use modern C++ features extensively. Visage is designed for C++17+ and expects modern idioms.

#### Modern Patterns to Use:

```cpp
// Use structured bindings
auto [width, height] = getComponentSize();
auto [x, y] = mouseEvent.position;

// Use if-init statements
if (auto* button = dynamic_cast<visage::Button*>(component); button != nullptr) {
    button->setEnabled(false);
}

// Use std::optional for nullable values
std::optional<Color> getThemeColor(const std::string& key) {
    if (auto it = colors_.find(key); it != colors_.end()) {
        return it->second;
    }
    return std::nullopt;
}

// Use variant for type-safe unions
using PropertyValue = std::variant<int, float, std::string, Color>;

class PropertyBag {
    std::unordered_map<std::string, PropertyValue> properties_;
    
public:
    template<typename T>
    std::optional<T> get(const std::string& key) const {
        if (auto it = properties_.find(key); it != properties_.end()) {
            if (auto* value = std::get_if<T>(&it->second)) {
                return *value;
            }
        }
        return std::nullopt;
    }
};

// Use concepts (C++20) where available
template<typename T>
concept Drawable = requires(T t, visage::Canvas& c) {
    { t.draw(c) } -> std::same_as<void>;
};

// Leverage constexpr for compile-time computation
constexpr Color interpolateColor(Color a, Color b, float t) {
    return Color{
        static_cast<uint8_t>(a.r + (b.r - a.r) * t),
        static_cast<uint8_t>(a.g + (b.g - a.g) * t),
        static_cast<uint8_t>(a.b + (b.b - a.b) * t),
        static_cast<uint8_t>(a.a + (b.a - a.a) * t)
    };
}
```

### 5. GPU State Management

**What this means for AI implementation:**

GPU state changes are expensive. Design your rendering code to minimize state changes and batch operations.

#### Batching Strategies:

```cpp
// Good: Batch similar operations
class BatchRenderer {
    struct DrawCommand {
        enum Type { RECT, CIRCLE, TEXT };
        Type type;
        Color color;
        union {
            struct { float x, y, w, h; } rect;
            struct { float x, y, r; } circle;
            struct { 
                float x, y, w, h; 
                std::string text;
                visage::Font* font;
            } text;
        };
    };
    
    std::vector<DrawCommand> commands_;
    
public:
    void addRect(float x, float y, float w, float h, Color color) {
        commands_.push_back({DrawCommand::RECT, color, {.rect = {x, y, w, h}}});
    }
    
    void flush(visage::Canvas& canvas) {
        // Sort by operation type and color to minimize state changes
        std::sort(commands_.begin(), commands_.end(), 
            [](const DrawCommand& a, const DrawCommand& b) {
                if (a.type != b.type) return a.type < b.type;
                return a.color < b.color;
            });
        
        // Execute batched commands
        Color currentColor = 0;
        for (const auto& cmd : commands_) {
            if (cmd.color != currentColor) {
                canvas.setColor(cmd.color);
                currentColor = cmd.color;
            }
            
            switch (cmd.type) {
                case DrawCommand::RECT:
                    canvas.fill(cmd.rect.x, cmd.rect.y, cmd.rect.w, cmd.rect.h);
                    break;
                // ... handle other types
            }
        }
        
        commands_.clear();
    }
};

// State preservation pattern
class StatePreservingRenderer {
    void drawComplexElement(visage::Canvas& canvas) {
        canvas.save(); // Save current state
        
        // Make state changes
        canvas.setColor(0xff0000ff);
        canvas.translate(10, 10);
        canvas.scale(2.0f, 2.0f);
        
        // Do drawing
        drawContent(canvas);
        
        canvas.restore(); // Restore previous state
    }
};
```

**GPU-Friendly Architecture:**
```cpp
// Design components to work with GPU state
class GPUFriendlyComponent : public visage::Frame {
    // Cache computed values to avoid per-frame calculation
    mutable bool geometryCacheValid_ = false;
    mutable std::vector<Triangle> triangleCache_;
    
    void updateGeometryCache() const {
        if (!geometryCacheValid_) {
            triangleCache_ = computeTriangles();
            geometryCacheValid_ = true;
        }
    }
    
    void draw(visage::Canvas& canvas) override {
        updateGeometryCache();
        
        // Draw all triangles in one batch
        canvas.setColor(fillColor_);
        for (const auto& tri : triangleCache_) {
            canvas.triangle(tri.p1, tri.p2, tri.p3);
        }
    }
    
    void boundsChanged() override {
        geometryCacheValid_ = false;
    }
};
```

## Key Takeaways for AI Assistants

1. **Start with the bridge** - Plugin window embedding is the most critical part
2. **Create missing components** - Visage is more minimal than JUCE
3. **Respect initialization order** - Frame hierarchy and palette setup matters
4. **Use incremental migration** - Port components one at a time
5. **Leverage GPU acceleration** - Design for transforms and batching
6. **Test frequently** - Build and test in DAWs throughout migration
7. **Document API differences** - Maintain compatibility layers where needed

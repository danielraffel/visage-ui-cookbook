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

### 6. Comprehensive Font Handling in Visage: Font System Architecture

Visage's font system differs significantly from JUCE's approach. Here's what AI assistants need to understand:

```cpp
// JUCE: Fonts are system-dependent or loaded from files
Font juceFont("Arial", 14.0f, Font::plain);

// Visage: Fonts must be embedded as binary data
#include "fonts/roboto_regular.h"
auto font = std::make_unique<visage::Font>(14.0f, 
    roboto_regular_data, 
    roboto_regular_size);
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

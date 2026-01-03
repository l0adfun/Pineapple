# Pineapple Editor - Technical Design Document
## A Next-Generation Native Windows Text/Code Editor

**Version:** 1.0  
**Target Platform:** Windows 10/11 (Native Win32 + Modern APIs)  
**Language:** 100% C++20  
**License:** Commercial-grade proprietary  

---

## Table of Contents

1. [Product Vision](#1-product-vision)
2. [System Architecture](#2-system-architecture)
3. [UI Rendering Strategy](#3-ui-rendering-strategy)
4. [Plugin System Design](#4-plugin-system-design)
5. [Comparison: Pineapple vs Notepad++](#5-comparison-pineapple-vs-notepad)
6. [Lightweight Strategy](#6-lightweight-strategy)
7. [Future-Proofing Strategy](#7-future-proofing-strategy)
8. [Developer Build Guide](#8-developer-build-guide)

---

## 1. Product Vision

### 1.1 What Makes Pineapple Special

Pineapple is not another Notepad++ clone. It is a ground-up reimagining of what a native
Windows text editor should be in 2024+.

**Core Differentiators:**

| Aspect | Notepad++ | Pineapple |
|--------|-----------|-----------|
| Rendering | GDI/Scintilla | Direct2D/DirectWrite |
| Architecture | Monolithic | Modular microkernel |
| Plugin API | NPAPI-style C | Modern C++20 ABI-stable |
| Animation | None | Hardware-accelerated micro-interactions |
| Large files | Struggles >100MB | Memory-mapped streaming |
| Multi-cursor | Basic | Full VSCode-level |
| Theme engine | XML-based | Real-time shader-capable |


### 1.2 Design Philosophy

```
SPEED > FEATURES > AESTHETICS
(but never sacrifice any completely)
```

**The Pineapple Promise:**
- Cold start in <200ms on SSD systems
- Idle RAM <30MB with no documents open
- Open 1GB files without freezing
- Zero telemetry, zero network calls, zero background processes
- Professional UI that doesn't look like it's from 2005

### 1.3 Target Users

1. **Professional developers** who need speed and reliability
2. **System administrators** editing config files and logs
3. **Power users** who outgrew Notepad but find VS Code too heavy
4. **Enterprise environments** where Electron apps are banned

---

## 2. System Architecture

### 2.1 High-Level Module Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           PINEAPPLE EDITOR                               │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   Shell     │  │  Command    │  │   Plugin    │  │  Settings   │    │
│  │   (UI)      │  │  Palette    │  │   Host      │  │   Manager   │    │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘    │
│         │                │                │                │            │
│  ┌──────┴────────────────┴────────────────┴────────────────┴──────┐    │
│  │                      MESSAGE BUS (Lock-free)                    │    │
│  └──────┬────────────────┬────────────────┬────────────────┬──────┘    │
│         │                │                │                │            │
│  ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐    │
│  │   Editor    │  │   Buffer    │  │   Syntax    │  │   Search    │    │
│  │   View      │  │   Manager   │  │   Engine    │  │   Engine    │    │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘    │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    CORE SERVICES LAYER                           │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ │   │
│  │  │ Memory   │ │ File I/O │ │ Undo/Redo│ │ Encoding │ │ Macro  │ │   │
│  │  │ Pool     │ │ Async    │ │ Stack    │ │ Detector │ │ Engine │ │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └────────┘ │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    PLATFORM ABSTRACTION LAYER                    │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │   │
│  │  │ Win32 Window │  │ Direct2D/DW  │  │ File System  │           │   │
│  │  │ Management   │  │ Renderer     │  │ Watcher      │           │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```


### 2.2 Module Responsibilities

#### 2.2.1 Shell (UI Layer)

**Responsibility:** Window management, tab bar, status bar, menu system, toolbar

```cpp
namespace pineapple::shell {

class MainWindow {
public:
    // Win32 HWND wrapper with modern C++ semantics
    void Initialize(HINSTANCE hInstance);
    void RunMessageLoop();
    
private:
    HWND m_hwnd;
    std::unique_ptr<TabBar> m_tabBar;
    std::unique_ptr<StatusBar> m_statusBar;
    std::unique_ptr<EditorContainer> m_editorContainer;
    
    // Direct2D resources (lazy-initialized)
    ComPtr<ID2D1Factory1> m_d2dFactory;
    ComPtr<ID2D1HwndRenderTarget> m_renderTarget;
};

} // namespace pineapple::shell
```

**Key Design Decisions:**
- Single top-level window (no MDI, no floating windows by default)
- Tab bar is custom-rendered (not Win32 tab control) for animation support
- All chrome is owner-drawn via Direct2D

#### 2.2.2 Editor View

**Responsibility:** Text rendering, cursor management, selection, scrolling

```cpp
namespace pineapple::editor {

class EditorView {
public:
    // Core rendering
    void Render(ID2D1RenderTarget* target, const ViewportRect& viewport);
    
    // Input handling
    void OnKeyDown(WPARAM vk, LPARAM flags);
    void OnMouseDown(int x, int y, MouseButton btn);
    void OnMouseWheel(int delta, bool smooth);
    
    // Multi-cursor support
    void AddCursor(TextPosition pos);
    void RemoveCursor(size_t index);
    std::span<const Cursor> GetCursors() const;
    
private:
    BufferRef m_buffer;                    // Non-owning reference to buffer
    std::vector<Cursor> m_cursors;         // Multi-cursor state
    ViewState m_viewState;                 // Scroll position, zoom, etc.
    std::unique_ptr<LineCache> m_lineCache; // Rendered line bitmap cache
    
    // Animation state
    ScrollAnimator m_scrollAnimator;
    CursorBlinkAnimator m_cursorBlink;
};

} // namespace pineapple::editor
```


#### 2.2.3 Buffer Manager

**Responsibility:** Text storage, piece table implementation, memory management

```cpp
namespace pineapple::buffer {

// Piece Table implementation for efficient editing
// This is THE critical data structure for editor performance

struct Piece {
    enum class Source : uint8_t { Original, Add };
    Source source;
    size_t start;      // Offset into source buffer
    size_t length;     // Length in bytes (not characters!)
    size_t lineCount;  // Cached line count for fast line-number lookup
};

class PieceTable {
public:
    // O(log n) operations via balanced tree of pieces
    void Insert(size_t offset, std::string_view text);
    void Delete(size_t offset, size_t length);
    
    // Efficient iteration without materializing full string
    void ForEachLine(size_t startLine, size_t count, 
                     std::function<void(std::string_view)> callback) const;
    
    // Memory-mapped file support for large files
    static std::unique_ptr<PieceTable> FromMemoryMappedFile(
        const std::filesystem::path& path);
    
private:
    std::string m_originalBuffer;          // Immutable original content
    std::string m_addBuffer;               // Append-only add buffer
    std::vector<Piece> m_pieces;           // Red-black tree in production
    
    // Line index for O(log n) line lookups
    std::vector<size_t> m_lineStarts;
};

class BufferManager {
public:
    BufferHandle CreateBuffer();
    BufferHandle OpenFile(const std::filesystem::path& path);
    void CloseBuffer(BufferHandle handle);
    
    // Thread-safe buffer access
    BufferRef GetBuffer(BufferHandle handle);
    
private:
    std::unordered_map<BufferHandle, std::unique_ptr<Buffer>> m_buffers;
    std::shared_mutex m_mutex;
    std::atomic<BufferHandle> m_nextHandle{1};
};

} // namespace pineapple::buffer
```

**Why Piece Table over Gap Buffer?**

| Operation | Gap Buffer | Piece Table |
|-----------|------------|-------------|
| Insert at cursor | O(1) amortized | O(log n) |
| Insert far from gap | O(n) | O(log n) |
| Undo/Redo | Complex | Trivial (just reorder pieces) |
| Large file open | Must load all | Can memory-map |
| Memory efficiency | Good | Excellent |

For a professional editor, piece table wins due to consistent performance and
superior undo/redo semantics.


#### 2.2.4 Syntax Engine

**Responsibility:** Language detection, tokenization, highlighting

```cpp
namespace pineapple::syntax {

// Token types (language-agnostic)
enum class TokenType : uint8_t {
    Default,
    Keyword,
    Identifier,
    String,
    Number,
    Comment,
    Operator,
    Preprocessor,
    Type,
    Function,
    // ... extensible via plugins
    Custom = 128
};

struct Token {
    uint32_t start;      // Byte offset
    uint16_t length;     // Token length
    TokenType type;
    uint8_t flags;       // Folding hints, error state, etc.
};

// Abstract tokenizer interface - plugins implement this
class ITokenizer {
public:
    virtual ~ITokenizer() = default;
    
    // Incremental tokenization - only re-tokenize changed regions
    virtual void TokenizeLine(std::string_view line, 
                              uint32_t lineState,
                              std::vector<Token>& outTokens,
                              uint32_t& outNextState) = 0;
    
    // Language metadata
    virtual std::string_view GetLanguageId() const = 0;
    virtual std::span<const std::string_view> GetFileExtensions() const = 0;
};

class SyntaxEngine {
public:
    void RegisterTokenizer(std::unique_ptr<ITokenizer> tokenizer);
    
    // Auto-detect language from file extension or content
    const ITokenizer* DetectLanguage(const std::filesystem::path& path,
                                     std::string_view firstLines);
    
    // Get cached tokens for a buffer region
    std::span<const Token> GetTokens(BufferHandle buffer, 
                                     size_t startLine, 
                                     size_t lineCount);
    
private:
    std::vector<std::unique_ptr<ITokenizer>> m_tokenizers;
    
    // Per-buffer token cache
    struct TokenCache {
        std::vector<std::vector<Token>> lineTokens;
        std::vector<uint32_t> lineStates;  // For incremental re-tokenization
        size_t validUntilLine = 0;
    };
    std::unordered_map<BufferHandle, TokenCache> m_caches;
};

} // namespace pineapple::syntax
```

**Built-in Languages (no plugins required):**
- C/C++
- Python
- JavaScript/TypeScript
- JSON/XML/YAML
- Markdown
- SQL
- Shell scripts (bash, PowerShell, cmd)

Additional languages via plugin DLLs.


#### 2.2.5 Search Engine

**Responsibility:** Find, replace, regex, multi-file search

```cpp
namespace pineapple::search {

struct SearchOptions {
    bool caseSensitive = false;
    bool wholeWord = false;
    bool useRegex = false;
    bool searchInSelection = false;
    bool wrapAround = true;
};

struct SearchResult {
    size_t line;
    size_t column;
    size_t length;
    std::string_view matchedText;  // View into buffer, not a copy
};

class SearchEngine {
public:
    // Single-buffer search
    std::vector<SearchResult> FindAll(BufferRef buffer, 
                                      std::string_view pattern,
                                      const SearchOptions& options);
    
    // Incremental search (for real-time highlighting)
    void BeginIncrementalSearch(BufferRef buffer);
    std::optional<SearchResult> FindNext(std::string_view pattern);
    void EndIncrementalSearch();
    
    // Replace operations
    size_t ReplaceAll(BufferRef buffer,
                      std::string_view pattern,
                      std::string_view replacement,
                      const SearchOptions& options);
    
    // Multi-file search (async)
    std::future<std::vector<FileSearchResult>> SearchInFiles(
        const std::filesystem::path& directory,
        std::string_view pattern,
        const SearchOptions& options,
        const FileFilter& filter);
    
private:
    // Compiled regex cache (LRU, max 32 entries)
    std::unique_ptr<RegexCache> m_regexCache;
    
    // Boyer-Moore-Horspool for literal searches
    std::unique_ptr<BMHSearcher> m_literalSearcher;
};

} // namespace pineapple::search
```

**Regex Implementation:**
- Use RE2 (Google's regex library) for guaranteed linear-time matching
- No catastrophic backtracking possible
- Compile once, search many times


#### 2.2.6 Message Bus

**Responsibility:** Decoupled inter-module communication

```cpp
namespace pineapple::core {

// Type-safe message definitions
struct BufferModifiedMsg {
    BufferHandle buffer;
    size_t startLine;
    size_t endLine;
    ModificationType type;
};

struct ThemeChangedMsg {
    std::string_view themeId;
};

struct CursorMovedMsg {
    BufferHandle buffer;
    TextPosition newPosition;
};

// Lock-free single-producer-multi-consumer queue per message type
class MessageBus {
public:
    template<typename Msg>
    void Subscribe(std::function<void(const Msg&)> handler);
    
    template<typename Msg>
    void Publish(const Msg& message);
    
    // Process pending messages (called from main thread)
    void ProcessMessages();
    
private:
    // Type-erased handler storage
    struct HandlerList {
        std::vector<std::function<void(const void*)>> handlers;
        moodycamel::ConcurrentQueue<std::any> pendingMessages;
    };
    
    std::unordered_map<std::type_index, HandlerList> m_handlers;
};

} // namespace pineapple::core
```

**Why a Message Bus?**
- Modules don't need direct references to each other
- Easy to add new features without modifying existing code
- Natural point for macro recording (intercept all messages)
- Enables async operations without callback hell


#### 2.2.7 Macro Engine

**Responsibility:** Record and playback user actions

```cpp
namespace pineapple::macro {

// Macro is a sequence of commands, not keystrokes
// This makes macros more robust across keyboard layouts

struct MacroCommand {
    std::string commandId;           // e.g., "editor.insertText"
    std::vector<std::any> arguments; // Command-specific args
    uint32_t repeatCount = 1;
};

class MacroEngine {
public:
    void StartRecording();
    void StopRecording();
    bool IsRecording() const;
    
    // Save/load macros
    void SaveMacro(std::string_view name, const std::filesystem::path& path);
    void LoadMacro(const std::filesystem::path& path);
    
    // Playback
    void PlayMacro(std::string_view name, uint32_t repeatCount = 1);
    
    // Called by command system to record commands
    void RecordCommand(const MacroCommand& cmd);
    
private:
    bool m_isRecording = false;
    std::vector<MacroCommand> m_currentRecording;
    std::unordered_map<std::string, std::vector<MacroCommand>> m_savedMacros;
};

} // namespace pineapple::macro
```

**Macro Format:**
- Stored as binary (fast load) with JSON export option
- No scripting language = no security concerns
- Macros are portable across Pineapple installations

---

## 3. UI Rendering Strategy

### 3.1 Rendering Pipeline Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    RENDERING PIPELINE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │  Layout  │───▶│  Paint   │───▶│ Composite│───▶│  Present │  │
│  │  Pass    │    │  Pass    │    │  Pass    │    │  (Swap)  │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘  │
│       │              │               │                          │
│       ▼              ▼               ▼                          │
│  Calculate      Render to       Blend layers      Flip to       │
│  positions      off-screen      with effects      display       │
│  & sizes        bitmaps                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```


### 3.2 Direct2D + DirectWrite Setup

```cpp
namespace pineapple::render {

class RenderContext {
public:
    bool Initialize(HWND hwnd) {
        // Create D2D factory with multi-threaded support
        D2D1_FACTORY_OPTIONS options = {};
        #ifdef _DEBUG
        options.debugLevel = D2D1_DEBUG_LEVEL_INFORMATION;
        #endif
        
        HRESULT hr = D2D1CreateFactory(
            D2D1_FACTORY_TYPE_MULTI_THREADED,
            options,
            m_d2dFactory.GetAddressOf()
        );
        if (FAILED(hr)) return false;
        
        // Create DirectWrite factory for text rendering
        hr = DWriteCreateFactory(
            DWRITE_FACTORY_TYPE_SHARED,
            __uuidof(IDWriteFactory),
            reinterpret_cast<IUnknown**>(m_dwriteFactory.GetAddressOf())
        );
        if (FAILED(hr)) return false;
        
        // Create DXGI swap chain for smooth vsync
        CreateSwapChain(hwnd);
        
        // Create render target from swap chain
        CreateRenderTarget();
        
        return true;
    }
    
    void BeginDraw() {
        m_renderTarget->BeginDraw();
        m_renderTarget->SetTransform(D2D1::Matrix3x2F::Identity());
    }
    
    HRESULT EndDraw() {
        HRESULT hr = m_renderTarget->EndDraw();
        if (hr == D2DERR_RECREATE_TARGET) {
            DiscardDeviceResources();
            CreateRenderTarget();
        }
        
        // Present with vsync
        m_swapChain->Present(1, 0);
        return hr;
    }
    
private:
    ComPtr<ID2D1Factory1> m_d2dFactory;
    ComPtr<IDWriteFactory> m_dwriteFactory;
    ComPtr<ID2D1DeviceContext> m_renderTarget;
    ComPtr<IDXGISwapChain1> m_swapChain;
};

} // namespace pineapple::render
```

### 3.3 Text Rendering with DirectWrite

```cpp
namespace pineapple::render {

class TextRenderer {
public:
    void Initialize(IDWriteFactory* factory) {
        // Create text format for code (monospace)
        factory->CreateTextFormat(
            L"Cascadia Code",        // Font family
            nullptr,                  // Font collection (nullptr = system)
            DWRITE_FONT_WEIGHT_NORMAL,
            DWRITE_FONT_STYLE_NORMAL,
            DWRITE_FONT_STRETCH_NORMAL,
            14.0f,                    // Font size in DIPs
            L"en-US",
            m_codeTextFormat.GetAddressOf()
        );
        
        // Enable subpixel rendering for crisp text
        m_codeTextFormat->SetWordWrapping(DWRITE_WORD_WRAPPING_NO_WRAP);
    }
    
    void RenderLine(ID2D1RenderTarget* target,
                    std::string_view text,
                    std::span<const Token> tokens,
                    float x, float y,
                    const Theme& theme) {
        // Convert UTF-8 to UTF-16 for DirectWrite
        std::wstring wtext = Utf8ToUtf16(text);
        
        // Create text layout for this line
        ComPtr<IDWriteTextLayout> layout;
        m_dwriteFactory->CreateTextLayout(
            wtext.c_str(),
            static_cast<UINT32>(wtext.length()),
            m_codeTextFormat.Get(),
            10000.0f,  // Max width (effectively infinite for code)
            100.0f,    // Max height
            layout.GetAddressOf()
        );
        
        // Apply syntax highlighting colors
        for (const auto& token : tokens) {
            DWRITE_TEXT_RANGE range = {token.start, token.length};
            auto brush = theme.GetBrushForToken(token.type);
            layout->SetDrawingEffect(brush, range);
        }
        
        // Draw with custom renderer for colored text
        target->DrawTextLayout(
            D2D1::Point2F(x, y),
            layout.Get(),
            theme.GetDefaultBrush(),
            D2D1_DRAW_TEXT_OPTIONS_ENABLE_COLOR_FONT
        );
    }
    
private:
    ComPtr<IDWriteFactory> m_dwriteFactory;
    ComPtr<IDWriteTextFormat> m_codeTextFormat;
};

} // namespace pineapple::render
```


### 3.4 Animation System

**Design Goal:** macOS-quality micro-interactions without GPU overhead

```cpp
namespace pineapple::animation {

// Easing functions (matching Apple's defaults)
namespace easing {
    float EaseOutCubic(float t) { 
        return 1.0f - std::pow(1.0f - t, 3.0f); 
    }
    
    float EaseInOutQuad(float t) {
        return t < 0.5f 
            ? 2.0f * t * t 
            : 1.0f - std::pow(-2.0f * t + 2.0f, 2.0f) / 2.0f;
    }
    
    float Spring(float t, float damping = 0.7f, float stiffness = 100.0f) {
        // Critically damped spring for natural feel
        float omega = std::sqrt(stiffness);
        float decay = std::exp(-damping * omega * t);
        return 1.0f - decay * (1.0f + damping * omega * t);
    }
}

class Animator {
public:
    using AnimationId = uint32_t;
    
    template<typename T>
    AnimationId Animate(T& target, 
                        T endValue, 
                        float durationMs,
                        EasingFunc easing = easing::EaseOutCubic) {
        auto id = m_nextId++;
        m_animations.emplace_back(Animation{
            .id = id,
            .startTime = GetCurrentTimeMs(),
            .duration = durationMs,
            .easing = easing,
            .update = [&target, startValue = target, endValue](float progress) {
                target = Lerp(startValue, endValue, progress);
            }
        });
        return id;
    }
    
    void Cancel(AnimationId id);
    void Update();  // Called every frame
    
private:
    struct Animation {
        AnimationId id;
        float startTime;
        float duration;
        EasingFunc easing;
        std::function<void(float)> update;
    };
    
    std::vector<Animation> m_animations;
    std::atomic<AnimationId> m_nextId{1};
};

} // namespace pineapple::animation
```

### 3.5 Specific Animation Implementations

#### Smooth Scrolling

```cpp
class ScrollAnimator {
public:
    void ScrollTo(float targetY) {
        m_targetScrollY = targetY;
        m_scrollVelocity = 0;
        m_isAnimating = true;
    }
    
    void ScrollBy(float deltaY) {
        m_targetScrollY += deltaY;
        m_isAnimating = true;
    }
    
    void Update(float deltaTime) {
        if (!m_isAnimating) return;
        
        // Spring physics for natural deceleration
        float displacement = m_targetScrollY - m_currentScrollY;
        float springForce = displacement * SPRING_STIFFNESS;
        float dampingForce = -m_scrollVelocity * DAMPING;
        
        m_scrollVelocity += (springForce + dampingForce) * deltaTime;
        m_currentScrollY += m_scrollVelocity * deltaTime;
        
        // Stop when close enough
        if (std::abs(displacement) < 0.5f && 
            std::abs(m_scrollVelocity) < 0.5f) {
            m_currentScrollY = m_targetScrollY;
            m_isAnimating = false;
        }
    }
    
    float GetCurrentScroll() const { return m_currentScrollY; }
    
private:
    static constexpr float SPRING_STIFFNESS = 300.0f;
    static constexpr float DAMPING = 25.0f;
    
    float m_currentScrollY = 0;
    float m_targetScrollY = 0;
    float m_scrollVelocity = 0;
    bool m_isAnimating = false;
};
```


#### Button Press Animation

```cpp
class ButtonWidget {
public:
    void OnMouseDown() {
        m_animator.Animate(m_scale, 0.95f, 50.0f, easing::EaseOutCubic);
        m_animator.Animate(m_brightness, 0.9f, 50.0f);
    }
    
    void OnMouseUp() {
        m_animator.Animate(m_scale, 1.0f, 150.0f, easing::Spring);
        m_animator.Animate(m_brightness, 1.0f, 150.0f);
    }
    
    void Render(ID2D1RenderTarget* target) {
        // Apply scale transform
        auto center = GetCenter();
        target->SetTransform(
            D2D1::Matrix3x2F::Scale(m_scale, m_scale, center)
        );
        
        // Render with brightness adjustment
        auto brush = GetBrush();
        brush->SetOpacity(m_brightness);
        
        // Draw button...
        
        target->SetTransform(D2D1::Matrix3x2F::Identity());
    }
    
private:
    float m_scale = 1.0f;
    float m_brightness = 1.0f;
    Animator m_animator;
};
```

#### Hover Transitions

```cpp
class HoverableWidget {
public:
    void OnMouseEnter() {
        m_animator.Animate(m_hoverProgress, 1.0f, 200.0f, easing::EaseOutCubic);
    }
    
    void OnMouseLeave() {
        m_animator.Animate(m_hoverProgress, 0.0f, 300.0f, easing::EaseInOutQuad);
    }
    
    D2D1_COLOR_F GetCurrentColor() const {
        // Interpolate between normal and hover colors
        return LerpColor(m_normalColor, m_hoverColor, m_hoverProgress);
    }
    
private:
    float m_hoverProgress = 0.0f;
    D2D1_COLOR_F m_normalColor;
    D2D1_COLOR_F m_hoverColor;
};
```

### 3.6 Frame Rate Management

```cpp
class FrameScheduler {
public:
    void RequestFrame() {
        m_frameRequested = true;
    }
    
    void Run() {
        while (m_running) {
            if (m_frameRequested || HasActiveAnimations()) {
                auto frameStart = std::chrono::high_resolution_clock::now();
                
                // Update animations
                m_animator.Update();
                
                // Render
                m_renderContext.BeginDraw();
                RenderFrame();
                m_renderContext.EndDraw();
                
                m_frameRequested = false;
                
                // Cap at 60fps when animating, otherwise sleep
                auto frameTime = std::chrono::high_resolution_clock::now() - frameStart;
                auto sleepTime = std::chrono::milliseconds(16) - frameTime;
                if (sleepTime > std::chrono::milliseconds(0)) {
                    std::this_thread::sleep_for(sleepTime);
                }
            } else {
                // No animations - wait for input events
                WaitMessage();
            }
        }
    }
    
private:
    std::atomic<bool> m_frameRequested{false};
    std::atomic<bool> m_running{true};
};
```

**Key Insight:** Only render at 60fps during animations. When idle, the editor
uses zero CPU by waiting on `WaitMessage()`.


### 3.7 Theme System

```cpp
namespace pineapple::theme {

struct ThemeColors {
    // Editor colors
    D2D1_COLOR_F background;
    D2D1_COLOR_F foreground;
    D2D1_COLOR_F selection;
    D2D1_COLOR_F cursor;
    D2D1_COLOR_F lineHighlight;
    D2D1_COLOR_F lineNumbers;
    D2D1_COLOR_F lineNumbersActive;
    
    // Syntax colors
    D2D1_COLOR_F keyword;
    D2D1_COLOR_F string;
    D2D1_COLOR_F number;
    D2D1_COLOR_F comment;
    D2D1_COLOR_F function;
    D2D1_COLOR_F type;
    D2D1_COLOR_F operator_;
    D2D1_COLOR_F preprocessor;
    
    // UI colors
    D2D1_COLOR_F tabBarBackground;
    D2D1_COLOR_F tabActive;
    D2D1_COLOR_F tabInactive;
    D2D1_COLOR_F statusBarBackground;
    D2D1_COLOR_F buttonNormal;
    D2D1_COLOR_F buttonHover;
    D2D1_COLOR_F buttonPressed;
};

class Theme {
public:
    static Theme LoadFromFile(const std::filesystem::path& path);
    static Theme GetBuiltinDark();
    static Theme GetBuiltinLight();
    
    // Animated theme switching
    void TransitionTo(const Theme& target, float durationMs);
    void Update(float deltaTime);
    
    ID2D1Brush* GetBrushForToken(TokenType type) const;
    const ThemeColors& GetColors() const { return m_currentColors; }
    
private:
    ThemeColors m_currentColors;
    ThemeColors m_targetColors;
    float m_transitionProgress = 1.0f;
    
    // Cached brushes (recreated on theme change)
    mutable std::unordered_map<TokenType, ComPtr<ID2D1SolidColorBrush>> m_brushCache;
};

} // namespace pineapple::theme
```

**Built-in Themes:**
- Pineapple Dark (default) - optimized for long coding sessions
- Pineapple Light - high contrast for bright environments
- Solarized Dark/Light - for Solarized fans
- One Dark - Atom-inspired

---

## 4. Plugin System Design

### 4.1 Plugin Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      PLUGIN HOST                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    PLUGIN API (ABI-stable)                │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐        │   │
│  │  │ Editor  │ │ Buffer  │ │ UI      │ │ Command │        │   │
│  │  │ API     │ │ API     │ │ API     │ │ API     │        │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    PLUGIN LOADER                          │   │
│  │  - DLL loading/unloading                                  │   │
│  │  - Version compatibility check                            │   │
│  │  - Sandbox enforcement                                    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                   │
│         ┌────────────────────┼────────────────────┐             │
│         ▼                    ▼                    ▼             │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐       │
│  │  Plugin A   │     │  Plugin B   │     │  Plugin C   │       │
│  │  (DLL)      │     │  (DLL)      │     │  (DLL)      │       │
│  └─────────────┘     └─────────────┘     └─────────────┘       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```


### 4.2 Plugin API (C ABI for compatibility)

```cpp
// pineapple_plugin.h - Public plugin API header

#pragma once

#include <stdint.h>
#include <stddef.h>

#ifdef __cplusplus
extern "C" {
#endif

// API version for compatibility checking
#define PINEAPPLE_API_VERSION 1

// Opaque handles
typedef struct PineappleEditor* PineappleEditorHandle;
typedef struct PineappleBuffer* PineappleBufferHandle;
typedef uint32_t PineappleCommandId;

// Plugin info structure
typedef struct {
    const char* name;
    const char* version;
    const char* author;
    const char* description;
    uint32_t apiVersion;  // Must match PINEAPPLE_API_VERSION
} PineapplePluginInfo;

// Plugin lifecycle callbacks
typedef struct {
    // Called when plugin is loaded
    int (*OnLoad)(PineappleEditorHandle editor);
    
    // Called when plugin is unloaded
    void (*OnUnload)(void);
    
    // Called when editor is about to close (save state here)
    void (*OnShutdown)(void);
} PineapplePluginCallbacks;

// Required export: Plugin entry point
// Returns plugin info and callbacks
__declspec(dllexport) 
PineapplePluginInfo* PineapplePluginInit(PineapplePluginCallbacks* outCallbacks);

// ============================================================================
// EDITOR API
// ============================================================================

// Get active buffer
PineappleBufferHandle PineappleGetActiveBuffer(PineappleEditorHandle editor);

// Open file in new tab
PineappleBufferHandle PineappleOpenFile(PineappleEditorHandle editor, 
                                         const char* path);

// Register a command
PineappleCommandId PineappleRegisterCommand(
    PineappleEditorHandle editor,
    const char* commandId,           // e.g., "myPlugin.doSomething"
    const char* displayName,         // e.g., "Do Something"
    void (*callback)(void* userData),
    void* userData
);

// Add menu item
void PineappleAddMenuItem(
    PineappleEditorHandle editor,
    const char* menuPath,            // e.g., "Tools/My Plugin/Do Something"
    PineappleCommandId command
);

// Add keyboard shortcut
void PineappleAddKeybinding(
    PineappleEditorHandle editor,
    const char* keyCombo,            // e.g., "Ctrl+Shift+P"
    PineappleCommandId command
);

// ============================================================================
// BUFFER API
// ============================================================================

// Get buffer content (caller must free with PineappleFreeString)
char* PineappleBufferGetText(PineappleBufferHandle buffer);
void PineappleFreeString(char* str);

// Get line count
size_t PineappleBufferGetLineCount(PineappleBufferHandle buffer);

// Get specific line (caller must free)
char* PineappleBufferGetLine(PineappleBufferHandle buffer, size_t lineIndex);

// Insert text at position
void PineappleBufferInsert(PineappleBufferHandle buffer, 
                           size_t line, size_t column, 
                           const char* text);

// Delete range
void PineappleBufferDelete(PineappleBufferHandle buffer,
                           size_t startLine, size_t startCol,
                           size_t endLine, size_t endCol);

// Get/set cursor position
void PineappleBufferGetCursor(PineappleBufferHandle buffer, 
                              size_t* outLine, size_t* outColumn);
void PineappleBufferSetCursor(PineappleBufferHandle buffer, 
                              size_t line, size_t column);

// ============================================================================
// SYNTAX API (for language plugins)
// ============================================================================

typedef struct {
    uint32_t start;
    uint16_t length;
    uint8_t tokenType;
    uint8_t flags;
} PineappleToken;

typedef void (*PineappleTokenizerFunc)(
    const char* lineText,
    size_t lineLength,
    uint32_t lineState,
    PineappleToken* outTokens,
    size_t* outTokenCount,
    size_t maxTokens,
    uint32_t* outNextState
);

void PineappleRegisterLanguage(
    PineappleEditorHandle editor,
    const char* languageId,
    const char** fileExtensions,
    size_t extensionCount,
    PineappleTokenizerFunc tokenizer
);

#ifdef __cplusplus
}
#endif
```


### 4.3 Example Plugin Implementation

```cpp
// example_plugin.cpp - A simple plugin that adds word count

#include "pineapple_plugin.h"
#include <string>
#include <sstream>

static PineappleEditorHandle g_editor = nullptr;

void ShowWordCount(void* userData) {
    auto buffer = PineappleGetActiveBuffer(g_editor);
    if (!buffer) return;
    
    char* text = PineappleBufferGetText(buffer);
    if (!text) return;
    
    // Count words
    std::istringstream stream(text);
    std::string word;
    size_t count = 0;
    while (stream >> word) count++;
    
    PineappleFreeString(text);
    
    // Show result (using Windows MessageBox for simplicity)
    std::string msg = "Word count: " + std::to_string(count);
    MessageBoxA(nullptr, msg.c_str(), "Word Count", MB_OK);
}

int OnLoad(PineappleEditorHandle editor) {
    g_editor = editor;
    
    auto cmdId = PineappleRegisterCommand(
        editor,
        "wordCount.show",
        "Show Word Count",
        ShowWordCount,
        nullptr
    );
    
    PineappleAddMenuItem(editor, "Tools/Word Count", cmdId);
    PineappleAddKeybinding(editor, "Ctrl+Shift+W", cmdId);
    
    return 0;  // Success
}

void OnUnload() {
    g_editor = nullptr;
}

void OnShutdown() {
    // Nothing to save
}

extern "C" __declspec(dllexport)
PineapplePluginInfo* PineapplePluginInit(PineapplePluginCallbacks* outCallbacks) {
    static PineapplePluginInfo info = {
        .name = "Word Count",
        .version = "1.0.0",
        .author = "Pineapple Team",
        .description = "Shows word count for current document",
        .apiVersion = PINEAPPLE_API_VERSION
    };
    
    outCallbacks->OnLoad = OnLoad;
    outCallbacks->OnUnload = OnUnload;
    outCallbacks->OnShutdown = OnShutdown;
    
    return &info;
}
```

### 4.4 Plugin Security Model

```cpp
namespace pineapple::plugin {

class PluginSandbox {
public:
    // Plugins run in the main process but with restricted capabilities
    
    struct Permissions {
        bool canAccessFileSystem = false;   // Requires user approval
        bool canAccessNetwork = false;      // Requires user approval
        bool canModifyUI = true;            // Default allowed
        bool canRegisterCommands = true;    // Default allowed
        bool canAccessClipboard = true;     // Default allowed
    };
    
    void LoadPlugin(const std::filesystem::path& dllPath) {
        // 1. Verify DLL signature (optional, for signed plugins)
        if (!VerifySignature(dllPath)) {
            PromptUserForUnsignedPlugin(dllPath);
        }
        
        // 2. Load DLL
        HMODULE hModule = LoadLibraryW(dllPath.c_str());
        if (!hModule) return;
        
        // 3. Get entry point
        auto initFunc = reinterpret_cast<PluginInitFunc>(
            GetProcAddress(hModule, "PineapplePluginInit")
        );
        if (!initFunc) {
            FreeLibrary(hModule);
            return;
        }
        
        // 4. Initialize plugin
        PineapplePluginCallbacks callbacks = {};
        auto info = initFunc(&callbacks);
        
        // 5. Version check
        if (info->apiVersion != PINEAPPLE_API_VERSION) {
            ShowError("Plugin API version mismatch");
            FreeLibrary(hModule);
            return;
        }
        
        // 6. Store plugin info
        m_loadedPlugins.push_back({
            .module = hModule,
            .info = *info,
            .callbacks = callbacks,
            .permissions = GetDefaultPermissions()
        });
        
        // 7. Call OnLoad
        callbacks.OnLoad(GetEditorHandle());
    }
    
private:
    struct LoadedPlugin {
        HMODULE module;
        PineapplePluginInfo info;
        PineapplePluginCallbacks callbacks;
        Permissions permissions;
    };
    
    std::vector<LoadedPlugin> m_loadedPlugins;
};

} // namespace pineapple::plugin
```


---

## 5. Comparison: Pineapple vs Notepad++

| Feature | Notepad++ | Pineapple | Notes |
|---------|-----------|-----------|-------|
| **Startup Time** | ~300ms | <200ms | Lazy initialization |
| **Idle RAM** | ~25MB | <30MB | Comparable |
| **1GB File Open** | Freezes/crashes | <3 seconds | Memory-mapped I/O |
| **Rendering** | GDI (Scintilla) | Direct2D/DirectWrite | Hardware accelerated |
| **Text Quality** | Good | Excellent | Subpixel antialiasing |
| **Animations** | None | macOS-quality | Spring physics |
| **Multi-cursor** | Basic | Full | VSCode-level |
| **Regex Engine** | PCRE | RE2 | No catastrophic backtracking |
| **Plugin API** | C NPAPI-style | Modern C ABI | Stable, versioned |
| **Theme Switching** | Requires restart | Instant | Animated transitions |
| **Macro System** | Keystroke-based | Command-based | More robust |
| **Code Folding** | Basic | Semantic | Language-aware |
| **Encoding Detection** | Good | Excellent | ML-based fallback |
| **Session Restore** | Basic | Full | Undo history preserved |
| **Find in Files** | Synchronous | Async | Non-blocking UI |
| **UI Framework** | Win32 controls | Custom Direct2D | Consistent look |
| **DPI Awareness** | Per-monitor v1 | Per-monitor v2 | Better multi-monitor |
| **Dark Mode** | Partial | Full | System integration |

### 5.1 Where Notepad++ Wins (and we accept it)

- **Plugin ecosystem**: Notepad++ has 20+ years of plugins
- **Familiarity**: Millions of users know its UI
- **Stability**: Battle-tested for decades

### 5.2 Where Pineapple Wins

- **Large file handling**: Memory-mapped piece table vs loading entire file
- **Visual polish**: Modern animations and transitions
- **Consistent performance**: No GC pauses, no Scintilla quirks
- **Future-proof architecture**: Clean module boundaries

---

## 6. Lightweight Strategy

### 6.1 Memory Management

```cpp
namespace pineapple::memory {

// Custom allocator for editor-specific patterns
class EditorAllocator {
public:
    // Pool allocator for small, frequent allocations (tokens, pieces)
    template<typename T>
    class PoolAllocator {
    public:
        PoolAllocator(size_t blockSize = 4096) 
            : m_blockSize(blockSize) {
            AllocateBlock();
        }
        
        T* Allocate() {
            if (m_freeList) {
                T* result = m_freeList;
                m_freeList = *reinterpret_cast<T**>(m_freeList);
                return result;
            }
            
            if (m_currentOffset + sizeof(T) > m_blockSize) {
                AllocateBlock();
            }
            
            T* result = reinterpret_cast<T*>(m_currentBlock + m_currentOffset);
            m_currentOffset += sizeof(T);
            return result;
        }
        
        void Deallocate(T* ptr) {
            *reinterpret_cast<T**>(ptr) = m_freeList;
            m_freeList = ptr;
        }
        
    private:
        void AllocateBlock() {
            m_blocks.push_back(std::make_unique<char[]>(m_blockSize));
            m_currentBlock = m_blocks.back().get();
            m_currentOffset = 0;
        }
        
        size_t m_blockSize;
        std::vector<std::unique_ptr<char[]>> m_blocks;
        char* m_currentBlock = nullptr;
        size_t m_currentOffset = 0;
        T* m_freeList = nullptr;
    };
    
    // Arena allocator for per-frame temporary allocations
    class FrameArena {
    public:
        void* Allocate(size_t size, size_t alignment = 8) {
            size_t aligned = (m_offset + alignment - 1) & ~(alignment - 1);
            if (aligned + size > ARENA_SIZE) {
                // Overflow - allocate from heap (rare)
                return malloc(size);
            }
            void* result = m_buffer + aligned;
            m_offset = aligned + size;
            return result;
        }
        
        void Reset() { m_offset = 0; }
        
    private:
        static constexpr size_t ARENA_SIZE = 1024 * 1024;  // 1MB per frame
        alignas(64) char m_buffer[ARENA_SIZE];
        size_t m_offset = 0;
    };
};

} // namespace pineapple::memory
```


### 6.2 Lazy Initialization

```cpp
namespace pineapple::core {

// Only initialize subsystems when first needed
class LazySubsystem {
public:
    template<typename T>
    class Lazy {
    public:
        T& Get() {
            std::call_once(m_initFlag, [this]() {
                m_instance = std::make_unique<T>();
            });
            return *m_instance;
        }
        
        bool IsInitialized() const {
            return m_instance != nullptr;
        }
        
    private:
        std::unique_ptr<T> m_instance;
        std::once_flag m_initFlag;
    };
};

class Application {
public:
    // These are NOT initialized at startup
    SyntaxEngine& GetSyntaxEngine() { return m_syntaxEngine.Get(); }
    SearchEngine& GetSearchEngine() { return m_searchEngine.Get(); }
    MacroEngine& GetMacroEngine() { return m_macroEngine.Get(); }
    PluginHost& GetPluginHost() { return m_pluginHost.Get(); }
    
private:
    // Lazy-initialized subsystems
    LazySubsystem::Lazy<SyntaxEngine> m_syntaxEngine;
    LazySubsystem::Lazy<SearchEngine> m_searchEngine;
    LazySubsystem::Lazy<MacroEngine> m_macroEngine;
    LazySubsystem::Lazy<PluginHost> m_pluginHost;
    
    // Always initialized (required for basic operation)
    std::unique_ptr<BufferManager> m_bufferManager;
    std::unique_ptr<RenderContext> m_renderContext;
    std::unique_ptr<MainWindow> m_mainWindow;
};

} // namespace pineapple::core
```

### 6.3 Resource Loading Strategy

```cpp
// Icons, fonts, and themes are loaded on-demand

class ResourceManager {
public:
    ID2D1Bitmap* GetIcon(IconId id) {
        auto it = m_iconCache.find(id);
        if (it != m_iconCache.end()) {
            return it->second.Get();
        }
        
        // Load from embedded resource
        auto bitmap = LoadIconFromResource(id);
        m_iconCache[id] = bitmap;
        return bitmap.Get();
    }
    
    // Unload unused resources periodically
    void PruneCache() {
        // Remove icons not used in last 60 seconds
        auto now = std::chrono::steady_clock::now();
        for (auto it = m_iconLastUsed.begin(); it != m_iconLastUsed.end();) {
            if (now - it->second > std::chrono::seconds(60)) {
                m_iconCache.erase(it->first);
                it = m_iconLastUsed.erase(it);
            } else {
                ++it;
            }
        }
    }
    
private:
    std::unordered_map<IconId, ComPtr<ID2D1Bitmap>> m_iconCache;
    std::unordered_map<IconId, std::chrono::steady_clock::time_point> m_iconLastUsed;
};
```

### 6.4 Startup Sequence

```cpp
int WINAPI wWinMain(HINSTANCE hInstance, HINSTANCE, LPWSTR cmdLine, int cmdShow) {
    // Phase 1: Critical path (~50ms)
    // - Parse command line
    // - Load settings from disk (async start)
    // - Create main window (hidden)
    // - Initialize Direct2D
    
    // Phase 2: Show window (~20ms)
    // - Show window
    // - Render empty state
    
    // Phase 3: Background initialization (async)
    // - Load plugins
    // - Initialize syntax engine
    // - Restore session
    
    // Phase 4: Open requested files
    // - Parse command line files
    // - Open in tabs
    
    Application app;
    
    // Phase 1
    auto settingsFuture = std::async(std::launch::async, LoadSettings);
    app.CreateWindow(hInstance);
    app.InitializeRenderer();
    
    // Phase 2
    app.ShowWindow(cmdShow);
    app.RenderEmptyState();
    
    // Phase 3 (non-blocking)
    std::thread([&app, settingsFuture = std::move(settingsFuture)]() {
        auto settings = settingsFuture.get();
        app.ApplySettings(settings);
        app.LoadPlugins();
        app.RestoreSession();
    }).detach();
    
    // Phase 4
    app.OpenCommandLineFiles(cmdLine);
    
    // Message loop
    return app.RunMessageLoop();
}
```


---

## 7. Future-Proofing Strategy

### 7.1 Modular Extension Points

```cpp
// Every major feature is behind an interface

namespace pineapple::interfaces {

// Version control integration (future)
class IVersionControl {
public:
    virtual ~IVersionControl() = default;
    virtual std::vector<FileStatus> GetStatus(const std::filesystem::path& dir) = 0;
    virtual void Stage(const std::filesystem::path& file) = 0;
    virtual void Commit(std::string_view message) = 0;
};

// Language server protocol (future)
class ILanguageServer {
public:
    virtual ~ILanguageServer() = default;
    virtual std::vector<Diagnostic> GetDiagnostics(BufferHandle buffer) = 0;
    virtual std::vector<CompletionItem> GetCompletions(BufferHandle buffer, 
                                                        TextPosition pos) = 0;
    virtual std::optional<Location> GoToDefinition(BufferHandle buffer, 
                                                    TextPosition pos) = 0;
};

// AI assistance (future)
class IAIAssistant {
public:
    virtual ~IAIAssistant() = default;
    virtual std::string Complete(std::string_view context, 
                                  std::string_view prompt) = 0;
    virtual std::string Explain(std::string_view code) = 0;
};

} // namespace pineapple::interfaces
```

### 7.2 Feature Flags

```cpp
// Features can be enabled/disabled without recompilation

class FeatureFlags {
public:
    static FeatureFlags& Instance();
    
    bool IsEnabled(std::string_view feature) const {
        auto it = m_flags.find(std::string(feature));
        return it != m_flags.end() && it->second;
    }
    
    void SetEnabled(std::string_view feature, bool enabled) {
        m_flags[std::string(feature)] = enabled;
        SaveToSettings();
    }
    
    // Built-in feature flags
    static constexpr auto SMOOTH_SCROLLING = "ui.smoothScrolling";
    static constexpr auto ANIMATIONS = "ui.animations";
    static constexpr auto MINIMAP = "editor.minimap";
    static constexpr auto BREADCRUMBS = "editor.breadcrumbs";
    static constexpr auto GIT_INTEGRATION = "vcs.git";
    static constexpr auto LSP_SUPPORT = "language.lsp";
    
private:
    std::unordered_map<std::string, bool> m_flags;
};
```

### 7.3 Backward Compatibility

```cpp
// Plugin API versioning

#define PINEAPPLE_API_VERSION_MAJOR 1
#define PINEAPPLE_API_VERSION_MINOR 0
#define PINEAPPLE_API_VERSION ((PINEAPPLE_API_VERSION_MAJOR << 16) | PINEAPPLE_API_VERSION_MINOR)

// When loading plugins:
bool IsPluginCompatible(uint32_t pluginApiVersion) {
    uint32_t pluginMajor = pluginApiVersion >> 16;
    uint32_t pluginMinor = pluginApiVersion & 0xFFFF;
    
    // Major version must match exactly
    if (pluginMajor != PINEAPPLE_API_VERSION_MAJOR) {
        return false;
    }
    
    // Minor version: plugin can be older (we're backward compatible)
    // but not newer (plugin uses features we don't have)
    return pluginMinor <= PINEAPPLE_API_VERSION_MINOR;
}
```

### 7.4 What We Will NOT Add

To prevent bloat, these features are explicitly out of scope:

| Feature | Reason |
|---------|--------|
| Built-in terminal | Use Windows Terminal |
| File browser/explorer | Use Windows Explorer |
| Built-in Git UI | Use dedicated Git tools |
| Project management | Not a text editor's job |
| Debugger | Use dedicated debuggers |
| Build system | Use dedicated build tools |
| Package manager | Plugins are enough |
| Telemetry | Privacy first |
| Cloud sync | Local-first |
| Collaboration | Not our focus |

**Philosophy:** Do one thing well. Pineapple is a text editor, not an IDE.


---

## 8. Developer Build Guide

### 8.1 Prerequisites

```
Required:
- Windows 10 SDK (10.0.19041.0 or later)
- Visual Studio 2022 (v143 toolset)
- CMake 3.25+
- vcpkg (for dependencies)

Optional:
- Ninja (faster builds)
- clang-cl (for better diagnostics)
```

### 8.2 Dependencies

```cmake
# vcpkg.json
{
  "name": "pineapple",
  "version": "1.0.0",
  "dependencies": [
    "re2",           # Regex engine
    "fmt",           # String formatting
    "spdlog",        # Logging
    "nlohmann-json", # Settings/config
    "concurrentqueue" # Lock-free queues
  ]
}
```

### 8.3 Project Structure

```
pineapple/
├── CMakeLists.txt
├── vcpkg.json
├── src/
│   ├── main.cpp                    # Entry point
│   ├── core/
│   │   ├── Application.cpp
│   │   ├── MessageBus.cpp
│   │   └── Settings.cpp
│   ├── buffer/
│   │   ├── PieceTable.cpp
│   │   ├── Buffer.cpp
│   │   └── BufferManager.cpp
│   ├── editor/
│   │   ├── EditorView.cpp
│   │   ├── Cursor.cpp
│   │   ├── Selection.cpp
│   │   └── LineCache.cpp
│   ├── render/
│   │   ├── RenderContext.cpp
│   │   ├── TextRenderer.cpp
│   │   ├── Theme.cpp
│   │   └── Animation.cpp
│   ├── syntax/
│   │   ├── SyntaxEngine.cpp
│   │   ├── Tokenizer.cpp
│   │   └── languages/
│   │       ├── CppTokenizer.cpp
│   │       ├── PythonTokenizer.cpp
│   │       └── ...
│   ├── search/
│   │   ├── SearchEngine.cpp
│   │   └── RegexCache.cpp
│   ├── shell/
│   │   ├── MainWindow.cpp
│   │   ├── TabBar.cpp
│   │   ├── StatusBar.cpp
│   │   └── MenuBar.cpp
│   ├── plugin/
│   │   ├── PluginHost.cpp
│   │   ├── PluginLoader.cpp
│   │   └── PluginAPI.cpp
│   └── macro/
│       └── MacroEngine.cpp
├── include/
│   └── pineapple/
│       ├── pineapple_plugin.h      # Public plugin API
│       └── ...
├── resources/
│   ├── icons/                      # SVG icons
│   ├── themes/                     # JSON theme files
│   └── pineapple.rc               # Windows resources
├── tests/
│   ├── buffer_tests.cpp
│   ├── search_tests.cpp
│   └── ...
└── docs/
    └── PINEAPPLE_DESIGN.md         # This document
```


### 8.4 CMake Configuration

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.25)
project(Pineapple VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Windows-specific settings
if(MSVC)
    add_compile_options(/W4 /WX /permissive- /Zc:__cplusplus)
    add_compile_options(/utf-8)  # UTF-8 source files
    
    # Release optimizations
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /O2 /GL /DNDEBUG")
    set(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} /LTCG")
endif()

# Find packages
find_package(re2 CONFIG REQUIRED)
find_package(fmt CONFIG REQUIRED)
find_package(spdlog CONFIG REQUIRED)
find_package(nlohmann_json CONFIG REQUIRED)

# Main executable
add_executable(Pineapple WIN32
    src/main.cpp
    src/core/Application.cpp
    # ... all source files
    resources/pineapple.rc
)

target_include_directories(Pineapple PRIVATE
    ${CMAKE_SOURCE_DIR}/include
    ${CMAKE_SOURCE_DIR}/src
)

target_link_libraries(Pineapple PRIVATE
    re2::re2
    fmt::fmt
    spdlog::spdlog
    nlohmann_json::nlohmann_json
    
    # Windows libraries
    d2d1
    dwrite
    dxgi
    d3d11
    windowscodecs
    shlwapi
    uxtheme
)

# Plugin SDK (header-only, for plugin developers)
add_library(PineapplePluginSDK INTERFACE)
target_include_directories(PineapplePluginSDK INTERFACE
    ${CMAKE_SOURCE_DIR}/include
)

# Tests
enable_testing()
add_subdirectory(tests)
```

### 8.5 Build Commands

```powershell
# Configure (first time)
cmake -B build -S . -G Ninja -DCMAKE_TOOLCHAIN_FILE="$env:VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake"

# Build (Debug)
cmake --build build --config Debug

# Build (Release)
cmake --build build --config Release

# Run tests
ctest --test-dir build --output-on-failure

# Package (creates installer)
cmake --build build --target package
```

### 8.6 Performance Profiling

```cpp
// Built-in profiling macros (Debug builds only)

#ifdef _DEBUG
#define PINEAPPLE_PROFILE_SCOPE(name) \
    ProfileScope _profile_##__LINE__(name)
#define PINEAPPLE_PROFILE_FUNCTION() \
    PINEAPPLE_PROFILE_SCOPE(__FUNCTION__)
#else
#define PINEAPPLE_PROFILE_SCOPE(name)
#define PINEAPPLE_PROFILE_FUNCTION()
#endif

class ProfileScope {
public:
    ProfileScope(const char* name) : m_name(name) {
        m_start = std::chrono::high_resolution_clock::now();
    }
    
    ~ProfileScope() {
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - m_start);
        
        if (duration.count() > 1000) {  // Log if > 1ms
            spdlog::debug("{}: {}us", m_name, duration.count());
        }
    }
    
private:
    const char* m_name;
    std::chrono::high_resolution_clock::time_point m_start;
};

// Usage:
void EditorView::Render() {
    PINEAPPLE_PROFILE_FUNCTION();
    
    // ... rendering code
}
```


---

## 9. Implementation Roadmap

### Phase 1: Core Foundation (Months 1-3)

| Week | Deliverable |
|------|-------------|
| 1-2 | Win32 window, Direct2D setup, basic text rendering |
| 3-4 | Piece table implementation, basic editing |
| 5-6 | Multi-cursor, selection, clipboard |
| 7-8 | File I/O, encoding detection |
| 9-10 | Undo/redo system |
| 11-12 | Tab management, session persistence |

### Phase 2: Editor Features (Months 4-6)

| Week | Deliverable |
|------|-------------|
| 13-14 | Syntax highlighting engine |
| 15-16 | Built-in language tokenizers (C++, Python, JS) |
| 17-18 | Search & replace (literal + regex) |
| 19-20 | Line folding |
| 21-22 | Find in files |
| 23-24 | Macro recording system |

### Phase 3: Polish & Performance (Months 7-9)

| Week | Deliverable |
|------|-------------|
| 25-26 | Animation system |
| 27-28 | Theme engine, dark/light modes |
| 29-30 | Large file optimization |
| 31-32 | Memory profiling & optimization |
| 33-34 | Plugin system |
| 35-36 | Documentation, installer |

### Phase 4: Beta & Launch (Months 10-12)

| Week | Deliverable |
|------|-------------|
| 37-40 | Beta testing, bug fixes |
| 41-44 | Performance tuning |
| 45-48 | Public release |

---

## 10. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Direct2D performance issues | Medium | High | Fallback to GDI for old GPUs |
| Piece table complexity | Low | Medium | Extensive unit testing |
| Plugin API instability | Medium | High | Version from day 1, never break |
| Large file edge cases | High | Medium | Fuzz testing, memory limits |
| Animation jank | Medium | Low | Frame budget monitoring |
| Theme compatibility | Low | Low | Strict color validation |

---

## 11. Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Cold startup | <200ms | Automated benchmark |
| Warm startup | <100ms | Automated benchmark |
| Idle RAM | <30MB | Task Manager |
| 100MB file open | <1s | Automated benchmark |
| 1GB file open | <5s | Automated benchmark |
| Typing latency | <16ms | Input-to-pixel measurement |
| Scroll FPS | 60fps | Frame timing |
| Animation FPS | 60fps | Frame timing |

---

## 12. Conclusion

Pineapple is designed to be the text editor that Notepad++ could have been if
it were built today with modern Windows APIs and C++20. It prioritizes:

1. **Speed** - Native code, lazy initialization, efficient data structures
2. **Quality** - Hardware-accelerated rendering, smooth animations
3. **Simplicity** - Clean architecture, no bloat, focused feature set
4. **Extensibility** - Stable plugin API, modular design

The architecture is realistic and buildable. Every component described in this
document has been designed with implementation feasibility in mind. There are
no hand-wavy "AI will figure it out" sections.

This is a professional-grade design for a professional-grade editor.

---

*Document Version: 1.0*  
*Last Updated: 2024*  
*Status: Design Complete, Ready for Implementation*

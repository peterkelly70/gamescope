# OpenGL compatibility renderer

## Goal

Add a legacy nested renderer that lets Gamescope run on GPUs which have usable OpenGL support but no usable Vulkan implementation.

The first target is old Windows games under Wine on X11. The renderer must isolate the game's requested resolution from the physical displays while Gamescope itself can present as a borderless or fullscreen host window.

## Why this is a compatibility renderer

Current Gamescope is architected around Vulkan. Vulkan types are present in the backend interface and `FrameInfo_t` carries `CVulkanTexture` objects. The SDL backend creates a Vulkan surface and calls the Vulkan compositor directly.

The initial OpenGL mode therefore deliberately implements only the subset needed for legacy games instead of duplicating every modern Gamescope feature.

## MVP scope

Supported initially:

- nested mode only
- SDL2 host window
- Xwayland clients
- SDR output
- single output
- fullscreen and borderless host presentation
- nearest and linear scaling
- letterbox / fit / stretch
- cursor
- one primary game surface plus simple overlay/cursor layers
- Wine resolution changes contained inside the nested display

Not required for the first milestone:

- DRM/embedded mode
- HDR or color-management LUTs
- VR
- FSR
- NIS
- ReShade
- direct scan-out
- explicit sync
- variable refresh rate
- PipeWire capture
- hardware planes

## Renderer strategy

### Stage 1: broad compatibility path

Use an SDL OpenGL context for presentation and a simple OpenGL textured-quad compositor.

For the oldest hardware, prefer CPU-accessible client buffers and upload them with `glTexSubImage2D`. In this mode Xwayland can be run without glamor where necessary so that a lack of DMA-BUF/OpenGL interop does not prevent operation.

This path trades copies for compatibility. That is acceptable for the initial target: older DirectDraw/OpenGL games at modest resolutions.

### Stage 2: zero-copy path

Add EGL DMA-BUF import where the driver exposes it. Client buffers can then become GL textures without a CPU readback/upload.

Stage 1 remains as the fallback.

## Existing code we can reuse

Gamescope descends from Valve's `steamos-compositor`. The old `steamcompmgr.c` renderer used OpenGL/GLX and `GLX_EXT_texture_from_pixmap`. Its window selection, scaling and GL composition code is useful reference material even though modern Gamescope's Xwayland/wlroots buffer path requires a different import layer.

The goal is to reuse the old compositing behaviour and mathematics, not to restore the old compositor wholesale.

## Refactor boundary

The long-term renderer boundary should be API-neutral.

A renderer owns:

- texture import
- texture lifetime
- frame composition
- presentation
- synchronization

Core window-management code should describe a frame without requiring Vulkan objects.

A minimal direction is:

```cpp
class ICompositeTexture;

struct CompositeLayer
{
    Rc<ICompositeTexture> texture;
    vec2_t offset;
    vec2_t scale;
    float opacity;
    GamescopeUpscaleFilter filter;
};

class IRenderer
{
public:
    virtual ~IRenderer() = default;
    virtual bool Init() = 0;
    virtual Rc<ICompositeTexture> ImportBuffer(wlr_buffer *buffer) = 0;
    virtual bool CompositeAndPresent(const CompositeFrame &frame) = 0;
};
```

The Vulkan implementation should initially be a thin adapter over existing `rendervulkan.cpp` behaviour. The OpenGL implementation can then be added without putting GL conditionals throughout `steamcompmgr.cpp`.

## First implementation milestones

1. Introduce an API-neutral renderer selector and interface while retaining Vulkan as the only implementation.
2. Move SDL presentation ownership out of Vulkan-specific code.
3. Add `SDL_WINDOW_OPENGL` context creation for the GL renderer.
4. Implement an RGBA CPU-upload texture path.
5. Draw a single client layer with nearest/linear scaling.
6. Make `gamescope --renderer gl -f -- wine ...` contain game mode changes and present fullscreen.
7. Add cursor and additional simple layers.
8. Add EGL DMA-BUF import as an optional fast path.
9. Only after the legacy path is stable, consider advanced effects.

## Compatibility target

The initial renderer should avoid requiring modern OpenGL features. OpenGL 2.1-class hardware with framebuffer objects and basic GLSL is a reasonable desktop baseline; a GLES2 implementation can share most shader logic if required for embedded Mesa drivers.

## Test case

Total Annihilation under Wine is the first acceptance test:

- game may request its historical fullscreen resolutions
- physical monitor layout and resolution must not change
- Gamescope host window remains borderless/fullscreen at desktop resolution
- mouse and keyboard focus remain correct
- alt-tab restores the desktop without a display modeset

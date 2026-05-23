# TrussC Graphics Reference

## Drawing Primitives

All available as global functions (delegate to default RenderContext).

```cpp
drawRect(x, y, w, h)
drawRectRounded(x, y, w, h, radius)       // Circular arc corners
drawRectSquircle(x, y, w, h, radius)      // Superellipse corners
drawCircle(cx, cy, radius)
drawEllipse(cx, cy, rx, ry)
drawLine(x1, y1, x2, y2)                  // Always 1px (GL_LINES)
drawTriangle(x1, y1, x2, y2, x3, y3)
drawPoint(x, y)

// 3D
drawBox()
drawSphere(radius, resolution)
drawCone(radius, height, resolution)
```

Most accept Vec3/Vec2 overloads too.

**Note:** `drawLine()` is always 1px. `setLineWidth()` does NOT exist in sokol. For thick lines, build a `Path` and call `path.drawStroke()` (uses StrokeMesh internally), or assemble StrokeMesh directly.

---

## Curves & Bézier  *(v0.5.2+)*

Strokes only — open curves can't be filled. Segment count comes from the
current `CurveStyle` (see **Curve Quality** under Style below).

```cpp
// Bézier (cubic / quadratic / N-th order via de Casteljau).
drawBezier(p0, p1, p2, p3)
drawBezier(p0, p1, p2)
drawBezier(vector<Vec3>& ctrlPoints)

// Catmull-Rom interpolating spline. drawCurve(p0,p1,p2,p3) emits the
// segment p1→p2 with p0/p3 as tangent influences (oF semantics).
drawCurve(p0, p1, p2, p3)
drawCurve(vector<Vec3>& pts)              // chained segments
drawCurve(vector<Vec3>& pts, bool closed) // wraps last↔first when closed

// Arc — radians, counter-clockwise convention.
drawArc(x, y, radius, angleBegin, angleEnd)
drawArc(Vec3 center, radius, angleBegin, angleEnd)
```

For chainable paths use `Path` (also tolerance-aware):

```cpp
Path p;
p.moveTo(0, 0);
p.lineTo(100, 0);
p.bezierTo(120, 0, 140, 20, 140, 40);          // cubic
p.quadBezierTo(150, 80, 100, 100);             // quadratic
p.curveTo(50, 120);                            // Catmull-Rom (needs ≥4 pts)
p.arc(Vec2(100, 100), 30, 0, HALF_TAU);        // arc segment
p.close();
p.draw();                                      // fill or stroke per current style
p.drawStroke();                                // thick stroke via StrokeMesh
```

Pass `resolution = -1` (default) on `Path` curve methods to honour the
current `CurveStyle`; pass a positive int to force a fixed segment count.

---

## Style

### Color

```cpp
setColor(r, g, b, a = 1.0)        // 0.0–1.0 float
setColor(gray, a = 1.0)           // Grayscale
setColor(Color c)
getColor() -> Color

// Other color spaces
setColorHSB(h, s, b, a)           // H: 0–1
setColorOKLab(L, a, b, alpha)
setColorOKLCH(L, C, H, alpha)
```

### Fill / Stroke

```cpp
fill()                             // Solid shapes (disables stroke)
noFill()                           // Outlines only (disables fill)
isFillEnabled()
isStrokeEnabled()
```

Fill and stroke are **mutually exclusive**.

### Stroke Settings

```cpp
setStrokeWeight(float)
setStrokeCap(StrokeCap)            // Butt, Round, Square
setStrokeJoin(StrokeJoin)          // Miter, Round, Bevel
```

### Style Stack

```cpp
pushStyle()                        // Save color, fill/stroke, textAlign
popStyle()                         // Restore
resetStyle()
```

### Curve Quality

One knob controls segment count for **circle, ellipse, roundedRect,
squircle, arc, bezier, curve, and `Path` curves** — all routed through
`decideCircleSegments` / `decideArcSegments`.

```cpp
setCurveTolerance(0.5f);          // adaptive (default): target pixel error.
                                  // accounts for scale()/zoom, so curves
                                  // stay smooth when transformed.
setCurveResolution(24);           // fixed segment count regardless of zoom.

setCircleResolution(20);          // deprecated alias — forwards to
                                  // setCurveResolution with a warning.
```

---

## Color Class

```cpp
Color(r, g, b, a = 1.0)
Color(gray, a = 1.0)
Color::fromBytes(r, g, b, a = 255)    // 0–255 range
Color::fromHex(0xRRGGBB)
Color::fromHSB(h, s, b, a)            // H: 0–1
Color::fromOKLCH(L, C, H, a)
Color::fromOKLab(L, a, b, alpha)

// Interpolation (default uses OKLab — perceptually uniform)
color.lerp(target, t)
color.lerpRGB(target, t)
color.lerpHSB(target, t)
color.lerpOKLab(target, t)
color.lerpOKLCH(target, t)

// Conversion
color.toLinear()    // ColorLinear
color.toHSB()       // ColorHSB
color.toOKLab()     // ColorOKLab
color.toOKLCH()     // ColorOKLCH

// Predefined (~120 named colors)
colors::white, colors::black, colors::cornflowerBlue, colors::salmon ...
```

---

## Text

### Bitmap String (built-in fixed font)

```cpp
drawBitmapString(text, x, y)
drawBitmapString(text, x, y, scale)
drawBitmapString(text, x, y, hAlign, vAlign)

getBitmapStringWidth(text)
getBitmapStringHeight(text)
getBitmapFontHeight()              // ~18px

setTextAlign(Direction h, Direction v)
setBitmapLineHeight(float)
```

### Font (TrueType via stb_truetype)

```cpp
Font font;
font.load("/path/to/font.ttf", 14);
font.setAlign(Direction::Center, Direction::Center);
font.setLineHeightEm(1.2);
font.drawString(text, x, y);
// Also: drawString(text, x, y, hAlign, vAlign) overload
```

Platform fonts:
```cpp
TC_FONT_SANS   // Helvetica (macOS)
TC_FONT_SERIF  // Times
TC_FONT_MONO   // Menlo
```

### Direction Enum

```cpp
Direction::Left, Center, Right, Top, Bottom, Baseline
// Also: tc::Left, tc::Center, tc::Right (without ::)
```

---

## Matrix Transforms

```cpp
pushMatrix()
popMatrix()
resetMatrix()

translate(x, y)  or  translate(x, y, z)  or  translate(Vec3)
rotate(radians)                         // Z-axis
rotateX(radians); rotateY(radians); rotateZ(radians);
scale(s)  or  scale(sx, sy)  or  scale(sx, sy, sz)

getCurrentMatrix() -> Mat4
```

---

## Pixels (CPU-only pixel buffer)

Safe in background threads. No GPU dependency.

```cpp
Pixels pix;
pix.allocate(w, h, channels = 4, PixelFormat::U8);
pix.load("/path/to/image.jpg");
pix.save("/path/to/output.png");

pix.isAllocated();
pix.getWidth(); pix.getHeight(); pix.getChannels();
pix.getData();                     // unsigned char*
pix.getDataF32();                  // float* (if F32 format)

pix.getColor(x, y);
pix.setColor(x, y, Color);
pix.clone();                       // Deep copy

// No copy constructor — use std::move()
Pixels p2 = std::move(pix);
```

PixelFormat: `U8` (unsigned char) or `F32` (float)

---

## Texture (GPU-only)

Main thread only. Created from Pixels or empty.

```cpp
Texture tex;
tex.allocate(pix, TextureUsage::Immutable);    // From Pixels
tex.allocate(w, h, 4, TextureUsage::Dynamic);  // Empty

tex.draw(x, y);
tex.draw(x, y, w, h);
tex.drawSubsection(x, y, w, h, sx, sy, sw, sh);

tex.loadData(pix);                 // Update (Dynamic/Stream only)
tex.bind(); tex.unbind();          // For shader use

tex.setFilter(TextureFilter::Linear);     // Linear or Nearest
tex.setWrap(TextureWrap::ClampToEdge);    // ClampToEdge, Repeat, MirroredRepeat

tex.clear();
```

**TextureUsage:**
- `Immutable` — set once, fastest (for loaded images)
- `Dynamic` — update occasionally (for live editing)
- `Stream` — update every frame (for video)
- `RenderTarget` — for Fbo attachment

**Limitation:** `loadData()` can only be called ONCE per frame. Second call is silently skipped.

---

## Image (CPU + GPU unified)

Main thread only. Combines Pixels + Texture.

```cpp
Image img;
img.load("/path/to/image.jpg");    // Creates Immutable texture
img.allocate(w, h, 4);             // Creates Dynamic texture

img.draw(x, y);
img.draw(x, y, w, h);

// Pixel manipulation
img.getColor(x, y);
img.setColor(x, y, Color);
img.getPixels();                   // Pixels& reference
img.update();                      // Sync pixel changes to GPU (once/frame!)

img.save("/path/to/output.png");
img.clear();
```

---

## Fbo (Framebuffer / Off-screen Render)

Main thread only.

```cpp
Fbo fbo;
fbo.allocate(w, h, sampleCount = 1);  // sampleCount > 1 for MSAA

fbo.begin();                    // Start rendering to fbo
fbo.begin(r, g, b, a);         // With clear color
// ... draw commands ...
fbo.end();                      // Back to screen

fbo.draw(x, y);                // Display result
fbo.save("screenshot.png");

fbo.getTexture();               // For shader binding
fbo.copyTo(Image&);             // Readback to Image
```

**Nested Fbo not supported** (sokol limitation).

---

## Shader

```cpp
Shader shader;
shader.load(myShaderDesc);         // sokol-shdc generated descriptor

shader.begin();
shader.setUniform(0, someValue);   // float, Vec2/3/4, Color
shader.setTexture(0, tex.getImage(), tex.getSampler());
drawRect(0, 0, 100, 100);         // Drawn with shader
shader.end();

drawCircle(50, 50, 20);           // Normal sokol_gl drawing
```

Shader draws are deferred and execute between sokol_gl layers. Stack-based for nesting.

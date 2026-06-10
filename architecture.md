# Structuring TrussC Apps — Node Hierarchy & Mods

TrussC's API is openFrameworks-like on the surface, but its intended app structure is
**Unity-like**: a scene graph of Node subclasses (≈ GameObjects with behavior) plus
Mods attached to them (≈ Components). The most common structural mistake is writing
the whole app inside `tcApp::update()` / `tcApp::draw()` with manual coordinate math —
that works for a sketch, but TrussC gives you better tools the moment an app has more
than a couple of elements.

**Two separation axes:**

1. **Vertical (hierarchy):** each visual/interactive element is its own Node subclass —
   it owns its state, draws itself in local coordinates, and handles its own input.
   The parent only composes children; it doesn't micro-manage them.
2. **Horizontal (Mods):** cross-cutting behavior (animation, layout, dragging, physics…)
   lives in Mods attached to nodes — so the node's `update()`/`draw()` stay clean and
   the behavior is reusable on any node.

## The App IS the Root Node

`App` inherits from `RectNode` (which inherits `Node`), auto-sized to the window. So
`tcApp` is the scene root: `addChild()` in `setup()` and the framework recursively
updates, draws, and dispatches events through the whole tree. You rarely need code in
`tcApp::update()` — the tree updates itself.

```cpp
void tcApp::setup() {
    scene_ = make_shared<GameScene>();
    hud_   = make_shared<Hud>();
    addChild(scene_);
    addChild(hud_);        // draws after scene_ (sibling order = draw order)
}
void tcApp::draw() {
    clear(0.1f);           // background; children draw themselves afterwards
}
```

## Writing a Node Subclass

```cpp
class EnemyShip : public Node {
public:
    EnemyShip(float speed) : speed_(speed) {}   // plain members in constructor

    void setup() override {
        // runs once before first update/draw; shared_ptr is complete here,
        // so addChild()/addMod()/callEvery() are safe (NOT in the constructor!)
        addChild(make_shared<HealthBar>());
        callEvery(2.0, [this]() { fire(); });
    }

    void update() override {
        setX(getX() - speed_ * getDeltaTime());
        if (getX() < -50) destroy();            // deferred, safe mid-update
    }

    void draw() override {
        // local coordinates: (0,0) is this node's origin.
        // Parent transform (pos/rot/scale) is already applied.
        setColor(colors::tomato);
        drawCircle(0, 0, 20);
    }

protected:
    bool onMousePress(const MouseEventArgs& e) override {
        explode();
        return true;        // consume — stop propagation
    }
private:
    float speed_;
};
```

Key rules:

- **Constructor = plain state, `setup()` = tree work.** `addChild()` in a constructor
  crashes (`weak_from_this()` not ready). `setup()` is auto-deferred to before the
  node's first update/draw, even when added mid-frame.
- **Draw in local coordinates.** Never compute "where am I on screen" — set the node's
  transform (`setPos`/`setRot`/`setScale`) and draw around (0,0). Children inherit the
  transform automatically. `getMouseX()/getMouseY()` inside a node are local too.
- **Input is opt-in:** call `enableEvents()` (base `Node` does not enable it; `RectNode`
  does). Mouse events arrive in local coords; return `true` to consume.
- **Lifecycle:** `destroy()` marks the node dead (removed safely later); `cleanup()` is
  the teardown hook; `setActive(false)` disables update+draw+events for the subtree;
  `setVisible(false)` hides but keeps updating.

### Node or RectNode?

- **`RectNode`** — anything 2D-UI-ish: it has `setSize()`, rect hit-testing, events
  enabled by default, `setClipping(true)` scissor, `onSizeChanged()`, and per-instance
  `mousePressed`/`mouseDragged`… `Event` members you can listen to from outside.
- **`Node`** — pure behavior, 3D content, or free-form 2D geometry. No size; override
  `hitTest(Ray localRay, float& outDistance)` if it needs custom picking.

### Hierarchy = code separation

A practical decomposition for a mid-size app:

```
tcApp (root)
├── GameScene : Node            // world coords, owns game state
│   ├── Background : Node
│   ├── Player : Node           // input + movement + its own draw
│   │   └── WeaponMount : Node  // offset child, inherits player transform
│   └── EnemyShip × N           // spawned with addChild, die with destroy()
└── Hud : RectNode              // screen coords, drawn on top (sibling order)
    ├── ScoreLabel : RectNode
    └── PauseButton : RectNode  // onMousePress → consume
```

- **Scene switching:** keep scenes as siblings and toggle `setActive()` — an inactive
  subtree costs nothing and receives no events.
- **Draw order** is sibling order (`moveToFront()`/`moveToBack()` to reorder).
- Mutating the tree during update/draw/dispatch is safe (children are snapshotted;
  `destroy()` and removals are deferred).
- To *see* the structure while building it, drop in **tcxNodeInspector** (bundled
  addon): a Unity-style hierarchy + inspector + gizmo over the live tree. Great for
  checking that your decomposition actually looks like the diagram above.

## Loose Coupling with Event<T> + EventListener

The third pillar besides hierarchy and Mods. The rule that keeps a node tree from
turning into a web of cross-references: **dependencies point downward only.** A parent
knows its children; a child never knows its parent's type or its siblings. Anything a
child needs to tell the outside world goes out through an `Event<T>` member — whoever
cares listens.

```cpp
class PauseButton : public RectNode {
public:
    Event<void> pressed;                       // emitter: public Event member
protected:
    bool onMousePress(const MouseEventArgs& e) override {
        pressed.notify();
        return true;
    }
};

class Hud : public RectNode {
    void setup() override {
        auto btn = make_shared<PauseButton>();
        addChild(btn);
        // listener side: keep the EventListener as a member — RAII.
        // When Hud dies, the connection dies with it. No dangling callbacks.
        pauseListener_ = btn->pressed.listen([this]() {
            pauseRequested.notify();           // re-fire upward (bubbling)
        });
    }
public:
    Event<void> pauseRequested;                // Hud's own event, coarser-grained
private:
    EventListener pauseListener_;
};
```

Why this beats raw `function<>` callbacks: the `EventListener` disconnects itself on
destruction, so a listener outliving (or outlived by) the emitter is never a crash.
Multiple listeners can attach to one event, notify is thread-safe, and removal during
notify is safe.

Design guidance:

- **Bubble, don't broadcast:** each layer listens to its children and re-fires a
  *semantically coarser* event of its own (`PauseButton::pressed` →
  `Hud::pauseRequested` → app pauses the scene). Layers stay swappable.
- The same mechanism is used by the framework itself — `RectNode::mousePressed`,
  `Node::localMatrixChanged`, `TweenMod::complete` are all `Event<T>` — so app wiring
  and framework wiring read identically.
- Full API rules (T& argument, Event<void>, storing listeners): see
  [node-system.md](node-system.md) § Event System.

### App-wide events: the AppEvents singleton (recommended)

Bubbling is right for parent↔child. But some signals are genuinely app-global —
"show a modal", "config was re-downloaded", "operator locked the controls" — and
bubbling those through five unrelated layers couples every layer to concerns it
doesn't own. For these, the recommended pattern is **one event-bus struct as a Meyers
singleton**:

```cpp
// AppEvents.h
#pragma once
#include <TrussC.h>
using namespace std;
using namespace tc;

struct ModalEventArgs {
    string message;
    float autoDismiss = 0.0f;   // 0 = manual dismiss only
};

// App-wide event bus (singleton)
struct AppEvents {
    // Group events by concern, one comment each — this file IS the
    // documentation of "what can happen in this app".
    Event<ModalEventArgs> showModal;
    Event<void>  dismissModal;
    Event<void>  configUpdated;     // config.json downloaded & saved
    Event<float> syncProgress;      // 0.0-1.0
    Event<bool>  lockChanged;

    // Shared state that mirrors an event. Listeners may also poll the flag.
    // NEVER write the flag directly — fire the event; a self-listener syncs it.
    bool isLocked = false;

    AppEvents() {
        lockSelfListener_ = lockChanged.listen([this](bool& v) { isLocked = v; });
    }
private:
    EventListener lockSelfListener_;
};

AppEvents& appEvents();

// AppEvents.cpp
AppEvents& appEvents() {
    static AppEvents instance;   // Meyers singleton: constructed on first use
    return instance;
}
```

Usage from anywhere in the tree — emitter and listener never know each other:

```cpp
// Some deeply nested settings widget:
appEvents().lockChanged.notify(true);

// A toolbar on the other side of the app:
lockListener_ = appEvents().lockChanged.listen([this](bool& locked) {
    setActive(!locked);
});
```

Rules that keep the bus healthy:

- **Events are the write path, flags are the read path.** If a piece of state lives
  on the bus (`isLocked`), it is synced by a self-listener in the constructor; code
  fires the event and never assigns the flag. One source of truth, and every
  listener still gets notified.
- **Typed payload structs** (`ModalEventArgs`) over parallel primitive events — the
  payload can grow without re-wiring listeners.
- **Comment each event** with who fires it and who's expected to react — the bus
  header becomes the app's interaction map.
- **Don't dump local events on the bus.** A button's `pressed` belongs to the button;
  the bus is only for signals with multiple unrelated listeners across the tree.
  If everything goes through the bus, you've traded sibling-coupling for
  everything-coupled-to-everything.
- One cascade level is fine (e.g. `experienceStarted` self-listener also starts the
  global timer), but if bus events start firing chains of other bus events, the flow
  has become invisible — refactor.

## Mods — behavior without polluting update/draw

A Mod attaches to a node and gets its own lifecycle hooks, input events, and even hit
shape — so common functionality doesn't accumulate inside every node's `update()`.

```cpp
node->addMod<TweenMod>();                 // attach; setup() runs immediately
auto* layout = node->getMod<LayoutMod>(); // type-indexed lookup
node->removeMod<TweenMod>();              // deferred if called mid-dispatch
```

Mod hooks (all optional):

| Hook | When | Use for |
|------|------|---------|
| `setup()` | inside `addMod<T>()`, owner already set | init, read owner |
| `earlyUpdate()` | **before** owner's `update()` | physics, tweens, anything the node should *see applied* |
| `update()` | after owner's `update()` | reacting to node changes |
| `draw()` | after owner's `draw()`, in local space | overlays, gizmos, debug |
| `onMousePress(MouseEventArgs)` etc. | same local-space events as the node; return `true` to consume | interaction behaviors |
| `hitTest(Ray, float&)` | alongside the node's own | extra/custom hit shapes |
| `onDestroy()` | on removal or owner death | teardown |

Built-in Mods:

- **`LayoutMod`** (exclusive, RectNode children) — VStack/HStack auto-arrangement:
  `addMod<LayoutMod>(LayoutDirection::Vertical, 4.0f)` then `setPadding()`,
  `setMainAxis(AxisMode::Content)` (parent grows to fit children),
  `setCrossAxis(AxisMode::Fill)`. Call `updateLayout()` after manual size changes.
- **`TweenMod`** (non-exclusive — several can run at once) — property animation:
  `tween->moveTo(100, 200).duration(0.5).ease(EaseType::Cubic).start();` plus
  `scaleTo`/`rotateTo`/`moveBy`/`moveFrom` variants, `delay()`, `complete` Event,
  `isPlaying()`. Runs in `earlyUpdate()`, so the node's `update()` sees the tweened
  transform.

### Subclass or Mod? — the decision rule

- **Identity → subclass.** "What this thing *is*" (a button, an enemy, a panel):
  Node subclass.
- **Capability → Mod.** "Something this thing *can also do*" (animate, lay out
  children, be draggable, blink, snap to grid, emit particles) — especially if two
  unrelated node types might want it: Mod.
- If you're copy-pasting the same block between two nodes' `update()` functions,
  that block wants to be a Mod.

### Writing your own Mod

```cpp
// Makes any node draggable — no changes to the node class itself.
class DraggableMod : public Mod {
public:
    bool onMousePress(const MouseEventArgs& e) override {
        grab_ = e.pos;                       // local coords
        return true;                         // consume
    }
    bool onMouseDrag(const MouseEventArgs& e) override {
        auto* n = getOwner();
        n->setPos(n->getPos() + Vec3(e.pos - grab_));
        return true;
    }
};

anyNode->addMod<DraggableMod>();
```

Override `isExclusive()` → `true` for one-per-node mods (like LayoutMod), and
`canAttachTo(Node*)` to reject incompatible owners (e.g. require a RectNode).

Mod gotchas:

- `addMod<T>()` runs `setup()` **synchronously** and returns `T*` — chainable:
  `node->addMod<LayoutMod>(...)->setPadding(5);`
- Removal during dispatch is deferred until the outermost dispatch returns —
  `removeSelf()` from your own event handler is safe.
- Mod `draw()` happens after the node's `draw()` but before children? No —
  order per node is: `beginDraw()` → `draw()` → mods' `draw()` → `drawChildren()` → `endDraw()`.

## Camera Scope Nodes (3D inside the tree)

To put 3D content in the hierarchy, write a node that brackets a camera in its
`beginDraw()`/`endDraw()` — children render *and pick* under that camera:

```cpp
class CameraRig : public Node {
public:
    EasyCam cam;
protected:
    void beginDraw() override { cam.begin(); }
    void endDraw()   override { cam.end(); }
};

auto rig = make_shared<CameraRig>();
addChild(rig);
rig->addChild(make_shared<SpinningCube>());   // drawn & mouse-picked in 3D
```

TrussC stamps each node with the camera context it was drawn under, and screen picking
unprojects through *that* camera per node — so 2D UI siblings and 3D children coexist
and both hit-test correctly. (Caveat: a node drawn under two cameras in one frame keeps
the last stamp; FBO-rendered content is unpickable from the screen.)

## Event Flow (who gets the click?)

1. Overlay listeners (e.g. tcxImGui at `BeforeApp` priority) may consume first —
   ImGui windows eat their own clicks.
2. `tcApp`'s own `mousePressed()`/`keyPressed()` virtuals fire.
3. If not consumed: tree dispatch — mouse events hit-test (reverse draw order: topmost
   first) and the hit node's handlers run, node mods included; return `true` stops it.
   Key events broadcast deepest-first to event-enabled nodes.

## Anti-Patterns (refactor signals)

- `tcApp::draw()` full of `pushMatrix()/translate()` blocks for distinct elements →
  those are child nodes wanting to exist.
- Screen-coordinate math inside an element ("my x is parent.x + offset") → set the
  transform, draw locally.
- A node's `update()` that interpolates positions manually → `TweenMod`.
- Manual x/y bookkeeping for stacked UI items → `LayoutMod` (+ `ScrollContainer` +
  `ScrollBar` for lists; see node-system.md).
- One node directly calling methods on siblings → `Event<T>` / `EventListener`.
- `if (gameState == ...)` fanned through update/draw → scenes as sibling nodes toggled
  with `setActive()`.

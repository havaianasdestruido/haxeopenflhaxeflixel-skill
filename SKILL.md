# Haxe + HaxeFlixel + OpenFL Development Skill

```yaml
name: haxeflixel-openfl-dev
description: >
  Enables accurate, safe, and idiomatic development, debugging, refactoring,
  and optimization of Haxe projects using HaxeFlixel and OpenFL (and the
  underlying Lime toolchain). Use whenever the working repository contains
  .hx files, a project.xml (Lime/OpenFL), .hxml files, or dependencies on
  openfl, lime, flixel, or flixel-addons.
```

---

## 0. Operating Principles

These override generic coding instincts. Internalize before writing any code:

1. **This is not JavaScript/Java/C#/C++.** Haxe has its own type system, compile-time metaprogramming, target-specific compilation, and a static single-pass compiler. Do not port idioms from other languages without checking Haxe equivalents first.
2. **Truth lives in the repo, not in memory.** HaxeFlixel and OpenFL have changed APIs significantly across major versions (Flixel 4 → 5, OpenFL 8 → 9 → 10, Lime 7 → 8). Before using or suggesting any API, verify it exists in the **actual installed version** in this project (`haxelib list`, `haxelib.json`, `project.xml`/`.hxml` version pins, or the vendored library source under `.haxelib/` / lix `haxe_libraries/`).
3. **Inspect before you architect.** Read the existing state machine, folder layout, naming conventions, and helper utilities before proposing new patterns. Match what's already there unless asked to change it.
4. **Minimal, targeted diffs.** Don't refactor unrelated code, rename things, reformat files, or "improve" style while fixing a bug or adding a feature, unless asked.
5. **Preserve behavior.** Any change must not alter existing observable behavior unless that is the explicit goal. When in doubt, prefer additive changes over rewriting.
6. **Distinguish fact from hypothesis.** State clearly when something is confirmed by reading source/docs vs. when it's a guess that needs verification (e.g., by running the build, adding a trace, or checking a changelog).
7. **No unjustified `Dynamic`, reflection, or casts.** Haxe's static typing is a feature. Use abstracts, generics, enums, and typedefs before reaching for `Dynamic`, `untyped`, or `cast`.
8. **Respect the compiler's dead-code elimination (DCE) and target semantics.** Code that "should" run may be eliminated, conditionally compiled out, or behave differently on `html5` vs `cpp` vs `hl`.

---

## 1. Mandatory Investigation Protocol

Before making non-trivial changes, gather the following (skip only what's clearly irrelevant to the task):

### 1.1 Identify versions in use
- `haxelib list` or `haxe_libraries/*.hxml` (lix) → exact `flixel`, `flixel-addons`, `openfl`, `lime` versions.
- `project.xml` `<haxelib name="flixel" />` (version may be pinned or come from `haxelib.json`).
- Check `Project.xml`/`application.xml` for `<set name="..." />`, `<haxedef />`, and target definitions.
- If a library is git-referenced (`<haxelib name="flixel" git="..." />` or lix `.lock`), the API may differ from the last official release — check the actual source in `.haxelib`/`haxe_libraries`, not the public docs, if there's any doubt.

### 1.2 Map the project structure
- Where is `Main.hx` (entry point)? What does it do before creating `FlxGame`?
- Where are states (`FlxState` subclasses)? Substates? Is there a state-transition helper/manager?
- Where are assets referenced — `Assets/`, `assets/`, via `openfl.utils.Assets` or Flixel's `AssetPaths`/generated asset classes (from `flixel-tools`/`openfl-cli` asset generation)?
- Are there custom base classes (`FlxSprite` subclass conventions, custom `FlxState` base, signal/event bus, custom camera system)?
- Look for `.hxml` files (`build.hxml`, `hxml/*.hxml`) vs `project.xml`-driven builds (`openfl build`, `lime build`) — determine which build path is actually used (check README, CI config, `Makefile`, VSCode `tasks.json`).

### 1.3 Understand any metadata/macro layer before touching it
See **Section 12**. Never delete or "clean up" unfamiliar `@metadata` without tracing its consumer.

### 1.4 Check for existing conventions
- Null safety mode (`--macro checkAllFields`? `@:nullSafety`? strict null checks via `-D` defines?).
- Naming convention for signals/events, private field prefixing, use of `FlxSignal` vs OpenFL `Event`.
- Whether the project uses `flixel-addons` (FlxUI, FlxSpriteGroup extensions, transition effects, FlxNestedSprite, particle FX, etc.) or third-party libs (e.g., `flixel-ui`, `funkin`-style forks, `hscript`, `hxcpp`-specific natives).

---

## 2. Toolchain Anatomy — How Haxe, Lime, OpenFL, and HaxeFlixel Fit Together

```
Haxe compiler (haxe/hxml)
   └─ compiles .hx → target output (js, cpp/hxcpp, hl/c, neko, jvm, python, swf, cs...)

Lime (foundation layer)
   └─ provides: window/stage creation, input backends, asset packaging,
      native/CFFI bindings, platform templates, build tooling (`lime build/test`)
   └─ project.xml (Lime XML format) drives asset lists, icons, permissions,
      per-target config, and generates the .hxml passed to the Haxe compiler

OpenFL (on top of Lime)
   └─ provides Flash-API-compatible display list: Sprite, DisplayObject,
      BitmapData, Graphics, TextField, Event model, Stage
   └─ reuses Lime's windowing/input/asset system under the hood
   └─ its own project.xml is a superset of Lime's (adds swf-lib support,
      asset embedding for the display list, etc.)

HaxeFlixel (on top of OpenFL)
   └─ FlxGame extends openfl.display.Sprite (added to the OpenFL display tree)
   └─ Provides game loop (FlxState, FlxObject, update/draw cycle),
      camera system built on OpenFL's rendering primitives,
      input wrappers around Lime/OpenFL input events, FlxG global services
```

### 2.1 Build pipeline (typical OpenFL/Lime CLI flow)

1. `lime build <target>` (or `openfl build <target>`, which delegates to Lime) reads `project.xml`.
2. Lime resolves haxelib dependencies, merges per-target `<haxedef>`/`<define>`, assembles asset manifest, and **generates an `.hxml`** (visible in `export/<target>/haxe/` or similar) plus generated glue code (e.g., `ApplicationMain.hx`, asset manifest classes like `__ASSET__...` under `export/.../haxe/manifest` or `src`).
3. Haxe compiler runs with that generated hxml, compiling to the target's intermediate/native form:
   - `cpp` → generates C++ via hxcpp, then invokes native compiler/linker (per-platform).
   - `html5` → generates JS (+ optional asset copy).
   - `hl` → generates HashLink bytecode/C.
   - `neko` → Neko bytecode (legacy dev target).
   - `android`/`ios` → cpp target wrapped in native project (Gradle / Xcode) generated by Lime.
4. Final packaging step (per target) produces the runnable artifact (`.exe`, `.app`, `index.html`+`bin`, `.apk`, `.ipa`).

**Key implication for debugging:** a bug may originate at any of these layers — Haxe source, Lime's generated glue/asset manifest, or the native backend. When something "impossible" happens (wrong asset loaded, black screen, crash only on one platform), consider regenerating (`lime rebuild`, clear `export/` cache) before assuming the `.hx` logic is wrong.

### 2.2 Initialization order (matters for lifecycle bugs)

1. Native/platform entry point (generated `Main`/`ApplicationMain`) sets up the Lime `Application`.
2. Lime creates the `Window`, GL context (or Canvas/Cairo context for html5/software targets), and dispatches `onWindowCreate`.
3. OpenFL's `Lib`/`Application` wires the Lime window to an OpenFL `Stage`.
4. **Preloader** runs first if assets are declared for async/streamed load (especially html5) — code in `Main` after `FlxGame` construction may run *before* all assets are ready unless you gate on the preloader completing (`Preloader`/`NMEPreloader`/custom `openfl.display.Preloader` subclass, or Flixel's own preloader hook if configured in `project.xml`).
5. `Main.hx` typically does: `super()`; `addChild(new FlxGame(...))`. `FlxGame`'s constructor sets up `FlxG`, camera, sound, input managers, then transitions to the initial `FlxState` (the class passed to `FlxGame`).
6. `FlxState.create()` runs after the state is attached to `FlxG.state` and its camera(s) exist — this is where you can safely instantiate `FlxSprite`s, add cameras, etc. Do **not** assume `FlxG.camera` or `FlxG.state` are ready in a state's constructor.
7. Every frame: Lime dispatches an `onUpdate` tied to its render loop → OpenFL ticks its event/timer system → `FlxGame.onEnterFrame`-equivalent (actually driven by `update()` override tied to `openfl.events.Event.ENTER_FRAME` internally) → calls `FlxG.game.update()` → updates `FlxG.state` (and substate if present) → then draws.

### 2.3 Dev vs. release builds

- `lime test <target>` / `-debug` flag: keeps `#if debug` code paths, `FlxG.debugger` overlay enabled, asserts, verbose traces, no minification (html5), slower but with `FlxG.watch`, visual debugger, and better stack traces.
- Release (`-final` or no `-debug`): DCE is more aggressive, `#if debug` blocks are stripped, Flixel's debugger is compiled out (`FLX_NO_DEBUG` define may be set), inlining/optimizations increase — **a bug that only appears in release is often caused by code that relied on something only true when debug asserts/traces kept a value alive, or by DCE removing an unused-looking but reflectively-referenced class/field.**
- Common relevant defines: `debug`, `release`, `final`, `FLX_NO_DEBUG`, `FLX_NO_SOUND_SYSTEM`, `FLX_NO_SOUND_TRAY`, `FLX_NO_FOCUS_LOST_SCREEN`, `FLX_RECORD`, `openfl_html5`, `html5`, `air`, `native`. Always check the project's `<haxedef>`/`-D` flags before assuming a feature is compiled in.

---

## 3. Haxe Language Reference (Agent-Focused)

### 3.1 Type system essentials
- Haxe is **statically typed with type inference**; unification errors at compile time are your friend — don't paper over them with `cast`/`Dynamic` unless the underlying design genuinely calls for dynamic behavior (e.g., JSON, reflection-based serialization).
- **Structural subtyping** applies to anonymous structures (`{x:Int, y:Int}`), not to classes (classes use nominal typing + interfaces).
- **`Null<T>`**: on statically-typed targets (cpp, hl, java, cs), a bare `Int`/`Float`/`Bool` cannot be `null` unless wrapped `Null<Int>`; on dynamic targets (js, neko, python) this distinction is erased at runtime but still enforced at compile time. Never assume `null` works uniformly across targets for value types.
- **Null safety** (`@:nullSafety` class/field/package metadata, or `-D nullSafety` at a level like `Strict`/`Loose`): if enabled in the project (check `hxml`/`project.xml`/top of files), the compiler will reject unguarded dereferences of `Null<T>`. Respect existing safety level; don't add unsafe patterns to null-safe modules.

### 3.2 Abstracts
- `abstract` types wrap an underlying representation with compile-time-only operator overloading and implicit casts — **zero runtime cost** (they compile to the underlying type, no boxing).
- HaxeFlixel/OpenFL make heavy use of abstracts, e.g. `FlxColor` (abstract over `Int`, provides `.red/.green/.blue`, `FlxColor.RED`, arithmetic, implicit `Int`↔`FlxColor` conversion), `FlxPoint` pooling wrappers in some versions, `EReg`-like typed wrappers.
- When you see code like `sprite.color = 0xFFCC00;` working even though `color` is typed `FlxColor`, that's an abstract's `@:from Int` implicit cast — not dynamic typing.
- Enum abstracts (`enum abstract Direction(Int) { var LEFT; var RIGHT; }`) are used for lightweight, allocation-free enumerations (common for state IDs, direction constants) — treat them like enums for pattern matching, but they compile to primitives.
- Never assume an abstract "is" its underlying type for identity/reflection purposes — `Std.isOfType`/`Type.getClass` behavior on an abstract targets the underlying runtime type, not the abstract wrapper.

### 3.3 Enums vs enum abstracts vs classes-as-constants
- Plain `enum` (algebraic data type) supports **pattern matching** (`switch`) with exhaustiveness checking and can carry constructor parameters (`Some(Int)`), unlike C-style enums. Prefer this for real ADTs (game states, message types with payloads).
- `enum abstract` is for cheap, flat, primitive-backed constants when you don't need parameterized constructors.
- Distinguish both from a class of `static inline var` constants (no type safety grouping, just literal inlining).

### 3.4 Typedefs
- `typedef` creates a name for a structure or type alias — purely compile-time, no runtime footprint. Used heavily for config records (`typedef LevelData = { tiles:Array<Int>, width:Int, height:Int }`) and for narrowing generic library types.
- A typedef to an anonymous structure is structurally typed: any object matching the fields is assignable, regardless of declared type.

### 3.5 Interfaces & generics
- Interfaces (`interface`) can declare fields/methods; classes `implements` them. Flixel uses interfaces like `IFlxDestroyable`, `IFlxBasic` sparingly to allow duck-typed collections (`FlxTypedGroup<T:FlxBasic>`).
- Generics (`class Foo<T>`) are monomorphized per concrete type on statically typed targets (like C++ templates) but represented more dynamically on JS — be aware that heavy generic use can affect code size on `cpp`.
- `FlxTypedGroup<T>` / `FlxGroup` (= `FlxTypedGroup<FlxBasic>`) is the generic pattern used throughout Flixel — prefer typed groups (`FlxTypedGroup<Enemy>`) over the base `FlxGroup` when you need typed iteration without casting.

### 3.6 Metadata & Macros — Investigation Procedure

Metadata (`@name` or `@name(args)`) attached to classes/fields is **inert at the language level** — it does nothing by itself. Its meaning is entirely defined by whatever consumes it: a **build macro** (`@:build`/`@:autoBuild`), a **custom compiler macro** run via `--macro` in an `.hxml`, a **code generator** external to the compiler, or (rarely) runtime reflection (`haxe.rtti.Meta`, `Context.getLocalClass().meta` at macro-time, or `@:meta` in some targets to emit native attributes).

**Given unfamiliar metadata like:**
```haxe
@posX
@formula("uDisplayRotateX(aPos)")
@set("properties")
public var x:Int;
```

**Do this before touching it:**
1. **Grep the entire repo** (not just the file) for the exact metadata name (`@posX`, `@formula`, `@set`) — both as literal text and, if it might be dynamically built, partial matches (`"formula"`, `"posX"`).
2. Look specifically for:
   - `@:build(...)` or `@:autoBuild(...)` macro classes referencing these names via `Context.getLocalClass().get().meta.extract("formula")` or similar (`haxe.macro.Context`, `ExprTools`, `MetaAccess`).
   - Any `.hxml` `--macro` invocation or `--macro include(...)`/`initMacro` that could scan these.
   - Non-Haxe tooling in the repo (Python/Node scripts, custom codegen, a `tools/` folder) that parses `.hx` source as text/AST outside the Haxe compiler entirely (common in engines with a custom property/reflection system, shader-uniform binding generators, or serialization codegen — this pattern strongly resembles a **shader-uniform / property-binding code generator**, e.g. mapping a Haxe field to a GLSL uniform (`uDisplayRotateX`) and a settable property bag (`"properties"`)).
   - Documentation/README mentioning a custom DSL, ECS, or reflection system.
3. If no consumer is found anywhere in the repo, it may be:
   - Dead/vestigial metadata from a removed tool — confirm via git blame/history before removing.
   - Consumed by an **external build step** not present in this checkout (a separate codegen repo, IDE plugin, or CI pipeline) — flag this explicitly rather than assuming it's safe to delete.
4. **Never** delete, rename, or "simplify" such metadata as a drive-by cleanup. If asked to modify the field, preserve all metadata verbatim unless the task specifically concerns it, and explain to the user what you found (or didn't find) about its consumer.
5. If asked to *add* similar metadata to a new field, first read at least one full macro/codegen consumer implementation to replicate the exact expected argument shape/types (string literal vs identifier vs expression) — mismatches here fail silently or throw only at codegen/macro time, not at normal type-check time.

**General macro literacy:**
- `@:build(pack.Macro.build())` runs at compile time per class, can add/modify fields — expect generated fields/methods that don't appear in the source file itself. When a field seems to be used but not defined anywhere, check for `@:build`/`@:autoBuild` on the class or its parent.
- `@:generic`, `@:analyzer`, `@:nullSafety`, `@:structInit`, `@:forward`, `@:transitive`, `@:enum` (legacy), `@:native`, `@:keep`, `@:noCompletion`, `@:allow` are common **compiler-recognized** metadata (not user macros) — know these vs. project-custom metadata.
  - `@:keep` / `@:keepSub` prevents DCE from stripping something reflection/macro-only code depends on — if removing "unused" code, check for `@:keep` first, and check whether a class is only reached via `Type.createInstance`/reflection (which DCE cannot always see).
  - `@:allow(pkg.Class)` grants private access across modules — expect cross-file coupling.
  - `@:forward` on abstracts/typedefs forwards underlying fields — explains why an abstract "has" methods not declared on it directly.
- Macro-time code (inside `#if macro` / `macro` functions / files under a `macro` scope) runs **during compilation on the Haxe compiler's own runtime (Neko/JIT)**, not on the target platform — it cannot use target-specific APIs and executes only once at build time, not per-frame.

### 3.7 Conditional compilation
- `#if target`, `#if debug`, `#if (flixel >= "5.0.0")` (library version conditionals via `haxelib.json`'s `version` — actually via `#if (haxe_ver >= ...)` or library-specific defines), `#elseif`, `#else`, `#end`.
- Common target defines: `js`, `html5`, `cpp`, `hl`, `neko`, `python`, `java`, `cs`, `swf`, `flash`, `android`, `ios`, `mac`, `windows`, `linux`, `desktop` (Lime convenience define combining windows/mac/linux), `mobile` (Lime convenience combining android/ios).
- Flixel-specific: `FLX_NO_DEBUG`, `FLX_NO_SOUND_SYSTEM`, `FLX_NO_SOUND_TRAY`, `FLX_NO_NATIVE_CURSOR`, `FLX_NO_MOUSE`, `FLX_NO_KEYBOARD`, `FLX_NO_TOUCH`, `FLX_NO_GAMEPAD`, `FLX_POINTER_INPUT`, `FLX_UNSAFE` (disables some bounds checks) — check `project.xml`/hxml for which are set; code guarded by these may be silently absent on some target configs.
- When editing code inside `#if`/`#end` blocks, ensure you're editing the branch that's actually active for the target(s) the user cares about — check the active defines rather than assuming.

### 3.8 Dead Code Elimination (DCE)
- Default DCE mode is `std` (eliminates unused std lib + your code not reachable from `Main`/entry points); `full` is stricter; `no` disables it.
- Classes/fields only referenced via `Type.resolveClass`, `Type.createInstance(name)`, JSON-driven factory patterns, or macro-injected calls **may be eliminated** unless marked `@:keep`, referenced in an explicit `--macro include('pkg')`, or listed in an `<icon>`/reflect config. If a "seemingly working" reflective factory breaks only in release/optimized builds, DCE stripping the target class is a prime suspect — verify by checking for `@:keep`/explicit `include` and confirming class registration.

### 3.9 Package/module conventions
- One `.hx` file per top-level type by convention (though multiple types are legal in one file); package path must match folder path (`src/entities/Player.hx` → `package entities;`).
- `import` order/grouping conventions vary by project — match existing style. Watch for `import openfl.Assets;` vs `flixel.system.FlxAssets` vs `openfl.utils.Assets` — these are **different classes** despite similar names; verify which one a project actually uses before adding new asset-loading code (mixing them can cause double-loading or cache-key mismatches, see §5.7 and §11).

---

## 4. HaxeFlixel Architecture Reference

*(APIs described are for the Flixel 5.x line; if the project pins Flixel 4.x, verify differences — notably: `FlxSprite.antialiasing` defaults, `FlxG.state.subState` API stability, `FlxTilemap` vs `FlxTilemapExt`, and camera filter API changed between major versions.)*

### 4.1 Core class hierarchy
```
FlxBasic (id, active, exists, alive, camera list, update()/draw() virtuals, destroy())
 └─ FlxObject (x, y, width, height, velocity, acceleration, drag, maxVelocity,
               angle, moves, immovable, solid/allowCollisions, physics integration)
     └─ FlxSprite (graphic/BitmapData, animation, scale, offset, origin, color, alpha,
                   blend, pixel-perfect render, FlxFrame/atlas support)
         └─ FlxText, FlxTileblock, FlxButton (flixel-ui/addons), FlxBackdrop (addons), etc.
     └─ FlxTilemap (tile-based collision + render, extends FlxObject, not FlxSprite)
 └─ FlxGroup / FlxTypedGroup<T> (container of FlxBasic, forwards update/draw to members)
     └─ FlxState extends FlxGroup effectively-composed (actually FlxState extends FlxUIState? no —
        FlxState extends FlxGroup via composition through `members`; check exact version)
     └─ FlxSpriteGroup (groups sprites, exposes aggregate x/y/alpha that propagate to children)
 └─ FlxSubState extends FlxState (modal overlay state; pauses/keeps parent state visible)
```
`FlxState` and `FlxSubState` are **not** `openfl.display.DisplayObject`s in the traditional sense for child management — Flixel maintains its own scene-graph-like `members` array and calls `update`/`draw` manually each frame; rendering ultimately still goes through OpenFL's `Graphics`/tilesheet batching under the hood (`FlxDrawQuadsItem`/`FlxDrawTrianglesItem` on the OpenGL-backed renderer, or `openfl.display.Tilesheet`/`Graphics.drawTiles` compatibility path on other renderers).

### 4.2 Game loop / lifecycle order per frame
1. `FlxGame.onEnterFrame` (driven by OpenFL's stage `ENTER_FRAME`) computes elapsed time, applies `FlxG.timeScale`/`FlxG.updateFramerate` throttling.
2. `FlxG.game.update()`:
   - Updates `FlxG.sound`, input managers (`FlxG.keys`, `FlxG.mouse`, `FlxG.touches`, `FlxG.gamepads`).
   - Calls `FlxG.state.tryUpdate(elapsed)` → if a `subState` is active, typically **only the substate updates** (parent state paused) unless `persistentUpdate = true` on the substate.
   - Within a state/group: `update(elapsed)` iterates `members`, calling each object's `update(elapsed)` — order in the `members` array **is** update order and (usually) draw order; be mindful of insertion order for correct layering and for one object's update depending on another's already-updated state this frame.
   - `FlxObject.update()` applies physics integration (`velocity += acceleration * elapsed`, drag, then `position += velocity * elapsed`), then calls `updateMotion`... then collision **is not automatic** — you must explicitly call `FlxG.collide`/`FlxG.overlap` (commonly in the state's `update`, not inside object `update`) each frame.
3. Camera update (scroll, follow lerp, shake, flash/fade effects, zoom) happens as part of `FlxG.cameras` update.
4. Draw pass: `FlxG.state.draw()` → each `FlxBasic`'s `draw()` iterates its **camera list** (`getCameras()`), rendering to each camera it's visible on. Sprites not added to any camera default to `FlxG.cameras.list` (all active cameras).

### 4.3 States and SubStates
- Switch state: `FlxG.switchState(new PlayState())` — destroys the old state (calls `destroy()` on it and all its members) after transition effect (if `FlxTransitionableState`/addons transition plugin used) completes. **Never hold references to the old state's objects across a switch** without nulling them — they will be destroyed and their internal resources (bitmap data references, tweens, timers) freed/invalidated.
- `openSubState(new PauseSubState())` / `closeSubState()`: keeps the parent state alive underneath; parent's `update()` is skipped by default (`persistentUpdate = false`) but its `draw()` still runs by default (`persistentDraw = true`) so it remains visible behind the substate.
- `create()` lifecycle hook runs once, after the object/state is fully constructed and (for states) attached — this is where you build the scene graph, not in the constructor (camera/`FlxG.state` references may not be ready in the constructor).
- `destroy()` is where you must null out/`destroy()` any manually-held references (tweens not attached to an auto-destroyed object, custom listeners, `FlxTypedGroup`s not added as members) to avoid leaks — Flixel calls `destroy()` recursively on `members` of groups/states automatically, but only for objects actually added via `add()`.

### 4.4 FlxBasic contract: `active`, `visible`, `exists`, `alive`
- `exists` gates both update **and** draw and group membership checks (`FlxG.overlap` skips non-existing objects) — setting `exists = false` is how pooling/"kill" without destroy works.
- `alive` is a finer flag (mainly meaningful for `FlxObject`/`kill()`/`revive()` used with `FlxTypedGroup.recycle()` object pools) — `kill()` sets `alive=false` and typically `exists=false`; `revive()` resets both plus health if applicable.
- `active` gates only `update()`; `visible` gates only `draw()`. Distinguish all four when debugging "object won't update/draw/collide" — check the right flag rather than guessing.

### 4.5 Groups & recycling
- `FlxTypedGroup<T>.recycle(Class<T>, ?factory, ?force, ?revive)` is Flixel's built-in object pool — reuses a dead (`exists=false`) member instead of allocating, calling the factory only if none available. Prefer this over manual `new T()` in hot spawn loops (bullets, particles, enemies) to reduce GC pressure, especially on GC-sensitive targets (hxcpp, hl).
- `add()`/`remove()`/`insert()` manage membership; removing from a group does **not** call `destroy()` — you must call it explicitly if the object should be freed, otherwise it just becomes an orphaned, exists=true instance sitting in memory doing nothing until GC'd normally (a common leak source only if something else still references it, e.g. an event listener).

### 4.6 Cameras (`FlxCamera`)
- `FlxG.camera` = default camera; `FlxG.cameras.add(cam)` registers additional ones; `FlxG.cameras.reset(cam)` replaces the default set.
- Each `FlxBasic` tracks which cameras it draws to (`sprite.cameras = [camA]`); default is `null` meaning "all `FlxG.cameras.list`" at draw time (dynamic, not a snapshot) — assigning an explicit array **freezes** which cameras it's tied to even if more are added later. This is a common bug: newly added HUD camera doesn't show a sprite because the sprite's `.cameras` was explicitly set earlier to `[gameCamera]`.
- Camera effects: `.follow(target, style, lerp)`, `.fade()`, `.flash()`, `.shake()`, `.zoom`, `.scroll` (an `FlxPoint`), `.bgColor`, `.setFilters()` (OpenFL `BitmapFilter` list — GPU cost warning, see §9).
- `FlxCamera.pixelPerfectRender` / global `FlxG.renderBlit` vs `FlxG.renderTile` (legacy Flixel 4 distinction; in Flixel 5, tile/hardware rendering is essentially the standard path — check version before assuming `renderBlit` exists).

### 4.7 Sprites, Spritesheets, Animations
- `FlxSprite.loadGraphic(path, animated, width, height)` for simple sheets; `loadGraphicFromSprite`, `makeGraphic()` for procedural placeholder textures.
- Atlas-based: `FlxAtlasFrames` from `TexturePacker` (`FlxAtlasFrames.fromTexturePackerJson/Xml`), Sparrow (`fromSparrow`), Aseprite export, or `FlxAtlasFrames.fromTileSheet` — assign to `sprite.frames`.
- `sprite.animation.add(name, frames, frameRate, looped)`, `.play(name, force, reversed, frame)`, `.finishCallback`. Check `looped` default and whether `.play` restarts an already-playing animation (it doesn't, unless `force=true`) — a very common "animation doesn't restart" bug.
- `FlxFrame`/`FlxImageFrame` underlie the frame system — direct manipulation is rarely needed; prefer the atlas/animation API.
- `antialiasing`, `pixelPerfectRender`, `flipX`/`flipY`, `scale` (an `FlxPoint`, not a single number — use `.set()` on it or `setGraphicSize()`), `offset` (adjusts hitbox vs. graphic alignment — critical for correct collision with non-tight art).

### 4.8 Tweens (`FlxTween`) and Timers (`FlxTimer`)
- `FlxTween.tween(obj, {x: 100, y: 50}, duration, {ease: FlxEase.quadOut, onComplete: fn, type: FlxTween.ONESHOT})` — tweens are managed by a global `FlxTween` manager tied to `FlxG.state` by default (`FlxTween` instances get auto-destroyed on state switch **unless** created with `TweenManager` context tied elsewhere) — don't assume a tween survives a state switch.
- `FlxTimer` (`new FlxTimer().start(seconds, callback, loops)`): also tied to the global timer manager; `loops = 0` means infinite. Cancel with `.cancel()`, not just dereferencing, if the owning object is destroyed early — otherwise the callback may fire against a destroyed/stale object (classic dangling-callback bug — always guard callbacks with an `exists`/`alive` check or cancel explicitly in `destroy()`).
- Both support pooling internally; avoid manually caching/reusing `FlxTween`/`FlxTimer` instances across unrelated logical uses.

### 4.9 Input handling
- Keyboard: `FlxG.keys.pressed.SPACE` (held), `.justPressed.SPACE` (this frame only), `.justReleased.SPACE`. Prefer `justPressed` for discrete actions (jump, menu confirm) and `pressed` for continuous (move).
- Mouse: `FlxG.mouse.x/y` (world coords relative to a camera — use `FlxG.mouse.getScreenPosition()`/`getWorldPosition(camera)` for precision with multiple cameras/zoom), `.justPressed`, `.pressed`, `.wheel`.
- Touch: `FlxG.touches.list`, each `FlxTouch` similar API to mouse.
- Gamepad: `FlxG.gamepads.firstActive`, `.pressed.A`, analog `getAnalogAxis`.
- Action-map style input (multiple bindings → one logical action) is often custom in the project — check for an `Input.hx`/`Controls.hx` wrapper before adding raw `FlxG.keys` checks scattered through gameplay code.

### 4.10 Collision & Physics
- `FlxG.collide(objectOrGroupA, objectOrGroupB, ?notifyCallback)` and `FlxG.overlap(...)` are **the** primary APIs — both use `FlxObject.separate` internally for collide (resolves overlap physically) vs overlap (detection only, no resolution).
- Both accept a `FlxSpatialQuadTree` broad phase internally (`FlxQuadTree`) — performance implication: calling `FlxG.collide` many times per frame with large groups is O(n log n)-ish via quadtree rebuild each call; batching all collidable groups into fewer `collide` calls per frame (or a single `collide(allSolids, allSolids)`), when correctness allows, reduces quadtree rebuild overhead. Verify actual usage pattern before assuming this is the bottleneck (profile first).
- `allowCollisions` (`FlxObject.ANY`, `.NONE`, `.LEFT/RIGHT/UP/DOWN`, `.WALLS`, `.FLOOR`) and `immovable` control resolution direction/behavior — a `!immovable` object colliding with another `!immovable` object splits the separation between both; platformers almost always set terrain/platforms `immovable = true`.
- `FlxTilemap` collision uses per-tile `allowCollisions` (auto-set via `setTileProperties` or the tileset's collision range in `loadMapFromArray`/`loadMapFromCSV`) — much cheaper than per-tile `FlxSprite` objects for large levels.
- Ray casting: `FlxObject.separate`, or addons' `FlxVelocity`/`FlxAngle` helpers for aimed movement (`velocityFromAngle`, `moveTowardPoint`).
- This is **not a full physics engine** (no rotational torque, no complex polygon collision) — if the project needs that, it likely integrates a separate library (e.g. `nape`, `box2d` bindings) — check imports before assuming Flixel's built-in AABB system applies.

### 4.11 Tilemaps
- `FlxTilemap.loadMapFromCSV`/`loadMapFromArray`/`loadMapFrom2DArray` + a tileset image, tile width/height, and a `FlxTilemapAutoTiling` mode (`OFF`, `AUTO`, `ALT`) for auto-tiling bitmasking.
- Ogmo/Tiled integration is typically via `flixel-addons` (`FlxOgmo3Loader`) or a custom loader — check for one before writing a raw CSV parser.
- Rendering: tilemaps are drawn efficiently as a batched tile draw call (hardware-accelerated path) rather than per-tile sprites — do not replace a `FlxTilemap` with hundreds of individual `FlxSprite`s for "flexibility" without acknowledging the draw-call cost tradeoff.

### 4.12 Particles
- `FlxEmitter` (extends `FlxTypedGroup<FlxParticle>`) — `emitter.loadParticles(graphic)`, `.start(explode, frequency, quantity)`, configure `velocity`, `acceleration`, `alpha`, `scale`, `color` **ranges** (`FlxRange`/`.set(min,max)` on relevant range fields) for randomized variation.
- Particles are pooled automatically via the group's recycle mechanism — avoid manually `new FlxParticle()`-ing outside the emitter's factory unless you know why.
- `flixel-addons` provides more advanced particle/FX helpers (e.g. `FlxTrail`, `FlxSkewedSprite`, weather effects) — check before hand-rolling.

### 4.13 Text
- `FlxText` wraps OpenFL `TextField` for bitmap-cached rendering (`FlxText` re-rasterizes to a bitmap on change — mutating `.text` every frame is comparatively expensive; avoid per-frame `.text =` updates for large/frequently-updated text, batch/throttle updates instead, e.g., only update a score display when the score actually changes).
- `FlxBitmapText` is a lighter-weight bitmap-font-based alternative for performance-sensitive text (HUDs updated often, large amounts of text) since it avoids `TextField` rasterization overhead per change (still has a cost, but generally cheaper and more consistent across targets, notably html5 where `TextField`/font rendering can be inconsistent).

### 4.14 Shaders
- Flixel (5.x) sprites/cameras support OpenFL's `openfl.filters.ShaderFilter` / directly assigning `sprite.shader = new MyShader()` where `MyShader extends flixel.system.FlxShader` (which extends `openfl.display.Shader`, adding Flixel-specific uniforms like `openfl_Matrix`, `openfl_TextureCoordv`, etc., auto-populated by the renderer).
- Shaders are written in GLSL (via OpenFL's `@:glsl`-embedded string or `.glsl`/`.frag`/`.vert` files loaded through macros/build tooling — check `Assets.hx`/`project.xml` for shader asset declarations).
- Camera-wide shader/filter (`camera.setFilters([new ShaderFilter(shader)])`) applies post-processing to everything drawn on that camera — vs. per-sprite `shader` which only affects that sprite's draw. Performance-sensitive: filters typically force a render-to-texture pass; excessive per-object filters can create many extra draw calls/FBO switches, especially costly on mobile/HTML5 (WebGL) — verify actual GPU cost via the debugger overlay/profiler rather than assuming.
- **Not all targets support the same shader capability** — legacy `flash`/`air` targets have limited/no custom shader support in this stack (Flash's Pixel Bender vs GLSL are incompatible); if the project targets Flash/AIR (rare in modern Flixel projects, but check), confirm shader code has a fallback or is conditionally compiled out.

### 4.15 Audio
- `FlxG.sound.play(asset, volume, looped, ?group, ?autoDestroy, ?onComplete)`; `FlxG.sound.music` for a dedicated persistent music channel (`.fadeIn`/`.fadeOut`); `FlxG.sound.playMusic`.
- Under the hood, wraps `openfl.media.Sound`/`SoundChannel` — target-specific decode support differs (e.g., not all targets support all codecs identically — `ogg` widely supported on native targets, `mp3`/`wav` more universal; html5 depends on browser codec support). Check what audio formats are actually bundled and whether target-specific asset variants exist (`<assets path="..." if="html5" />`-style conditional asset declarations in `project.xml`).
- Volume is global-multiplied: `FlxG.sound.volume` scales all sound; per-channel `.volume` is independent — a "silent audio" bug should check both.

### 4.16 Save system
- `FlxSave` wraps `openfl.net.SharedObject` (a per-platform persistent key-value store — browser localStorage-like on html5, native file-backed on desktop/mobile). `save.bind("saveName", "companyName")` then read/write via dynamic fields (`save.data.highscore = 100; save.flush();`).
- Because `SharedObject`/`FlxSave` stores essentially arbitrary serialized Haxe data (via its own serialization), **changing the shape of saved data structures across versions can break old saves** (missing fields become `null`, requiring defensive `if (save.data.foo == null) save.data.foo = default;` migration code) — when modifying save-related fields, always consider backward compatibility with existing save files rather than assuming a clean slate.

### 4.17 Debugging tools
- Built-in visual debugger (`F2` or debug-key toggle by default, `FlxG.debugger.visible`) — bounding box overlay (`FlxG.debugger.drawDebug`/`FlxObject.ignoreDrawDebug` per-object), watch window (`FlxG.watch.addQuick("label", value)` / `.add(object, "field")`), console (`FlxG.console` — register custom commands via `FlxG.console.registerFunction`), stats overlay (FPS, memory, draw calls).
- Only compiled in for `debug` builds by default (unless `FLX_NO_DEBUG` forces it off, or the project explicitly force-enables in release — rare). Don't expect debugger APIs to be present/functional in a release/final build; guard any debug-only helper code with `#if debug` or check the project's convention.
- `FlxG.log`/`FlxG.watch` vs plain `trace()` — prefer the Flixel-native ones when the intent is visible in the in-game debugger overlay; use `trace()`/`Sys.println` (native) or browser console (html5) for pure text logging during development.

### 4.18 Performance-relevant Flixel specifics
- `FlxG.updateFramerate` vs `FlxG.drawFramerate` are decoupled — you can update logic more/less often than you render; verify which one the project tunes when investigating "feels laggy" vs "visually stutters" reports.
- `FlxG.autoPause` (pauses on focus loss) can hide bugs that only reproduce with the window focused/unfocused — be aware when debugging timing-sensitive issues.
- Excessive per-frame `new FlxPoint()`/`new FlxRect()` allocation is a classic GC-pressure source; Flixel provides pooled helpers (`FlxPoint.get()`/`.put()`, `FlxRect.get()`/`.put()`) specifically to avoid this — prefer these in hot paths (per-frame update/collision code) over raw `new FlxPoint(...)`, and make sure to `.put()` them back when done.

### 4.19 Ecosystem
- `flixel-addons`: transitions (`FlxTransitionableState`), UI widgets (grid/9-slice `FlxUI9SliceSprite`), `FlxOgmo3Loader`, `FlxSpriteAnimator` helpers, weather/effects, `FlxGridOverlay`, more.
- `flixel-ui`: XML-driven UI system (older, less commonly used in newer projects — check before assuming it's available).
- `flixel-tools`: CLI scaffolding, not a runtime dependency.
- Community forks (e.g., rhythm-game engines built on Flixel) sometimes patch core Flixel classes — if `flixel` is haxelib-git-sourced from a fork, assume **nothing** about stock API compatibility until confirmed against that fork's actual source.

---

## 5. OpenFL Architecture Reference

### 5.1 Application lifecycle
- Entry class generated per target wraps `lime.app.Application`; OpenFL's `openfl.display.Application`/`Lib.current` bridges Lime's `Window` to an OpenFL `Stage`.
- Key events: `Event.ADDED_TO_STAGE`/`REMOVED_FROM_STAGE` on display objects, `Event.RESIZE` on stage, `Event.ENTER_FRAME` (drives per-frame logic for anything not using Flixel's own loop), `Event.ACTIVATE`/`DEACTIVATE` (focus).

### 5.2 Display list
- Classic Flash-style retained-mode tree: `DisplayObjectContainer` (`Sprite`, `MovieClip`—rare in OpenFL usage, `Stage`) holds children (`addChild`/`removeChild`/`getChildAt`); every node has `x/y/scaleX/scaleY/rotation/alpha/visible/blendMode/mask/filters`.
- `DisplayObject` transforms are **local to parent** — global/world coordinates require `localToGlobal`/`globalToLocal`. HaxeFlixel's own coordinate system (`FlxObject.x/y` in "world" space relative to `FlxCamera.scroll`) is **conceptually separate** from raw OpenFL display coordinates — don't mix `sprite.x` (OpenFL) assumptions onto `flxSprite.x` (Flixel) without understanding Flixel manages the actual OpenFL-level draw position internally via its renderer, not a 1:1 direct `DisplayObject` per `FlxSprite` in the tile-rendering path (Flixel batches many sprites into few draw calls rather than one `DisplayObject` each — this is a major architectural difference from naive OpenFL `Sprite`-per-entity approaches).
- `Bitmap` (a `DisplayObject` showing a `BitmapData`) vs `BitmapData` (the actual pixel buffer, GPU-uploadable, supports `draw()`, `getPixels`/`setPixels`, `copyPixels`, `applyFilter`, `lock`/`unlock` for batched pixel ops) — mutating `BitmapData` per-pixel every frame is CPU-bound and slow at scale; prefer shaders/blend modes for effects achievable on GPU.
- `Shape`/`Graphics` (`graphics.beginFill`, `.drawRect`, `.lineTo`, etc.) for vector drawing — each redraw (`graphics.clear()` + redraw) has CPU cost; cache to `BitmapData` (`Bitmap.bitmapData = shape's rendered result via BitmapData.draw(shape)`) if drawn once and reused many frames.
- `TextField` — full Flash-compatible text engine (`defaultTextFormat`, `htmlText`, embedded font support via `openfl.text.Font`), heavier than `FlxBitmapText`; Flixel's `FlxText` wraps this.

### 5.3 Events
- OpenFL uses the Flash `EventDispatcher` model: `addEventListener(Type, handler, useCapture, priority, useWeakReference)`, `removeEventListener`, `dispatchEvent`. **Weak references** (`useWeakReference = true`) allow GC to collect a listener's owner even if still registered — relevant on targets that support it (not all targets honor weak refs identically; don't rely on weak references as your only cleanup strategy — always `removeEventListener` explicitly in `destroy()`/teardown to avoid leaks and stale-callback bugs, which is the more portable/reliable pattern).
- Common: `MouseEvent` (`CLICK`, `MOUSE_DOWN/UP/MOVE`, `MOUSE_WHEEL`), `KeyboardEvent` (`KEY_DOWN/UP`, `.keyCode`, `.charCode`), `TouchEvent`, `Event.RESIZE`, `Event.ENTER_FRAME`, `IOErrorEvent`/`Event.COMPLETE` for asset/network loading.
- HaxeFlixel wraps keyboard/mouse/touch input into its own `FlxG.keys`/`FlxG.mouse`/`FlxG.touches` managers that internally subscribe to these OpenFL events — prefer Flixel's wrappers in gameplay code (they handle `justPressed`/frame-accurate state), and reserve raw OpenFL event listeners for UI/system-level concerns (e.g., custom `Stage`-level resize handling, non-game overlay UI) or when you need an event Flixel doesn't expose.

### 5.4 Rendering backends
- OpenFL selects a renderer per target: hardware `GL`/OpenGL(ES)/WebGL context (`opengl`/`webgl` render surface — most native + html5 default), or a `Cairo`/`Canvas`(2D)/software fallback in specific configurations. HaxeFlixel's tile-batching renderer specifically targets the hardware/GL path for its performance benefits (`FlxDrawQuadsItem`/`FlxDrawTrianglesItem`) — a "software" or `-Dcanvas` html5 build changes performance characteristics substantially (falls back to slower `drawTiles`/canvas 2D compositing) — check `project.xml` `<window renderer="..."/>` / `-Dcanvas`/`-Dopengl` defines when diagnosing html5 performance differences from native.
- Draw call batching: Flixel/OpenFL try to batch consecutive draws sharing the same texture/blend mode/shader into one GPU draw call; interleaving different textures, blend modes, or per-object shaders breaks batching and increases draw calls — a key optimization lever (see §9).

### 5.5 Shaders (OpenFL layer)
- `openfl.display.Shader` base class; custom shaders subclass it and declare `@:glsl` fragment/vertex source as class fields (via a special string syntax processed by OpenFL's build macro) — variables tagged as OpenFL shader parameters (`@input`, uniform declarations recognized by the macro) automatically get typed Haxe properties generated (this is itself a macro/codegen system similar in spirit to the metadata example in §12 — investigate `openfl.display.Shader`'s macro if you need to understand exactly how a `.glsl` uniform maps to a Haxe field name).
- Filters (`openfl.filters.BitmapFilter` subclasses: `BlurFilter`, `GlowFilter`, `ShaderFilter`, `ColorMatrixFilter`) apply to a `DisplayObject`'s rendered output — cost scales with the object's rendered pixel area and filter complexity; applied per-frame on frequently-changing content forces re-render of the filter each frame (no caching benefit).

### 5.6 Assets
- **openfl.utils.Assets** (`Assets.getBitmapData(id)`, `Assets.getSound(id)`, `Assets.getFont(id)`, `Assets.getText(id)`) reads from a **manifest** built at compile time from `<assets path="..." />` entries in `project.xml`, with an `id` derived from path (or explicit `id="..."` attribute) — assets are referenced by **string ID**, which is fragile to typos (no compile-time checking) unless the project uses a generated typed-asset-paths class.
- Many Flixel projects use **flixel-tools' asset class generation** or a similar codegen step (`AssetPaths.hx` auto-generated listing every asset file as a typed static string constant) to get compile-time-checked asset references — check whether such a generated file exists (often in `source/` and marked "auto-generated, do not edit") before hand-writing raw string asset paths; if present, prefer it for new code to match convention and catch typos at compile time.
- Embedded (`embed="true"`) vs. streamed/loaded assets: embedding bakes assets directly into the executable/JS bundle (larger binary, available immediately, no load latency); non-embedded assets on html5 are fetched asynchronously and require the preloader (or manual `Assets.loadBitmapData` with a callback/`Future`) to complete before use — a `null` `BitmapData` or blank sprite on html5 (but fine on cpp, where assets are often embedded or loaded synchronously from disk) is a classic symptom of not waiting for async asset load.
- `Assets.getBitmapData` vs Flixel's `FlxG.bitmap.add(...)`/`FlxGraphic` cache — Flixel maintains its **own** bitmap cache (`FlxG.bitmap`) keyed by asset key on top of OpenFL's, to dedupe `FlxGraphic` wrapper objects (important for shared-sheet sprites) and manage its own cache-clearing on state switches (`FlxG.bitmap.clearCache()` or automatic based on `FlxG.stage.frameRate`/persistent tracking) — bypassing Flixel's cache by calling raw OpenFL `Assets.getBitmapData` directly for a `FlxSprite`'s graphic can lead to duplicate `BitmapData` copies in memory or cache invalidation mismatches; prefer `FlxSprite.loadGraphic`/Flixel's own asset helpers unless there's a specific reason (e.g., non-Flixel OpenFL `Bitmap` display objects mixed into the tree).

### 5.7 Audio
- `openfl.media.Sound`/`SoundChannel`/`SoundTransform` — lower-level than `FlxG.sound`; Flixel builds on this. Only reach for raw OpenFL audio APIs if you need something Flixel's sound system doesn't expose (e.g., raw PCM streaming, custom channel graphs), and understand you'll then be bypassing Flixel's volume-scaling/pause/focus-loss integration (must handle that yourself for consistency).

### 5.8 Networking
- `openfl.net.URLLoader`/`URLRequest` for HTTP(S) fetches (text/binary/variables), event-driven (`Event.COMPLETE`, `IOErrorEvent.IO_ERROR`, `ProgressEvent.PROGRESS`). No built-in Promise/async-await sugar (Haxe has no native `async`/`await`; some libraries add `haxe.EntryPoint`/`Future`/`Promise` wrappers via `lime.app.Future` — check for those before hand-rolling callback chains).
- Target-specific constraints: browser (html5) networking is subject to CORS; native targets may need explicit socket/HTTPS library support depending on Lime version and platform (e.g., Android network permissions declared in `project.xml`'s `<config:android>`).

### 5.9 Stage & window configuration
- `project.xml` `<window width="" height="" fps="" background="" resizable="" orientation="" />`, `<meta>` app metadata, per-platform `<config:android>`/`<config:ios>` blocks for permissions/orientation/icons.
- `Stage.scaleMode`/`Stage.align` (Flash-legacy Stage scaling concepts) interplay with Flixel's own scale-mode system (`FlxG.scaleMode` — `RatioScaleMode`, `FillScaleMode`, `RelativeScaleMode`) for handling window resize vs. fixed game resolution (`FlxG.initialWidth/Height` vs actual window size) — don't fight both systems at once; Flixel's `scaleMode` is almost always what a Flixel game should configure, not raw `Stage.scaleMode`.

### 5.10 Platform-specific APIs
- Access native platform features (e.g., Android intents, iOS-specific APIs, file system access) typically through Lime's `lime.system.System`, extension libraries (`extension-*` haxelib packages providing CFFI/JNI bridges), or platform-conditional code (`#if android ... #end`). Verify such extensions are actually included as dependencies before assuming an API is available at runtime — many platform extension APIs compile fine but no-op or throw on unsupported targets if the corresponding native extension isn't linked.

---

## 6. Interaction Deep-Dive: Practical Consequences

- **Generated code is real code** — if a bug seems to originate in a class you never wrote (`ApplicationMain`, `__ASSET__..._hx`, manifest classes under `export/<target>/haxe/`), regenerate rather than hand-edit generated output; hand-edits are lost on next build and hint the actual fix belongs in `project.xml`/asset declarations/Lime version.
- **Dependency resolution**: classic `haxelib` resolves versions from each library's `haxelib.json` and whatever's installed globally/locally (`haxelib list`, `.haxelib` folder or `haxelib set`); `lix`-based projects pin exact versions in `haxe_libraries/*.hxml` and `.haxerc` (Haxe compiler version itself). Mixing assumptions from one system when the project uses the other leads to wrong version guesses — check for `haxe_libraries/`/`.haxerc` (lix) vs a plain `.haxelib/` folder (classic haxelib) first.
- **Asset embedding differences across targets** can make a bug target-specific: e.g., case-sensitive asset paths fail only on Linux/html5 (case-sensitive filesystems/URLs) but "work" on Windows/macOS (case-insensitive by default) — a path-casing bug is a classic cross-platform gotcha to check for specifically when a report says "works on my Windows machine, broken on the Linux CI/build".
- **Compilation order for multi-target logic**: if the project maintains parallel target-specific implementations (`#if cpp ... #elseif js ... #end` or separate classes selected via typedef aliasing, e.g. `typedef PlatformImpl = #if cpp NativeImpl #else WebImpl #end;`), changes must be mirrored across all active branches relevant to the supported target list — check which targets are actually shipped (README/CI matrix) before deciding a branch is "dead."

---

## 7. Target-Specific Considerations Checklist

| Target | Watch for |
|---|---|
| **html5** | Async asset loading/preloader completion; WebGL vs Canvas renderer mode; browser codec support for audio; CORS for network; JS number semantics (no true 64-bit ints without `Int64`); file-path case sensitivity; larger initial load = bundle size sensitivity (embedded vs streamed assets); `window.requestAnimationFrame` timing differences vs native frame pacing. |
| **Windows/Linux/macOS (cpp via hxcpp)** | Native GC characteristics (hxcpp's own GC, different pause/alloc profile than JS); file system access is synchronous/direct; case-sensitive paths on Linux; DLL/so dependency packaging for native extensions; slower iteration (full native compile) vs `hl`/`neko` for dev loops. |
| **Android/iOS** | App lifecycle interruptions (background/foreground, `Event.DEACTIVATE`/`ACTIVATE`, `FlxG.autoPause`); touch-first input (mouse emulation caveats); permissions declared in `project.xml`; texture memory constraints (mobile GPUs, atlas sizing); orientation/safe-area handling; app store packaging steps outside Haxe entirely (Gradle/Xcode project settings generated by Lime, sometimes need manual native project tweaks). |
| **Neko/HashLink (dev targets)** | Used for fast iteration (`lime test neko`/`hl`), not typically shipped for html5-class visual fidelity checks — verify a "works in neko" claim doesn't mask a cpp/html5-only bug (e.g., int overflow behavior, GC timing differences). |

---

## 8. Debugging Methodology

Follow this order; do not jump straight to guessing code fixes.

1. **Reproduce and characterize**: Which target(s)? Debug or release build? Deterministic or intermittent? First-frame or after N frames/state switches?
2. **Check compilation layer**: Any compiler warnings/errors being ignored? Are DCE/`-D` defines consistent with expected code paths being included? Try `-verbose` or checking the generated `.hxml`/manifest if asset- or macro-related.
3. **Check initialization/order-of-operations**: Is the failing code assuming an object/asset/camera/state exists before the lifecycle guarantees it (constructor vs `create()`; before vs after preloader completion; before vs after `addChild`/`ADDED_TO_STAGE`)?
4. **Check framework lifecycle semantics**: `exists`/`active`/`visible`/`alive` flags; state vs substate update/draw persistence flags; group membership and destroy() ordering; camera assignment freezing (§4.6).
5. **Check runtime data**: Add targeted `trace()`/`FlxG.watch` at the narrowest possible point; verify actual values, not assumed ones (e.g., confirm an asset actually loaded rather than assuming `Assets.getBitmapData` never returns null).
6. **Check rendering last** (after confirming logic/state is correct): verify camera assignment, draw order (`members` array order/`add()` sequence), blend modes, alpha/visibility, shader/filter application, and whether the render backend (GL/canvas) matches expectations for the target.
7. **Cross-check target-specificity**: if it only reproduces on one target, consult §7 for that target's known gotchas before hypothesizing a general logic bug.
8. **Only then modify code** — make the smallest change that addresses the *root cause* identified above, not the symptom. State explicitly what evidence supports the diagnosis.

---

## 9. Performance Optimization Methodology

**Always profile/measure before optimizing.** Use Flixel's built-in stats overlay (draw calls, FPS, memory) and target-appropriate native profilers (browser devtools for html5, platform profilers for native) rather than assuming a bottleneck.

Classify the bottleneck before choosing a fix:

- **CPU (game logic)**: heavy per-frame math/branching across many objects (large `update()` loops), inefficient collision setup (many small `FlxG.collide` calls vs. batched), string operations/allocations in hot paths, per-frame `FlxText` text reassignment (§4.13), reflection (`Reflect.*`, `Type.*`) in per-frame code.
- **GPU (fill-rate/shader)**: oversized transparent overdraw, expensive per-pixel shaders on large surfaces, filters/blur applied every frame on large or many objects, non-power-of-two/huge textures forcing extra GPU work or memory bandwidth.
- **Allocation/GC pressure**: `new FlxPoint()`/`new FlxRect()`/anonymous structures/closures allocated every frame instead of pooled/reused (§4.18); excessive small object churn in particle/bullet systems not using `recycle()`.
- **Draw calls**: broken batching from interleaved textures/blend modes/per-sprite shaders (§5.4); excessive number of independent `FlxCamera`s each requiring separate render passes; unnecessary use of individual `DisplayObject`s alongside Flixel's batched sprite system.
- **Asset loading**: large synchronous loads blocking startup (especially html5 without proper preloader use); redundant asset decoding due to bypassing Flixel's `FlxGraphic`/bitmap cache (§5.6) causing duplicate textures in memory.
- **Framework-level**: unnecessary substates/state complexity causing redundant updates; `FlxG.updateFramerate`/`drawFramerate` mismatches; debug-only overlays/logging accidentally left enabled affecting perceived performance during testing (verify build is actually release before drawing conclusions from a debug-build profile).

Report findings as: **measured symptom → suspected layer (from the above) → specific verified cause → minimal fix**, not "this looks slow, let me rewrite it."

---

## 10. Coding Conventions & Safe Modification Rules

- Match existing indentation, brace style, naming (camelCase methods/vars, PascalCase types, and whatever prefix/suffix conventions already exist for private fields, signals, constants).
- Preserve existing null-safety strictness; don't introduce `Null<T>` fields into null-safe modules without proper guarding, and don't strip existing null checks.
- Preserve existing use (or non-use) of `@:allow`, `@:forward`, interfaces, and generics patterns — extend the existing architecture rather than introducing a parallel one (e.g., don't add a second event-bus system if one already exists).
- When adding new fields/classes that interact with an existing macro/metadata/codegen system (per §3.6/§12), follow the exact pattern of existing usages precisely — mismatched metadata argument types/shapes typically fail silently at codegen time rather than raising a normal compile error.
- Do not introduce global mutable singletons/static state as a shortcut unless the project already relies on that pattern (many small Flixel projects do use `FlxG`-style statics pervasively — match, don't fight, existing architecture, but don't add *new* unnecessary global state either).
- Avoid unjustified `cast`/`Dynamic`/`Reflect` — if genuinely needed (e.g., JSON parsing, plugin/factory systems), isolate it behind a narrow, well-typed boundary rather than letting `Dynamic` propagate through gameplay code.
- When fixing a bug, add the smallest reproducer-driven fix; when adding a feature, follow the nearest existing analogous pattern in the codebase (e.g., "how does the existing `EnemySpawner` do pooling" before writing a new spawner from scratch).

---

## 11. Common Patterns & Anti-Patterns

**Patterns (prefer these):**
- Object pooling via `FlxTypedGroup.recycle()` for frequently spawned/killed entities.
- `FlxPoint.get()`/`.put()` and `FlxRect.get()`/`.put()` pooling in hot per-frame code.
- Typed groups (`FlxTypedGroup<Enemy>`) over base `FlxGroup` + casting.
- Centralized input wrapper (`Controls`/`Input` class) over scattering raw `FlxG.keys` checks through gameplay classes.
- `FlxSignal`/typed callback fields over ad hoc string-based event names.
- Using Flixel's asset/bitmap cache (`FlxG.bitmap`, `loadGraphic`) rather than bypassing to raw `openfl.utils.Assets` for Flixel sprites.
- `#if debug`-guarded diagnostics rather than always-on `trace()` spam in hot loops.

**Anti-patterns (flag/avoid unless justified):**
- Allocating new `FlxPoint`/arrays/closures every frame inside `update()`.
- Holding references to objects across `FlxG.switchState()` without accounting for their destruction.
- Setting `sprite.cameras = [cam]` early and forgetting it won't auto-track newly added cameras (§4.6).
- Mutating `FlxText.text` every frame unconditionally instead of only on change.
- Mixing raw `openfl.utils.Assets` bitmap loads with Flixel sprites when the project otherwise consistently uses Flixel's own loaders (creates duplicate cached textures / cache-invalidation mismatches).
- Deleting/renaming unfamiliar metadata or "unused-looking" classes without checking for macro/reflection/DCE-`@:keep` consumers (§3.6, §3.8).
- Writing platform-agnostic-looking code that silently only works on one target due to an unguarded `#if` branch or target-specific null/async asset timing (§7).
- Introducing `Dynamic`/`Reflect`-based dispatch to "simplify" what a `switch` on an enum (with exhaustiveness checking) would express more safely.
- Fighting both `Stage.scaleMode` and `FlxG.scaleMode` simultaneously.

---

## 12. Worked Example: Reading Metadata-Heavy / DSL Code

Given:
```haxe
@posX
@formula("uDisplayRotateX(aPos)")
@set("properties")
public var x:Int;
```

**Reasoning process to model:**
1. None of `@posX`, `@formula`, `@set` are built-in Haxe compiler metadata (compare against the known list in §3.6) → they are **project-specific**, consumed by *something* not visible in this snippet.
2. The shape strongly suggests a **property/shader-uniform binding system**: `@formula("uDisplayRotateX(aPos)")` looks like it maps this Haxe field to a GLSL uniform expression (`uDisplayRotateX`, `aPos` looking like a shader attribute), and `@set("properties")` looks like it registers the field into a named property bag (perhaps for a runtime editor/inspector or serialization group called `"properties"`). `@posX` may be a marker/category tag (e.g., "this represents the X position semantic") consumed by whatever iterates fields with that tag.
3. **Action, not assumption**: search the repo for a `@:build` macro class, a codegen script, or a runtime `Reflect`-based scanner that calls something like `field.meta.extract("formula")` or a string search for `"formula("`/`"@set"` in any `Macro.hx`, `Build.hx`, `tools/`, or non-`.hx` codegen scripts. Also check for generated output files (possibly `.cpp`/`.glsl`/`.json`) that contain the literal string `uDisplayRotateX` — that confirms the consumer and its exact expected format.
4. Only after finding (or explicitly failing to find, and reporting that) the consumer should you: (a) modify the field while preserving all three metadata exactly, or (b) add an equivalent field elsewhere by copying the *exact* pattern from a working sibling field, or (c) modify the macro/consumer itself if the task requires changing behavior — and only with full understanding of every other field it also processes (a shared macro likely governs many fields; a change must not break the others).
5. If asked "what does this do?" without finding a consumer, answer honestly: "This is custom metadata; I found no macro/codegen consumer in this repository — it may be processed by an external tool not present in this checkout, or it may be vestigial. I recommend checking [specific place] / confirming with the team before treating it as dead code."

Apply this same discipline to **any** unfamiliar metadata, not just this example: `@:sceneGraph`, `@:serializable`, `@:editable`, `@:networked`/`@:sync` (multiplayer state-sync codegen), `@:component`/`@:ecsInclude` (ECS codegen), etc. are all real patterns seen in Haxe game codebases with custom build macros — the investigation procedure is identical.

---

## 13. Quick Reference

### 13.1 Frequently used `FlxG` statics
`FlxG.state`, `FlxG.camera`/`FlxG.cameras`, `FlxG.sound`, `FlxG.keys`, `FlxG.mouse`, `FlxG.touches`, `FlxG.gamepads`, `FlxG.width`/`height` (game/world resolution, not window pixels), `FlxG.save`, `FlxG.log`, `FlxG.watch`, `FlxG.bitmap`, `FlxG.plugins`, `FlxG.worldBounds`/`FlxG.worldTime`? (verify exact fields per version), `FlxG.scaleMode`, `FlxG.autoPause`, `FlxG.fixedTimestep`, `FlxG.timeScale`, `FlxG.elapsed`.

### 13.2 Version-sensitive APIs to double-check before use
- `FlxG.renderBlit`/`FlxG.renderTile` — Flixel 4 legacy distinction; largely irrelevant/removed by Flixel 5 (tile/hardware rendering standard).
- `FlxSprite.setFacingFlip` vs manual `flipX` — naming/behavior has shifted across versions.
- `FlxTilemap` auto-tiling constants and Ogmo loader location (core vs addons) have moved between versions.
- OpenFL `Assets` async vs sync API surface (`Assets.loadBitmapData` returning `Future<BitmapData>` in newer OpenFL vs callback-based in older) — check the installed OpenFL version's actual signature before writing loading code.
- Lime `lime.app.Application` vs older `lime.app.Application`/`openfl.display.Application` naming shifted across Lime major versions.
- **Rule**: when unsure, open the actual installed library source (`.haxelib`/`haxe_libraries` path) for the class in question rather than relying on possibly-outdated memorized signatures.

### 13.3 "Which layer owns this?" quick lookup
| Concern | Owning layer |
|---|---|
| Window creation, native input backends, asset packaging | Lime |
| Display list, `Sprite`/`BitmapData`/`TextField`, Flash-style events | OpenFL |
| Game loop, states, sprites-with-physics, collision, cameras, tweens/timers | HaxeFlixel |
| Compile-time types/macros/metadata/DCE | Haxe compiler |

---

## 14. Checklists

**Before proposing an architecture change:**
- [ ] Read the relevant state(s)/class hierarchy fully.
- [ ] Identified library versions in use.
- [ ] Confirmed no existing helper/pattern already solves this.
- [ ] Confirmed the change doesn't alter unrelated behavior.

**Before deleting/renaming anything:**
- [ ] Searched the whole repo for references (including string-based/reflective references).
- [ ] Checked for macro/`@:build`/`@:keep`/DCE interactions.
- [ ] Checked git history/blame for context if purpose is unclear.

**When debugging:**
- [ ] Reproduced with known target + build mode.
- [ ] Checked lifecycle ordering (construction vs `create()` vs first update vs first draw).
- [ ] Checked `exists`/`active`/`visible`/`alive`, camera assignment, group membership.
- [ ] Checked for target-specific asset/async timing issues.
- [ ] Only then changed code, with a stated root cause.

**When optimizing:**
- [ ] Profiled/measured (stats overlay or platform profiler), not assumed.
- [ ] Classified bottleneck (CPU/GPU/GC/draw-call/asset/framework).
- [ ] Verified fix against a release (not just debug) build where relevant.

**Before submitting any change:**
- [ ] Diff is minimal and scoped to the task.
- [ ] No unjustified `Dynamic`/`cast`/`Reflect` introduced.
- [ ] Matches existing style/conventions.
- [ ] Explained *why*, when the choice was non-obvious (e.g., "used `FlxTypedGroup.recycle` instead of `new` because this spawns 50+/sec and needs pooling to avoid GC spikes on hxcpp/mobile").

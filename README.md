# newskeington — isekai escape

A modular, performance-oriented SugarCube 2.37 UI for Twine/Tweego:
sidebars filled from dedicated passages, a layered scene canvas, and a
canvas-composited character portrait — all driven by a small macro API.

## Layout

```
┌──────────┬─────────────────────────────┬──────────┐
│  left    │  scene canvas (region 2)    │  right   │
│  sidebar ├─────────────────────────────┤  sidebar │
│ (Panel-  │  story passages (region 3)  │ (Panel-  │
│  Left)   │                             │  Right)  │
└──────────┴─────────────────────────────┴──────────┘
        + floating tabbed window (inventory, quests, …)
```

Both sidebars are empty shells wired to SugarCube's native
`data-passage` mechanism: their content is whatever the `PanelLeft` /
`PanelRight` passages produce, re-rendered automatically on **every
passage transition** and on every `UI.update()` call. Nothing in the
sidebars is hardcoded — they are composed sequentially from macros.

## Files (src/)

| File | Module | Contents |
|---|---|---|
| `00-meta.twee` | — | StoryTitle / StoryData |
| `10-shell.twee` | shell | `StoryInterface` regions + layout CSS |
| `20-core.twee` | core | `setup.ui.h()`, `setup.ui.refresh()`, `setup.media` resolver |
| `30-widgets.twee` | widgets | read-only display primitives (macros) |
| `40-actions.twee` | actions | `setup.actions` registry + `<<uiAction>>` + built-ins |
| `50-scene.twee` | scene | `<<scene>>` macro + layer renderer |
| `60-avatar.twee` | avatar | layered canvas portrait + `<<avatarview>>` / `<<avatarset>>` |
| `70-window.twee` | window | floating window + tab registry |
| `80-panels.twee` | content | `PanelLeft`, `PanelRight` — **edit these** |
| `85-windows.twee` | content | starter `Inventory` / `Quests` tab passages |
| `90-start.twee` | content | `StoryInit` + `Start` |

Script modules are plain `.twee` `[script]`/`[stylesheet]` passages —
no build tooling of your own is required; the knot extension (tweego)
compiles `src/` as-is.

## Widget macros (read-only)

```
<<uiContainer "id"? "title"?>> … <</uiContainer>>
<<uiCard "title"?>> … <</uiCard>>
<<uiRow>> … <</uiRow>>
<<uiLabel text>>
<<uiValue "label" value "unit"?>>
<<uiProgress "label" value max "color"?>>
<<uiCheck "label" bool>>
<<uiDivider "label"?>>
<<uiSpacer>>            → flex glue, pins following content to the bottom
<<uiRefresh>>           → re-render panels now (safe mid-passage)
```

All arguments are TwineScript expressions evaluated at render time:
`<<uiProgress "HP" $hp $maxhp "#c0392b">>`. Panels re-render on every
passage transition; after changing `$vars` *without* navigating, call
`<<uiRefresh>>` (or `setup.ui.refresh()` from JS).

## Actions (the only interactive widgets)

```
<<uiAction "back">>   <<uiAction "forward">>   <<uiAction "saves">>
<<uiAction "settings">>   <<uiAction "restart">>
<<uiAction "inventory">>  <<uiAction "quests">>
```

Register your own (or override a built-in):

```js
setup.actions.register("map", {
	label: "Map",
	icon: "\uF013",            // sc-icons PUA glyph, or any string (emoji…)
	handler: function () { setup.ui.openWindow("map"); },
	disabled: function () { return !$hasMap; }   // optional
});
```

Icons render via SugarCube's bundled `sc-icons` font when given a
Private-Use-Area character (e.g. `"\uF0C7"`); anything else renders as
plain text.

## Scene canvas

From any story passage (typically at the top):

```
<<scene "bg" "forest_clearing">>
<<scene "sprites" "alice" "merchant">>   → laid out with even spacing
<<scene "sprites">>                      → clears that layer
<<sceneClear>>                           → clears everything
```

The first argument is a layer **name** or **index**. Layers are
configured in JS (defaults: `bg`, `scene`, `sprites`, `effects`):

```js
setup.scene.configure({ layers: [
	{ name: "bg",      fit: "cover"   },   // full-bleed, stacked
	{ name: "sprites", fit: "even"    },   // side-by-side, bottom-aligned
	{ name: "fx",      fit: "contain" }    // centered, stacked
]});
```

Scene state lives in `$scene`, so saves, rewind, and back/forward all
restore the correct scene. Re-rendering diffs per layer — unchanged
layers don't touch the DOM.

## Avatar portrait

`<<avatarview>>` (already in `PanelRight`) mounts a persistent
`<canvas>`; layers are transparent PNGs composited in 2D, bottom → top.

```
<<avatarset "body_hair" "hair_red_long">>
<<avatarset "makeup">>        → layer to "none"
<<avatarclear>>
```

The layer stack is **yours to define** — edit the config block at the
top of `60-avatar.twee` (or call it from any script):

```js
setup.avatar.configure({
	width: 600, height: 900,            // design coordinate space
	layers: [
		{ id: "body_base" }, { id: "body_face" }, { id: "body_hair" },
		{ id: "apparel_bottom" }, { id: "apparel_top" },
		{ id: "makeup" }, { id: "accessory" }
	]
});
```

From JS (inventory/equipment systems, character creators):

```js
setup.avatar.setLayer("apparel_top", "armor_leather");
setup.avatar.setLayer("body_face", "face_scar", { sx: 0, sy: 0, sw: 300, sh: 300 });  // crop
setup.avatar.getLayer("apparel_top");   // → { img, … } | null
setup.avatar.clearLayer("makeup");
setup.avatar.preload(["hair_red_long", "armor_leather"]);
```

State lives in `$avatar` → save/rewind-safe. Renders are
rAF-coalesced (100 `setLayer` calls in one tick = 1 draw), images are
cached and preloaded, the backing store is devicePixelRatio-aware, and
the canvas is a singleton re-attached across panel re-renders (its
bitmap and cache survive). A `:avatarrender` jQuery event fires after
every draw.

## Images

Logical ids resolve through `setup.media`:

- default: `media/<id>.png` relative to the story html
- per-id override: `setup.media.map["alice"] = "img/chars/alice.webp"`
- global override: `setup.media.base = "art/"; setup.media.ext = ".webp"`
- ids that look like paths/URLs (`./x.png`, `https://…`) pass through

## Floating window

`Inventory` / `Quests` passages auto-register as tabs (that's all the
built-in buttons need). Custom tabs:

```js
setup.ui.registerTab({ id: "map", title: "Map", passage: "Map" });
// or: setup.ui.registerTab({ id: "x", title: "X", render: el => { … } });
setup.ui.openWindow("map");
```

## Notes on state & performance

- SugarCube snapshots `$vars` at each passage entry; mid-passage
  `<<set>>` mutations propagate *forward* on the next navigation but are
  rolled back if the player goes Back — standard SugarCube semantics.
  Keep `<<scene>>`/`<<avatarset>>` calls in passage top matter so the
  scene/portrait is always re-derived on render.
- Panel refresh (both sidebars) ≈ 2 ms; scene no-op apply ≈ 0 ms
  (diffed); avatar redraws are coalesced per animation frame.
- Panel/tab passages should be display-only — they re-wikify on
  `UI.update()`, so avoid `<<set>>` side effects in them.

## Building

No build files live in this repo on purpose. With tweego:

```
tweego -f sugarcube-2 -o game.html src
```

or just point the knot VSCode extension at `src/` as usual.

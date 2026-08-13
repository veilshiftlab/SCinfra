# SCinfra — Stable UI Infrastructure

A Tweego + SugarCube 2 project (`roommate` story). The shell is a **fixed virtual resolution** that scales uniformly to fit any viewport, so the layout never warps.

## Architecture overview

```
viewport (any size)
└── #stage (fullscreen black backdrop, flex-centers the wrapper)
    └── #scale-wrapper (1600×900, transform: scale(min(vw/1600, vh/900)))
        └── #app-shell (grid: 240px sidebar | 1fr main)
            ├── #left-sidebar (.sidebar)
            │   ├── <header class="sidebar-header"> (title placeholder → PNG later)
            │   └── <div class="sidebar-body"> (grid, align-content: space-between)
            │       ├── Time card        ← top
            │       ├── Resources card   ← middle (extra space distributed around it)
            │       └── Actions card     ← bottom
            │           ├── .ui-row-pair (back | forward, grid auto auto, natural width)
            │           ├── Saves button
            │           ├── Settings button
            │           └── Restart button
            └── #main-area
                ├── #scene-canvas (full width, 4:1, pixel-perfect, auto-hide)
                │   ├── <img class="scene-canvas__image"> (z=1, fills canvas)
                │   └── <img class="scene-canvas__character"> (z=2+, bottom-aligned)
                └── .story-content
                    ├── #passages (scrollable prose, flex: 1)
                    └── .action-bar (vertical choice list, flex column, min/max height)
```

## Grid vs Flex choices

| Element | Layout | Why |
|---------|--------|-----|
| `.app-shell` | grid `sidebar 1fr` | Two-column split, clean and declarative |
| `.sidebar-body` | grid + `align-content: space-between` | Distributes extra space between 3 cards without stretching them |
| `.ui-row-pair` | grid `auto auto` | Two buttons at natural width — no stretching |
| `.action-bar` | flex column | Homogeneous vertical list, natural stacking |
| `.main-area` | flex column | Scene + story-content stack, scene can collapse |
| `.story-content` | flex column | #passages (flex:1) + .action-bar (flex:0) |
| `.ui-grid` (time card, resources) | grid `auto-fit minmax(5em, 1fr)` | Responsive cells |

## Resolution

Base: **1600 × 900** (16:9). Scaled via JS `transform: scale(min(vw/1600, vh/900))`.

Layout math:
- Sidebar: 240px
- Main area: 1360px × 900px
- Scene: 1360 × 340 (4:1, full main-area width)
- Passage area: 1360 × 560 (fixed; grows when scene auto-hides)
- Action bar: 120–240px (min/max, vertical list)

To change: edit `--base-w`/`--base-h` in `01-shell.twee` AND `BASE_W`/`BASE_H` in `02b-shell-behavior.twee`.

## Scene canvas

- **Full width** of the main area, **4:1 aspect ratio** (h:w = 1:4).
- **Pixel-perfect**: `image-rendering: pixelated` on canvas and all images.
- **Auto-hide**: when passage content overflows (needs scrolling), the scene collapses to 0 height and the prose area grows. Controlled by `settings.sceneAutoHide` (default: true, toggleable in the Settings panel).
- **Layers**: background (`<<sceneImage>>`, z=1) + character layers (`<<characterImage>>`, z=2+, 5 positions). Both passage-scoped.

## Passage area

- **Fixed height** = `base-h - scene-h`. Grows when scene auto-hides.
- Scrollable vertically.
- Contains prose (`#passages`) + action bar (`.action-bar`).

## Action bar (choice list)

- **Vertical** list of full-width choice buttons (not horizontal grid).
- `flex-direction: column`, `min-height: 120px`, `max-height: 240px`.
- Holds 5–6 visible buttons; scrolls for 6+ choices.
- Links are auto-moved from the passage into the bar by `setup.shell.refreshActionBar()`.

## Sidebar (left panel)

- **Card order** (top → bottom): Time, Resources, Actions.
- **Space distribution**: `.sidebar-body` is a grid with `align-content: space-between`. Time at top, Actions at bottom, Resources in the middle. Extra space is split equally between the two gaps.
- **Back/forward**: wrapped in `.ui-row-pair` (grid `auto auto`) so they sit side by side at natural width — no stretching.
- **Title placeholder**: `<header class="sidebar-header">` with `min-height: 56px`, centered. Will be replaced with a game title PNG.
- **Responsive collapse**: below 760px viewport, sidebar slides off-screen, hamburger toggle appears.

## Settings panel

SugarCube's built-in `UI.settings()` dialog (opened via the Settings button) contains:

- **Auto-hide scene for long passages** (toggle, default: on) — hides the scene image when passage text overflows.

Add more settings in `src/11-settings.twee` via `Setting.addToggle()` / `Setting.addList()` / `Setting.addRange()`.

## NPC catalog (dialog colors)

`setup.npcs` is the single source of truth for NPC data. `<<dialog "npc_id">>` looks up the NPC and uses `npc.color`. Story passages never hardcode colors.

## Typography

Configurable via CSS variables:
```css
--font-family: "Noto Sans", "Liberation Sans", "DejaVu Sans", sans-serif;
--font-mono: "Noto Sans Mono", "Liberation Mono", "DejaVu Sans Mono", monospace;
--font-size-prose: 15px;
--font-size-ui: 13px;
```

To add a custom font: add a `@font-face` declaration and update `--font-family`.

## Macros

| Macro | Purpose |
|-------|---------|
| `<<sceneImage "path.png">>` | Set scene background (passage-scoped). |
| `<<characterImage "path.png" [position] [name]>>` | Add character layer (passage-scoped, stacks). |
| `<<removeCharacter "name">>` | Remove one named character layer. |
| `<<clearCharacters>>` | Remove all character layers. |
| `<<clearScene>>` | Remove background + all characters. |
| `<<dialog "npc_id">>...<</dialog>>` | Colorized dialogue, NPC-id driven. |

## Building

```bash
/home/z/my-project/scripts/build.sh
```

Compiles `src/` to `build/roommate.html` (gitignored, stays in repo). Packages source diff into `/home/z/my-project/download/SCinfra-<timestamp>.zip`.

## Files

| File | Role |
|------|------|
| `src/00-meta.twee` | Story metadata (title, IFID, format). |
| `src/01-shell.twee` | StoryInterface HTML + ShellStyle CSS (layout, palette, pixel-perfect rendering, responsive collapse, grid/flex choices). |
| `src/02-core.twee` | `setup.ui` helpers (h() element builder, refresh). |
| `src/02b-shell-behavior.twee` | `setup.shell` runtime: scale factor, sidebar-body wrapper, action bar, scene auto-hide, sidebar toggle. |
| `src/03-widgets.twee` | UI widget macros (uiCard, uiProgress, uiValue, etc.). |
| `src/03-widgets-style.twee` | Widget CSS. |
| `src/04-actions.twee` | `setup.actions` registry + `<<uiAction>>` macro. |
| `src/04-actions-style.twee` | Action button CSS. |
| `src/05-scene.twee` | `setup.scene` + `<<sceneImage>>` / `<<characterImage>>` macros. |
| `src/06-panels.twee` | `PanelLeft` passage — Time, Resources, Actions cards (top to bottom). |
| `src/07-catalogue.twee` | `setup.items` + `setup.npcs` registries. |
| `src/08-stats.twee` | `setup.stats` (minimal stat container). |
| `src/09-time.twee` | `setup.time` + time macros. |
| `src/10-dialog.twee` | `<<dialog "npc_id">>` macro (NPC-catalog driven). |
| `src/11-settings.twee` | SugarCube `Setting` definitions (scene auto-hide toggle). |
| `src/20-init.twee` | StoryInit — initial state. |
| `src/21-begin.tw` | Story passages. |
| `src/story.css` | Custom story styles (inline link styling, dialog overrides). |

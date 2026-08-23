# Vertical App Grid Max
A GNOME Shell extension that turns the default horizontal, paged app grid
into a single continuously-scrolling vertical one, with folders, app
hiding, drag-and-drop reordering, and touchscreen support layered on top.
App icon size and spacing can be customized in the extension preferences.

Originally forked from [lublst/gnome-vertical-app-grid](https://github.com/lublst/gnome-vertical-app-grid),
which only handled the vertical layout itself (no folders or drag-and-drop).
Renamed since it's since grown well beyond that scope; retains the original
project's MIT license (see `LICENSE`).

This fork adds four things the original didn't support:

- **Folders** — right-click (or long-press) any app icon → *Add to Folder* →
  pick an existing folder or create a new one, or just drag one icon onto
  another. Folders are backed by GNOME's own `org.gnome.desktop.app-folders`
  storage and rendered with GNOME's real `FolderIcon`/`AppFolderDialog`/
  `FolderView` classes (see "Reusing GNOME's real folder classes" below) --
  so they interoperate with the regular overview and GNOME Tweaks. Folders
  are also listed (and can be deleted) from the extension's preferences
  window.
- **Drag-and-drop** — drag any icon onto another app to group them into a
  new folder, drag it onto an existing folder to add it there, or drop it
  between icons to reorder the grid. Dragging into/out of the Favorites
  section adds/removes the favorite too, same as the stock dash. Reordering
  the main grid this way switches *App Sorting* to *Manual* automatically;
  reordering favorites switches *Favorites Sorting* to *Dash*. Both can
  be reset from preferences.
- **App hiding** — right-click (or long-press) any app icon → *Hide from App
  Grid*. Hidden apps disappear from the grid entirely; unhide them from the
  same menu (now reading *Show in App Grid*) or from the *Hidden Apps* list
  in preferences.
- **Touchscreen scrolling** — the grid can now be scrolled by dragging/
  flicking with a finger. This uses GNOME Shell's own `SwipeTracker`, the
  same gesture-recognition primitive the stock app grid uses internally for
  its paged swipe navigation, rather than hand-rolled touch-event code, so
  it should behave consistently with the rest of the overview's gestures.

## Reusing GNOME's real folder classes

Folders don't use a hand-rolled popup. Instead, `VerticalAppDisplay` acts as
a minimal `parentView` for GNOME's actual `AppDisplay.FolderIcon` /
`AppFolderDialog` / `FolderView` classes (imported from the stock
`ui/appDisplay.js` module) -- implementing just the handful of methods those
classes call on their parent (`addFolderDialog()`, `getAppInfos()`,
`selectApp()`). This means the folder tile, popup, inline rename, and
open/close animation are all genuine GNOME code, not a reimplementation.

One thing that *isn't* reused: `FolderIcon`'s own built-in drop-acceptance
(`_canAccept()`/`acceptDrop()`). That logic requires the dragged icon's
ancestor chain to include a real (and non-exported) `BaseAppView` subclass,
which this extension's custom vertical grid can never satisfy without
either subclassing something GNOME doesn't export or monkey-patching several
private call sites across `appDisplay.js`. Rather than do that, this
extension's own drag-and-drop (`_getDropTarget`/`_onDrop` in
`VerticalAppDisplay`) does its own hit-testing to decide when a drop should
create or join a folder, then writes directly to the same
`org.gnome.desktop.app-folders` storage `FolderIcon` itself reads from --
same end result, without touching gated internals.

## Notes on this fork

- **Folder storage changed from an earlier version of this fork.** If
  you've used a previous build that stored folders in this extension's own
  `folders` GSettings key (a JSON blob), those folders won't carry over --
  the key has been removed and folders now live in GNOME's own
  `org.gnome.desktop.app-folders`. Hidden apps (`hidden-apps`) are
  unaffected; that's still this extension's own key.
- A folder whose membership comes only from GNOME's category-matching (e.g.
  a distro-shipped default folder like "Utilities") isn't recognized by this
  extension at all -- it won't show up as a folder tile in the grid, and
  apps that belong to it purely by category won't be excluded from the main
  grid (they'll appear in both places). This extension only reads a
  folder's explicit `apps` list, deliberately: reading `.getAppIds()` off
  the cached `FolderIcon` instead (which resolves categories too) turned
  out to depend on that icon's own GSettings-notification timing, which
  isn't guaranteed to have caught up by the time this extension's own
  redisplay runs, and produced actual bugs (an app added to a folder
  appearing to vanish -- excluded from the grid, but not yet in the folder
  either). Folders created through this extension always have an explicit
  `apps` list, so this only affects folders this extension didn't create.
- The folder popup's own content grid isn't draggable (no reordering
  within a folder, no dragging an app back out that way) -- that still goes
  through the right-click menu → *Remove from Folder*.
- Drag-and-drop reordering (the *grid layout*, not folder membership) is
  stored independently per section: the main grid's custom order lives in
  the `custom-order` key and only takes effect while *App Sorting* is set to
  *Manual*; favorites reordering reuses GNOME's own favorites order (the
  same one the stock dash drags around), and is most reliable once
  *Favorites Sorting* is set to *Dash* (the first drag switches it
  there automatically, so results may look odd for exactly one drag if you
  were previously sorting favorites alphabetically or by usage).
- `extension.js` now checks that the private Shell internals it depends on
  (`Main.overview._overview._controls` and friends) still look the way it
  expects before touching them, and wraps each risky step in a try/catch
  that logs and unwinds cleanly instead of crashing -- similar in spirit to
  the defensive-checks pattern in `app-grid-tuner`. If GNOME Shell changes
  those internals in a future version, this extension should now fail
  inert (do nothing, with a warning in the logs) rather than crash the
  session.
- This was written and reviewed against the actual current GNOME Shell
  source (`js/ui/appDisplay.js`, `js/ui/iconGrid.js`) for the class
  signatures folders depend on, and the real
  `org.gnome.desktop.app-folders.gschema.xml`, but **has not been tested
  inside a running GNOME Shell session**. The riskiest pieces if something
  doesn't work out of the box on your shell version:
  - The `parentView` shim (`addFolderDialog`/`getAppInfos`/`selectApp` in
    `VerticalAppDisplay`) -- if `AppFolderDialog` or `FolderView` call
    something else on their parent that wasn't exercised by the code paths
    this extension uses, it'll show up as an exception when opening a
    folder for the first time.
  - `ModalDialog` usage in `promptText()` (used for naming/renaming
    folders) -- used the way GNOME Shell's own code uses it, but exact
    method signatures do shift between shell releases.
  - The `SwipeTracker` wiring in `VerticalScrollView` (touchscreen
    scrolling) -- in particular, scroll direction may come out inverted
    depending on how `SwipeTracker` interprets `Clutter.Orientation.VERTICAL`
    on your shell version; if so, negate `progress` in
    `_onSwipeUpdate`/`_onSwipeBegin`.
  - The drop-target/hit-testing logic (`_getDropTarget`,
    `_onDrop`/`_onDragMotion` in `VerticalAppDisplay`) that decides "merge
    into a folder" vs. "reorder here" -- this part is entirely custom code,
    unlike the folder classes themselves.
- To test changes quickly without logging out, run a nested GNOME Shell:
  `dbus-run-session -- gnome-shell --nested --wayland`, then install the
  extension into that session and watch `journalctl -f -o cat /usr/bin/gnome-shell`
  for errors.

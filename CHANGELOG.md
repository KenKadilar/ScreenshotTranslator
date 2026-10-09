# Changelog

Engineering log for Screenshot Translator. The app version is tracked in-app
(`APP_VERSION`) and in git history rather than in filenames. Dates approximate.

## v4.6: current, F11 hotkeys (F12 is reserved by Windows)

The v4.5 move to F12 was wrong, and the reason is worth writing down so nobody sets it
back. **Windows reserves F12 for the debugger**, so `RegisterHotKey` refuses it. From
Microsoft's RegisterHotKey documentation:

> "The F12 key is reserved for use by the debugger at all times, so it should not be
> registered as a hot key. Even when you are not debugging an application, F12 is
> reserved in case a kernel-mode debugger or a just-in-time debugger is resident."

Measured here, registering a full ctrl/shift/alt triple at once:

```
F12    ctrl=ok   shift=FAIL(1409)   alt=ok
F11    ctrl=ok   shift=ok           alt=ok
F10 / F9 / F8 / PageUp / Home / End / Insert / PrintScreen: all three ok
```

So the reservation is narrower than the docs state: bare F12 and Shift+F12 are refused
with 1409 (`ERROR_HOTKEY_ALREADY_REGISTERED`) while Ctrl+F12 and Alt+F12 register fine.
Relying on those two would mean depending on undocumented behavior, so all three defaults
are now `ctrl+f11`, `shift+f11` and `alt+f11`.

This cost hours of misdiagnosis, because 1409 reads exactly like "another app took it".
Stopping Steam, Logitech G HUB, Core Temp, Discord's elevated keybind helper, all 25 Razer
processes, NVIDIA Broadcast, NVDisplay.Container, explorer, taskhostw, atieclxx, TabTip
and ctfmon changed nothing. Two things that would have saved the time: a registered hotkey
is held by the owning thread and is freed only by unregistering it or by that process
exiting (suspending the process does nothing), and the API docs answer this outright.

Page Up and the other nav keys register fine but were rejected on cost: a `RegisterHotKey`
binding is system-wide, so the focused app never sees the key. Ctrl+PageUp would have
taken browser and editor tab switching, and Shift+PageUp page-wise text selection.
F11's only real collisions are Office's Alt+F11 and Shift+F11.

## v4.5: F12 hotkeys and a truthful hotkey error

This keyboard has no dedicated PrintScreen, so the three defaults moved to
`ctrl+f12`, `shift+f12` and `alt+f12`, in `DEFAULT_HOTKEY_*` and in the `.env` seed
template. The tray menu labels no longer hardcode the key names: they are built from
the live bindings by `update_capture_action_labels()`, which `update_tray_tooltip()`
calls, so rebinding through the in-app editor can no longer leave a menu item
advertising a shortcut that does nothing.

Found while testing the new bindings: **`shift+f12` will not register on this machine.**
`RegisterHotKey` returns 1409 (`ERROR_HOTKEY_ALREADY_REGISTERED`), so another running
app owns it. `ctrl+f12` and `alt+f12` register fine, as do `shift+f11`, `shift+f10` and
`shift+f9`.

That was hard to see, because the "Hotkey failed" toast reported `Windows error: 0`.
The cause is `ctypes.windll`, which does not enable last-error capture, so
`ctypes.get_last_error()` never saw the failure code. Hotkey calls now go through
`hotkey_user32()` (`ctypes.WinDLL("user32", use_last_error=True)`) with
`ctypes.set_last_error(0)` before the call, and 1409 is spelled out as "Another app
already owns this shortcut." A hotkey that silently fails is the kind of thing that
looks like a broken app for weeks.

## v4.4: English is the default OCR language

A fresh install came up on "All-arounder / Auto", which hands PaddleOCR its default
setup. English is what the tool actually gets pointed at, so that is now the default in
both places that decide it: `DEFAULT_LANG` (the fallback when `OCR_LANG` is absent) and
the `.env` seed template written on first run. Existing `.env` files are untouched, so
a saved language choice still wins; the tray menu picker is unchanged.

## v4.3: multi-monitor region capture

Moving to a two-screen desktop (extended mode, 2560x1440 each, the second one to the
LEFT of the primary so the virtual desktop starts at x = -2560) made region capture
unusable: the overlay showed both monitors squashed onto one screen, and a dragged box
came out roughly twice as wide as drawn, spilling across both displays.

One cause for both symptoms. `CaptureOverlay.__init__` sized the overlay to the whole
virtual desktop correctly, and then `_prepare_overlay` called `showFullScreen()`, which
discards an explicit geometry and clamps the window to a single screen. The frozen image
stayed 5120 wide while the widget became 2560, so the v3.8 conversion in
`local_rect_to_image_rect` scaled every x by `5120 / 2560 = 2.0`, and
`drawPixmap(self.rect(), ...)` squeezed the full-desktop image into one monitor. On a
single-screen laptop the virtual desktop *is* the primary screen, so the factor was 1.0
and the call looked harmless for three versions.

Two changes. `_prepare_overlay` now calls `show()`, so the geometry survives. And the
geometry comes from a new `qt_virtual_geometry()`, the union of `QApplication.screens()`,
instead of the mss monitor dict: `setGeometry` wants Qt's logical coordinates while mss
reports physical pixels, and the two only agree while every display sits at 100%
scaling. The v3.8 design is untouched, the one conversion point still does the scaling;
it just needs an overlay that genuinely covers the capture area. The mss values remain
as a fallback if Qt reports no screens.

The invariant worth keeping: **the overlay widget must cover exactly the area the frozen
image covers.** Anything that resizes it to one screen, `showFullScreen()` included,
brings this back.

## v4.2: quiet results

The result window popped up over whatever you were reading on every single capture,
which is the wrong trade for a tool you fire dozens of times an hour. A finished
capture now just swaps the existing progress bubble to "OCR complete" (or
"Translation complete") for one second and lets it disappear. The text still goes to
the clipboard and to its `.txt` file exactly as before, and "Show Last OCR Result" in
the tray menu still opens the full window on demand.

The result window is kept for **failures** (`on_ocr_failed`, `on_translation_failed`),
where you do want the detail on screen, and a failed translation still shows the OCR
text it recovered.

`StatusToast.show_status` grew an `auto_hide_ms` argument backed by a single-shot
`QTimer` that belongs to the widget. A new status stops that timer first, so a capture
fired inside the one-second window cannot have its "Reading text in progress..." bubble
hidden by the previous capture's pending timer.

## v4.1

- DeepSeek translation runs on its own worker thread; OCR text is preserved and
  copied to the clipboard even when the API call fails, so a network/key error never
  costs you the recognized text.
- Tray tooltip and menu reflect the live hotkey bindings and current OCR language.
- Per-month output folders (`YYYY-MM/`) under `DATA/ScreenshotTranslator/` for
  screenshots, OCR text, and translation output.

## v3.8: DPI / scaling correctness (the bug that drove the rewrite)

The selection overlay originally subtracted monitor offsets from the selection rect.
On laptops and multi-monitor setups with display scaling, Qt's widget coordinates and
mss's screenshot pixels are **not** 1:1, so the offset math produced crops shifted away
from what the user dragged over. Fixed by having the overlay store *widget-local*
coordinates and convert them to frozen-image pixel coordinates in one place
(`local_rect_to_image_rect`), scaling by `image_size / widget_size`. Monitor-offset
subtraction was removed entirely; reintroducing it is the easiest way to bring the bug
back.

## v3.x: native hotkeys, model cache, language handling

- **Native Win32 hotkeys.** Replaced the `keyboard` package's global hook (used in
  early versions) with `RegisterHotKey` + a `WM_HOTKEY` native event filter. The
  low-level hook could leave modifier keys (`Ctrl`/`Shift`/`Alt`) logically stuck after
  a capture; the native API doesn't. The `keyboard` package is now used only inside the
  hotkey-editor dialog to *record* a new shortcut, and the app unregisters its own
  hotkeys while that dialog is open so the recorder isn't fighting the live bindings.
- **Local model seeding.** Bundled PaddleOCR models are copied into the user's PaddleX
  cache on startup (size-compared, skip-if-present), with import/export tools in the
  debug menu and a copy-report log. Weights are never committed; they seed/download at
  runtime.
- **Language fallbacks.** `auto` uses PaddleOCR's default; `en`/`japan` use their
  models; `fr`/`tr` map to the English model on purpose. The dedicated `latin` model
  raised `No models are available for the language 'latin'` on this PaddleX build and
  the per-language models triggered extra remote downloads. Mapping to `en` handles
  most Latin-script text without that cost.

## v2.5: tray-menu screenshots without a pre-capture delay

Earlier versions closed the tray menu *before* grabbing the screen, which both lost the
menu from the capture and added a visible delay. Now the full virtual desktop is grabbed
*first* (while transient UI like the open tray menu is still on screen), then the real
menu is closed so it can't steal the first mouse click from the selection overlay. The
overlay paints the frozen grab underneath itself, so menus and tooltips can be captured.

## v2.x: OCR pipeline and threading

- **Persistent OCR worker.** PaddleOCR is initialized once in a `QThread` worker and
  reused across captures instead of constructing a new engine per image; changing the
  OCR language tears down and rebuilds that worker.
- **Tolerant result parsing.** `extract_text_lines_from_result` walks PaddleOCR's
  varying result shapes (dicts with `rec_texts`/`rec_scores`, nested list/tuple
  structures, generic text keys) and applies a configurable min-score filter.

## Cross-cutting / platform

- **No flashing consoles.** On Windows, `subprocess.Popen` is subclassed
  (`CREATE_NO_WINDOW` + hidden `STARTUPINFO`) so PaddleOCR's child processes don't pop
  console windows. The subclass preserves `Popen` as a class because asyncio subclasses
  it.
- **Diagnosable import failures.** The PaddleOCR import is wrapped; failures are written
  to a startup debug file (no secrets) instead of crashing the tray app.
- **Aggressive, deliberate exit.** The tray Exit stops worker threads and then calls
  `os._exit(0)` as a safety net; lingering Paddle worker threads could otherwise keep
  the process alive in Task Manager and block rebuilds.

## Origin

Merged from two earlier personal tools: a region-screenshot grabber and a PaddleOCR
image-to-text extractor. This app is the daily-driver successor to both.

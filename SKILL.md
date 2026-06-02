---
name: easyeda-pro
description: Edit, fix, audit, or reverse-engineer schematics in EasyEDA Pro / LCEDA Pro / JLCEDA / 嘉立创EDA专业版 (立创EDA) by driving the desktop app with computer-use and verifying every change against the exported netlist instead of the canvas. Use when the user wants to modify a schematic, fix circuit or connectivity defects, rename nets, connect floating/unconnected pins, add / move / delete / replace components, swap a chip part variant, wire SPI/power/ground, ground a thermal pad, or check a design's connectivity in EasyEDA Pro / LCEDA Pro / 嘉立创EDA / JLCEDA / EasyEDA. Encodes the per-pin net-port connectivity model, the delete-no-connect-flag → place-net-port → draw-wire workflow, Allegro .tel netlist export + grep/diff verification, the .eprj2/.epro2 file formats, computer-use automation patterns and macOS gotchas (stuck placement tools, occlusion render-freeze, 900s idle-timeout connector drops), and the "hand the user a precise change-list and verify their work via the netlist" division of labor.
version: 1.0.0
---

# EasyEDA Pro / LCEDA Pro / 嘉立创EDA专业版 — schematic editing & netlist-verified fixes

This skill is for editing and fixing **schematics** in EasyEDA Pro (international name) =
LCEDA Pro = JLCEDA = **嘉立创EDA专业版** (立创EDA专业版) — JLCPCB's pro EDA tool. The app has
**no public API and no MCP**, so you drive its GUI with **computer-use**. The whole approach is
built around one idea that makes GUI automation reliable:

## 0. Prime directive: verify with the NETLIST, not your eyes

The canvas lies. A net port that visually touches a pin can have a one-pixel gap and **not be
connected**. A wire can look fine and be on the wrong net. So:

> **After every change, export the netlist and check connectivity in the exported text file —
> never trust the screenshot.**

The netlist is the single source of truth. This catches the failures the eye misses:
- **Virtual connections** — a port/wire that looks attached but isn't (the pin is simply absent
  from the net).
- **Silent net merges** — e.g. importing an edited project once collapsed three separate 5 V rails
  (`5V_A`/`5V_B`/`5V_SYS`) into one 71-pin net. Invisible on the canvas, obvious in the netlist.
- **Part-swap pin remaps** — replacing a chip with a pin-compatible variant keeps pin *numbers* but
  changes pin *functions*; the netlist (which is pin-number based) shows whether you wired the new
  function.

Workflow shape for any task:
1. Open the project. **Export a baseline netlist** and read it — that's ground truth for "before".
2. Make a small, scoped change in the GUI.
3. **Re-export the netlist**, grep/diff the affected nets/pins against what you intended.
4. Only move on once the netlist confirms it. Repeat.

## 1. Mental model: per-pin **net ports** define connectivity

In EasyEDA Pro schematics, nets are usually defined by **per-pin net ports / net labels**
(网络端口 / 网络标签) — the little flags carrying a net name next to each pin — **not** by long
drawn wires. Two pins are on the same net because their ports share the same **name**.

Consequences:
- **Renaming a net = renaming net ports.** Changing one port's name moves *only that pin* to the new
  net. To rename a whole net you must rename **every** port carrying the old name, or the net splits.
- **Moving a pin to another net = renaming its port** (e.g. move a decoupling cap's hot pin from
  `24V_BUS` to `VM` by renaming its port).
- **Connecting a floating pin = adding a port** (and removing any no-connect flag first).
- A net's true membership is whatever the netlist says — always re-derive it from the export.

## 2. File formats (so you know what's editable)

| File | What it is | Editable? |
|---|---|---|
| `*.eprj2` | The project. **SQLite**; schematic data is an **encrypted blob** in `history_data`. | **No** — can't read/script it directly. Edit via the GUI. |
| `*.epro2` | An **export** of the project. It's a **zip**; inside, the `.epru` is a **plaintext** record stream (`head\|\|body\|`, `\n`-separated). | Parseable/scriptable, round-trips losslessly. **But re-importing it can silently merge nets** — verify hard afterward, or avoid the import path. |
| `*.tel` | A **netlist** export (Allegro Telesis format). Plaintext, lists every net and its pins. | **Read-only ground truth.** This is your verification oracle. |

⚠️ Components can share an `id` **across pages**; if you ever parse the `.epru`, index **per page
range** (`SCH_PAGE`), not globally.

## 3. Operation recipes (GUI)

Menu names are given in English (中文) for the localized UI. Adjust to the user's language.

### Export the netlist (your verification step — do it constantly)
`Export → Netlist` (导出 → 网表文件) → type **Allegro(.tel)** → `Export` (导出) → save to disk
(Desktop is convenient). Then read it with the Read tool / shell. Give each export a distinct name
(`Netlist_after_C6.tel`, …) so you can diff against earlier states; **don't overwrite your baseline.**

### Rename a net / move a pin to another net
1. Click the pin's **net port** to select it.
2. In the **Properties** panel → **名称 (Name)** field → type the new net name → Enter.
3. This changes **only that port**. Repeat for every port that must move.
4. **Verify**: re-export, grep the old and new net — confirm the right pins moved and none were left behind.

### Connect a floating / unconnected pin to a net (the robust method)
A floating pin often carries a **no-connect flag** (非连接标识, the green ✗). You must delete it first.
1. **Delete the no-connect flag**: click it → `Del`.
2. **Place a net port**: `Place → Net Port` (放置 → 网络端口, the "input" pentagon). Single-click to
   drop one *next to* the pin.
3. **Name it**: double-click / select the port → Properties → **名称** → set to the target net
   (`GND`, `AVDD`, `3V3`, …).
4. **Wire it**: `Place → Wire` (放置 → 导线), draw from the **pin endpoint** to the **port's
   connection point**.
5. **`Esc`** to exit the tool (if `Esc` doesn't take, **right-click** to cancel — see gotchas).
6. **Verify**: re-export; confirm the pin (`U10.26`, …) now appears in the target net.

> A quicker variant is `Place → Net Label` (网络标签) dropped directly on the pin endpoint, then
> rename. It works but the tool is twitchy (see gotchas), and on right-hand-side pins a stray click
> can start a long free wire. The port+wire method above is the safe default.

### Add a component
`Place → Component` (放置 → 器件, Shift+F) → search the part number → place → wire it / add ports.

### Replace / swap a part variant ⚠️
Select the component → right-click / Properties → **Replace** (替换器件) → search the new part.
**Pin numbers stay; pin functions can change.** Example: a motor driver's pins 33–36 were
`MODE/SLEW/OCP/GAIN` on the hardware variant and become `SDO/SDI/SCLK/nSCS` on the SPI variant — same
numbers, different meaning. After swapping, **re-wire by the new function** and **verify the netlist
pin assignments** (e.g. SPI_MISO must include the SDO pin number, not the old GAIN pin).

### Delete / move / rotate
Delete: select → `Del`. Move a value: double-click → edit **值 (Value)**. Rotate/mirror: select →
Space / right-click.

## 4. Driving the app with computer-use (automation patterns)

- **Batch the select→edit cycle.** The fastest reliable rename is, in one `computer_batch`:
  `left_click(netport)` → `triple_click(Properties 名称 field)` → `type(newname)` → `key(Return)`.
  Chain several pins in a single batch (coordinates all refer to the pre-batch screenshot).
- **Zoom before clicking small targets.** Net-port labels are tiny; use the read-only `zoom` to read
  exact pixel coordinates, then click. Pins ~10–12 px apart are doable but verify with the netlist.
- **Cross-probe to navigate.** Double-clicking a pin entry in the **Networks** panel (网络) jumps the
  canvas to that port and zooms in — handy for finding a specific pin on a big sheet.
- **The Networks panel has no rename.** Right-click there only offers expand/highlight; rename is
  done on the canvas via the port's Properties.
- **Save before exporting** (`Cmd/Ctrl+S`); the export reflects saved state.

## 5. computer-use gotchas (macOS — hard-won)

- **Placement tools get STUCK active.** After `Place → Net Label/Wire`, the tool often **does not exit
  on `Esc`** — and then **every subsequent click places another element** (you'll see NET1, NET2, …
  pile up and stray wires grow). Fix: **right-click to cancel** the placement, *then* the tool is off
  and `Cmd+Z` works again. When in doubt, right-click first, verify the tool is off (move the mouse —
  no preview line follows), then act.
- **`Cmd+Z` goes to whatever has focus.** If the left-panel search box has focus, undo edits the
  search text, not the schematic. Click an empty canvas spot to focus the canvas first — but only
  when no placement tool is active (else the click places something).
- **Copy-paste of a net port can spawn junk** (once produced a stray capacitor). Prefer the explicit
  place-port + wire recipe over paste.
- **Occlusion render-freeze (the UI looks dead).** If the Claude desktop app's window is fully
  covered by the target app, macOS marks it occluded and Chromium suspends painting — the user sees a
  stale frame and can't read/reply, though injected input still works. Mitigate: a **second display**
  (target app on one, Claude on the other) eliminates it; otherwise tile so a sliver of Claude stays
  visible; never run the target app in true-fullscreen. (Launching the app with
  `--disable-renderer-backgrounding --disable-backgrounding-occluded-windows
  --disable-background-timer-throttling` is the deeper fix.)
- **The connector drops after ~15 min idle.** The desktop app's IdleManager disconnects a local
  session after ~900 s idle and tears down computer-use **without auto-reconnect**. Symptom mid-task:
  `ToolSearch "computer-use"` returns nothing. It's independent of the render flags. Mitigation: work
  in **continuous bursts** (no long pauses), and after any lull **re-verify the connector is alive
  before acting**. This idle drop is the single biggest reason to prefer the hand-off pattern below
  for long jobs.
- **Frontmost-app gate.** If a non-allowlisted app (a browser, etc.) is frontmost, clicks on the EDA
  are rejected. Bring the EDA forward with `open_application` and retry (often the 2nd attempt passes).

## 6. When pixel-automation is too slow: hand off + verify

Dozens of component placements and pin-by-pin wires over a flaky connector is the *wrong* tool. The
**high-leverage division of labor** that actually ships:

- **You** produce a precise, netlist-grounded **change list** — per sheet, each item as
  `[location] current → target + the exact GUI action`, with done-markers and a how-to cheat sheet.
- **The user** (fast and fluent in the EDA) executes it.
- **They export a netlist per page and send it to you; you verify it to the net** — confirm each
  intended connection exists, no shorts, no typos, no virtual connections — and point to the exact
  pin/net to fix.

This plays to both strengths: the human is far faster at GUI manipulation; you are far better at
exhaustive, exact connectivity checking. Offer this split as soon as the work turns into many
component placements.

## 7. Netlist verification recipes (shell)

The `.tel` is Allegro Telesis: a `$PACKAGES` block (`footprint ! footprint ! value ; REFDES…`) then
a `$NETS` block (`'NETNAME' ; refdes.pin refdes.pin …`), with indented continuation lines for long
nets. Useful checks:

```bash
# Which net is a given pin on? (and is it where you intended?)
grep -nE "U10\.26( |$)" netlist.tel          # pin 26 of U10 — appears in which net's line?

# Full membership of a net (handles multi-line continuations):
awk '/^.?GND.? ;/{p=1} p{print} p&&/^[^ '\'']/&&!/GND ;/{exit}' netlist.tel \
  | grep -oE "U10\.[0-9]+" | sort -t. -k2 -n      # all U10 pins on GND, sorted

# Did a part-swap wire the SPI correctly? (pin numbers, by net)
grep -E "^'SPI_(MISO|MOSI|SCLK|CS_DRV)'" netlist.tel

# Are rails still separate (no silent merge)?
grep -E "^'?5V_(A|B|SYS)'? ;" netlist.tel        # expect three distinct lines

# Any stray auto-named nets left from a botched placement?
grep -nE "^'?NET[0-9]+'? ;" netlist.tel || echo "clean"

# A pin that's NOT in any net = floating (virtual connection / forgotten):
grep -oE "U10\.[0-9]+" netlist.tel | sort -u     # compare against the chip's pin list
```

Confirm explicitly, per change: the **new** net contains the pins it should, the **old** net no
longer does, and no third net accidentally absorbed anything.

## 8. End-of-job checklist
- [ ] Global **DRC** passes: no unconnected, no single-pin nets, no shorts.
- [ ] Power rails that should be separate are still separate in the netlist.
- [ ] Every "connect X to Y" landed (pin present in the target net) — re-checked from a fresh export.
- [ ] Any swapped part is wired by its **new** pin functions.
- [ ] One final full-project netlist export, diffed against the baseline, reviewed end to end.
</content>

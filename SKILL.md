---
name: easyeda-pro
description: Edit, fix, audit, or reverse-engineer schematics in EasyEDA Pro / LCEDA Pro / JLCEDA / 嘉立创EDA专业版 (立创EDA) — driving it through the official Extension API & WebSocket bridge, a third-party MCP server, or computer-use as a GUI fallback — and verifying every change against the exported netlist instead of the canvas. Use when the user wants to modify a schematic, fix circuit/connectivity defects, rename nets, connect floating/unconnected pins, add / move / delete / replace components, swap a chip part variant, wire SPI/power/ground, ground a thermal pad, or check a design's connectivity in EasyEDA Pro / LCEDA Pro / 嘉立创EDA / JLCEDA / EasyEDA. Encodes the netlist-first verification discipline (works no matter how you edit), the per-pin net-port connectivity model, the official API / MCP options (easyeda-api-skill, pro-api-sdk, jlcmcp, EasyEDA Pro MCP), the delete-no-connect-flag → place-net-port → draw-wire GUI workflow, Allegro .tel netlist export + grep/diff recipes, the .eprj2/.epro2 file formats, computer-use gotchas (stuck tools, occlusion freeze, 900s idle-timeout connector drops), and the "hand the user a precise change-list and verify via netlist" division of labor.
version: 1.1.0
---

# EasyEDA Pro / LCEDA Pro / 嘉立创EDA专业版 — schematic editing & netlist-verified fixes

For editing, fixing, and auditing **schematics** in EasyEDA Pro (international name) =
LCEDA Pro = JLCEDA = **嘉立创EDA专业版** (立创EDA专业版) — JLCPCB's pro EDA tool.

You can drive it **programmatically** (official Extension API / WebSocket bridge, or a third-party
MCP server) or, as a fallback, by **computer-use** on the GUI. Whatever the editing channel, the one
discipline that makes AI-driven schematic work reliable is the same:

## 0. Prime directive: verify with the NETLIST, not your eyes (or the API's "OK")

A change that *looks* applied — on the canvas, or in a successful API call — can still be wrong: a net
port one pixel off a pin doesn't connect; an API edit can land on the wrong net; a part swap silently
remaps pin functions. So, regardless of how you edit:

> **After every change, export the netlist and check connectivity in the exported text — never trust
> the screenshot or a bare "success".**

The netlist is the single source of truth. It catches the failures everything else hides:
- **Virtual connections** — a port/wire/edit that looks attached but isn't (the pin is simply absent
  from the net).
- **Silent net merges** — e.g. one bad import collapsed three separate 5 V rails
  (`5V_A`/`5V_B`/`5V_SYS`) into a single 71-pin net. Invisible on the canvas, obvious in the netlist.
- **Part-swap pin remaps** — a pin-compatible variant keeps pin *numbers* but changes pin *functions*;
  the netlist (pin-number based) shows whether you wired the new function.

Workflow shape for any task, any channel:
1. Open the project. **Export a baseline netlist** and read it — ground truth for "before".
2. Make a small, scoped change.
3. **Re-export the netlist**, grep/diff the affected nets/pins against what you intended.
4. Only move on once the netlist confirms it. Repeat.

## 1. Editing channels — prefer the API/MCP; computer-use is the fallback

EasyEDA Pro **does** have programmatic access (this surprises people — there's no REST API, but there
is an in-app JS Extension API reachable over a local WebSocket bridge). Prefer it:

- **Official Extension API + WebSocket bridge** — the most authoritative path.
  - [`easyeda/easyeda-api-skill`](https://github.com/easyeda/easyeda-api-skill) — EasyEDA's *own* AI
    skill: load the `run-api-gateway.eext` extension, it starts a local bridge on ports
    **49620–49629**, then run any EasyEDA Pro JS API call via `curl -X POST localhost:49620/execute -d
    '{"code":"..."}'`. Works with Claude / the Agent Skills standard.
  - [`easyeda/pro-api-sdk`](https://github.com/easyeda/pro-api-sdk) + docs at
    `prodocs.easyeda.com/en/api/` — the full Extension API surface (project, schematic, PCB,
    components, DRC, export).
  - ⚠️ **macOS bridge gotcha (verified, fixable):** `easyeda-api-skill`'s bridge binds
    **`127.0.0.1` (IPv4 only)** but the in-EDA extension dials `localhost`, which on macOS resolves
    to **IPv6 `::1`** first — so the extension can't find the bridge ("未找到 Bridge 服务 / Bridge not
    found", retries N/5 then gives up) even though `curl http://127.0.0.1:49620/health` works for you.
    Fix `scripts/bridge-server.mjs`: bind **dual-stack** (`LISTEN_HOST = '::'`) **and reject
    non-loopback peers** in the HTTP + WS handlers (so the code-exec endpoint stays local-only), or
    listen on both `127.0.0.1` and `::1`. Verify with `curl` on `127.0.0.1`, `localhost`, and `[::1]`.
    After fixing the bridge, the extension must **re-attempt** once (re-open/re-run it, or restart the
    EDA) — it doesn't auto-rescan after giving up.
- **Third-party MCP servers** (stdio MCP → WebSocket gateway → in-app bridge extension):
  - [`hyl64/jlcmcp`](https://github.com/hyl64/jlcmcp) — ~39 tools incl. **read schematic, export
    netlist, schematic DRC**, component move/place, routing, copper, impedance calc.
  - [`QuincySx/easyeda-pro`](https://www.pulsemcp.com/servers/quincysx-easyeda-pro) — ~72 tools,
    schematic→PCB→manufacturing.
  - [`sengbin/JLCEDA-MCP`](https://github.com/sengbin/JLCEDA-MCP),
    [`txwilliam/LCEDApro-mcp-extension`](https://github.com/txwilliam/LCEDApro-mcp-extension).
- **EasyEDA's in-app AI assistant** also ships an MCP build (real-time schematic refresh/review).

Use **computer-use** (sections 5–6) only when the bridge/MCP isn't set up, for a quick one-off, or for
cross-app workflows. It's slow and brittle (see the gotchas) — set up the API/MCP for anything beyond
a few edits.

> Reality check: these channels let you *make* changes faster, but none of them *verify* connectivity
> for you. Section 0 still applies. The unique value of this skill is the verification discipline +
> mental model + fallback playbook below — it sits on top of whatever channel you use.

## 2. Mental model: per-pin **net ports** define connectivity

In EasyEDA Pro schematics, nets are usually defined by **per-pin net ports / net labels**
(网络端口 / 网络标签) — the flags carrying a net name next to each pin — **not** by long drawn wires.
Two pins share a net because their ports share the same **name**.

Consequences (true for API edits and GUI edits alike):
- **Renaming a net = renaming net ports.** Changing one port's name moves *only that pin*. To rename a
  whole net you must change **every** port on it, or the net splits.
- **Moving a pin to another net = renaming its port** (e.g. move a cap's hot pin from `24V_BUS` to `VM`).
- **Connecting a floating pin = adding a port** (after removing any no-connect flag).
- A net's true membership is whatever the netlist says — always re-derive it from the export.

## 3. File formats (so you know what's editable)

| File | What it is | Editable? |
|---|---|---|
| `*.eprj2` | The project. **SQLite**; schematic data is an **encrypted blob** in `history_data`. | Not directly — edit via the API/MCP or GUI. |
| `*.epro2` | An **export** of the project. A **zip**; inside, the `.epru` is **plaintext** (`head\|\|body\|`, `\n`-separated), round-trips losslessly. | Scriptable, **but re-importing can silently merge nets** — verify hard, or avoid the import path. |
| `*.tel` | A **netlist** export (Allegro Telesis format). Plaintext, lists every net and its pins. | **Read-only ground truth — your verification oracle.** |

⚠️ Component `id`s can repeat **across pages**; if you parse `.epru`, index **per page** (`SCH_PAGE`).

## 4. Operation goals (express via API, MCP, or GUI)

These are the connectivity goals; achieve them through whichever channel, then verify in the netlist.

- **Rename a net / move a pin to another net** — change the pin's net-port name to the target net.
- **Connect a floating pin to a net** — remove its no-connect flag, then give the pin a port on the
  target net.
- **Add / delete / move a component**; **change a value**; **replace / swap a part variant** ⚠️ — after
  a swap, pin *numbers* stay but pin *functions* change (e.g. a motor driver's pins 33–36 go from
  `MODE/SLEW/OCP/GAIN` on the hardware variant to `SDO/SDI/SCLK/nSCS` on the SPI variant). Re-wire by the
  **new** function and verify the netlist pin assignments.
- **Export the netlist** for verification (Allegro `.tel`).

## 5. Fallback channel — driving the GUI with computer-use

When you must use the GUI. Menu names in English (中文).

- **Export netlist**: `Export → Netlist` (导出 → 网表文件) → **Allegro(.tel)** → save to disk. Give each
  export a distinct name; don't overwrite your baseline.
- **Rename a net / move a pin**: click the pin's net port → Properties → **名称 (Name)** → type the new
  net → Enter. Changes only that port; repeat for every port that must move. Then re-export & verify.
- **Connect a floating pin** (robust recipe):
  1. Delete the **no-connect flag** (非连接标识, green ✗): click it → `Del`.
  2. `Place → Net Port` (放置 → 网络端口, the "input" pentagon): single-click *next to* the pin.
  3. Name it: select → Properties → **名称** → target net (`GND`/`AVDD`/`3V3`…).
  4. `Place → Wire` (放置 → 导线): draw from the **pin endpoint** to the **port's connection point**.
  5. `Esc` (if it won't exit, **right-click** to cancel — see gotchas). Re-export & verify.
- **Add a component**: `Place → Component` (放置 → 器件, Shift+F) → search part → place → wire.
- **Batch the select→edit cycle** in one `computer_batch`: `left_click(port)` →
  `triple_click(Properties 名称 field)` → `type(name)` → `key(Return)`; chain several pins per batch.
- **Zoom before clicking tiny targets**; **cross-probe** via the Networks panel (网络) to jump to a pin;
  **save** (`Cmd/Ctrl+S`) before exporting.

## 6. computer-use gotchas (macOS — hard-won)

- **Placement tools get STUCK active.** After `Place → Net Label/Wire`, the tool often **won't exit on
  `Esc`**, and then **every click places another element** (NET1, NET2, … pile up; stray wires grow).
  Fix: **right-click to cancel**, *then* the tool is off and `Cmd+Z` works.
- **`Cmd+Z` follows focus.** If the panel search box has focus, undo edits the search text, not the
  schematic. Click an empty canvas spot to focus first — but only when no placement tool is active.
- **Copy-paste of a net port can spawn junk** (once produced a stray capacitor). Prefer place-port+wire.
- **Occlusion render-freeze.** If the agent's window is fully covered, macOS suspends its painting —
  stale frame, can't read/reply, though input still works. Use a **second display**, or tile so a sliver
  stays visible; never run the target app in true-fullscreen. (Launch flags
  `--disable-renderer-backgrounding --disable-backgrounding-occluded-windows
  --disable-background-timer-throttling` are the deeper fix.)
- **Connector drops after ~15 min idle.** The desktop app's IdleManager tears down computer-use after
  ~900 s idle with no auto-reconnect (`ToolSearch "computer-use"` returns nothing mid-task). Work in
  continuous bursts; re-verify the connector after any lull. This idle drop is a big reason to prefer
  the API/MCP, or the hand-off pattern below, for long jobs.
- **Frontmost-app gate.** If a non-allowlisted app is frontmost, clicks are rejected; bring the EDA
  forward (`open_application`) and retry.

## 7. When automation is slow/flaky: hand off + verify

If the API/MCP isn't set up and the GUI work is dozens of placements over a flaky connector, the
high-leverage split is:
- **You** produce a precise, netlist-grounded **change list** — per sheet, each item
  `[location] current → target + exact action`, with done-markers and a how-to cheat sheet.
- **The user** (fast in the EDA) executes it.
- **They export a netlist per page and send it; you verify it to the net** — every intended connection
  present, no shorts, no typos, no virtual connections — and point to the exact pin/net to fix.

The human is faster at GUI manipulation; you are better at exhaustive, exact connectivity checking.

## 8. Netlist verification recipes (shell)

The `.tel` is Allegro Telesis: a `$PACKAGES` block (`footprint ! footprint ! value ; REFDES…`) then a
`$NETS` block (`'NETNAME' ; refdes.pin refdes.pin …`), with indented continuation lines.

```bash
# Which net is a pin on, and is it where you intended?
grep -nE "U10\.26( |$)" netlist.tel

# Full membership of a net (handles multi-line continuations):
awk '/^.?GND.? ;/{p=1} p{print} p&&/^[^ '\'']/&&!/GND ;/{exit}' netlist.tel \
  | grep -oE "U10\.[0-9]+" | sort -t. -k2 -n

# Did a part-swap wire SPI correctly? (pin numbers, by net)
grep -E "^'SPI_(MISO|MOSI|SCLK|CS_DRV)'" netlist.tel

# Are rails still separate (no silent merge)?
grep -E "^'?5V_(A|B|SYS)'? ;" netlist.tel        # expect three distinct lines

# Any stray auto-named nets from a botched placement?
grep -nE "^'?NET[0-9]+'? ;" netlist.tel || echo "clean"
```

Confirm per change: the **new** net contains the pins it should, the **old** net no longer does, and no
third net absorbed anything.

## 9. End-of-job checklist
- [ ] Global **DRC** passes: no unconnected, no single-pin nets, no shorts.
- [ ] Rails that should be separate are still separate in the netlist.
- [ ] Every "connect X to Y" landed (pin present in the target net) — from a fresh export.
- [ ] Any swapped part is wired by its **new** pin functions.
- [ ] One final full-project netlist export, diffed against the baseline, reviewed end to end.
</content>

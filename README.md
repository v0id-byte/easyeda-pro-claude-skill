# EasyEDA Pro Claude Skill — netlist-verified schematic editing

> A [Claude](https://claude.com/claude-code) **skill** that makes AI-driven schematic editing in
> **EasyEDA Pro / LCEDA Pro / JLCEDA / 嘉立创EDA专业版 (立创EDA)** *reliable* — by **verifying every
> change against the exported netlist instead of trusting the screen or a bare "success".** It works
> on top of whatever editing channel you use: the official Extension API / WebSocket bridge, a
> third-party MCP server, or computer-use on the GUI.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
![Claude Skill](https://img.shields.io/badge/Claude-Skill-8A2BE2)

## What this is (and isn't)

EasyEDA Pro (a.k.a. **LCEDA Pro**, **JLCEDA**, **嘉立创EDA专业版**, **立创EDA** — JLCPCB's professional
EDA tool) already has solid programmatic access and an ecosystem of AI integrations (see
[Editing channels](#editing-channels) below). This skill is **not** another way to send commands — it's
the **discipline layer** that sits on top of any of them:

> **The canvas lies. A "success" can lie. The netlist doesn't. After every edit, export the netlist
> and check connectivity in the text.**

That one rule turns "click/call and hope" into a verifiable engineering loop, and it catches the exact
failures every channel hides:

- **Virtual connections** — a net port one pixel off the pin (or an edit on the wrong net): *looks*
  wired, isn't.
- **Silent net merges** — one bad import collapsed three separate 5 V rails into a single 71-pin net.
- **Part-swap pin remaps** — a pin-compatible variant keeps pin *numbers* but changes pin *functions*.

It complements [`easyeda/easyeda-api-skill`](https://github.com/easyeda/easyeda-api-skill) (EasyEDA's
official API skill) and the MCP servers below — they give an agent the *ability to edit*; this gives it
the *discipline to get connectivity right*, plus a computer-use fallback for when no bridge is set up.

## What's in the skill

- **The netlist-first verification discipline** — export, grep/diff, confirm; channel-agnostic.
- **The per-pin net-port connectivity model** — why "rename a net" means "rename every port on it."
- **Editing-channel guidance** — official API / WebSocket bridge, third-party MCP, or computer-use, and
  when to use which.
- **Computer-use fallback playbook** — battle-tested GUI recipes (connect a floating pin:
  `delete no-connect flag → place net port → draw wire → Esc`; rename nets; swap a part) and the
  hard **macOS gotchas** (stuck placement tools, `Cmd+Z` focus traps, occlusion render-freeze, the
  ~900 s idle-timeout connector drop).
- **The "hand-off + verify" division of labor** — when edits become many, give the human a precise,
  netlist-grounded change list and verify their per-page netlist exports to the net.
- **Shell verification recipes** — `grep`/`awk` over the Allegro `.tel` netlist.
- **File-format notes** — `.eprj2` (encrypted SQLite project), `.epro2` (plaintext export zip), `.tel`
  (netlist ground truth).

## Editing channels

Prefer programmatic control; use computer-use only as a fallback.

| Channel | What | Link |
|---|---|---|
| **Official API skill** | EasyEDA's own AI skill: `run-api-gateway.eext` WebSocket bridge (ports 49620–49629), run any Pro JS API via `curl /execute` | [easyeda/easyeda-api-skill](https://github.com/easyeda/easyeda-api-skill) |
| **Official Extension SDK** | Full Extension API + docs | [easyeda/pro-api-sdk](https://github.com/easyeda/pro-api-sdk) · [prodocs](https://prodocs.easyeda.com/en/api/) |
| **MCP (3rd-party)** | ~39 tools incl. read schematic / export netlist / schematic DRC | [hyl64/jlcmcp](https://github.com/hyl64/jlcmcp) |
| **MCP (3rd-party)** | ~72 tools, schematic→PCB→manufacturing | [QuincySx/easyeda-pro](https://www.pulsemcp.com/servers/quincysx-easyeda-pro) |
| **computer-use** | drive the GUI directly (fallback) | this skill, §5–6 |

## Install

```bash
git clone https://github.com/<you>/easyeda-pro-claude-skill.git
mkdir -p ~/.claude/skills/easyeda-pro
cp easyeda-pro-claude-skill/SKILL.md ~/.claude/skills/easyeda-pro/
```

Activates automatically when you ask Claude to work on an EasyEDA Pro / LCEDA Pro / 嘉立创EDA schematic.
(Global skills live in `~/.claude/skills/`; project skills in `<project>/.claude/skills/`.)

## Usage

Ask in plain language, e.g.:

- "Fix the connectivity defects in this EasyEDA Pro schematic and verify with the netlist."
- "Connect the DRV8316's PGND/AGND pins to GND and confirm in the exported netlist."
- "Swap U10 to the SPI variant, wire SCLK/SDI/SDO/nSCS, and check the pin assignments."
- "Audit this 嘉立创EDA project's power rails — are 5V_A/5V_B/5V_SYS still separate?"

## Requirements

- **Claude Code** (or Claude desktop).
- **EasyEDA Pro / LCEDA Pro / 嘉立创EDA专业版** installed.
- One editing channel: the official API bridge / an MCP server (recommended), or computer-use.
- macOS for the computer-use gotchas; the methodology is OS-agnostic.

## Suggested GitHub topics

`easyeda` · `easyeda-pro` · `lceda` · `lceda-pro` · `jlceda` · `claude` · `claude-code` ·
`claude-skill` · `mcp` · `computer-use` · `eda` · `pcb` · `schematic` · `netlist` ·
`hardware-design` · `ai-eda` · `electronics`

## Contributing

Born from a real engagement (fixing a BLDC motor-control board's schematic). The recipes and gotchas
are what actually worked. New EasyEDA Pro quirks, API/MCP workflow notes, Windows/Linux notes,
English-UI menu paths, and extra netlist checks all welcome — open an issue or PR.

## License

[MIT](LICENSE) — © 2026 Liuhaoran Qin (v0id-byte). Do whatever you want, no warranty.
</content>

# EasyEDA Pro Claude Skill — AI schematic editing & netlist-verified fixes

> A [Claude](https://claude.com/claude-code) **skill** that lets an AI agent edit, fix, and audit
> **schematics in EasyEDA Pro / LCEDA Pro / JLCEDA / 嘉立创EDA专业版 (立创EDA)** by driving the
> desktop app with computer-use — and **verifies every change against the exported netlist instead
> of trusting the screen.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
![Claude Skill](https://img.shields.io/badge/Claude-Skill-8A2BE2)
![Platform: macOS](https://img.shields.io/badge/platform-macOS-lightgrey)

EasyEDA Pro (a.k.a. **LCEDA Pro**, **JLCEDA**, **嘉立创EDA专业版**, **立创EDA专业版** — JLCPCB's
professional EDA tool) has **no public API and no MCP server**. So an AI assistant has to drive its
GUI directly. The naive way — clicking around and trusting the screenshot — fails constantly: ports
that *look* connected aren't, imports silently merge nets, part swaps quietly remap pins. This skill
encodes a **netlist-first methodology** that makes AI-driven schematic editing actually reliable, plus
a hard-won playbook of GUI recipes and computer-use gotchas.

As far as we know this is the **first Claude skill for EasyEDA Pro / 嘉立创EDA**. PRs welcome.

---

## Why this exists

Editing a schematic by pixel-clicking is error-prone. The breakthrough is simple:

> **The canvas lies. The netlist doesn't. After every edit, export the netlist and check
> connectivity in the text — never trust the screenshot.**

That one rule turns a flaky "click and hope" process into a verifiable engineering loop, and it
catches the exact failures a human (or an AI) misses by eye:

- **Virtual connections** — a net port one pixel off the pin: looks wired, isn't.
- **Silent net merges** — a single bad import once collapsed three separate 5 V rails into one net.
- **Part-swap pin remaps** — a pin-compatible variant keeps pin *numbers* but changes pin *functions*.

## What's in the skill

- **The per-pin net-port connectivity model** — how nets are really defined in EasyEDA Pro, and why
  "rename a net" means "rename every port on it."
- **Battle-tested GUI recipes** — rename a net, connect a floating pin
  (`delete no-connect flag → place net port → draw wire → Esc`), add/move/delete/replace components,
  swap a chip variant, export the netlist.
- **computer-use automation patterns** — the `select → Properties name → type → Enter` batch, zoom-to-
  click, cross-probe navigation.
- **macOS computer-use gotchas** — stuck placement tools (right-click to cancel), `Cmd+Z` focus traps,
  occlusion render-freeze, and the **900 s idle-timeout connector drop** — with mitigations.
- **The "hand-off + verify" division of labor** — when the job becomes dozens of placements, give the
  human a precise, netlist-grounded change list and verify their per-page netlist exports to the net.
- **Shell verification recipes** — `grep`/`awk` one-liners over the Allegro `.tel` netlist.
- **File-format notes** — `.eprj2` (encrypted SQLite project), `.epro2` (plaintext export zip),
  `.tel` (netlist ground truth).

## Install

Copy the skill into your Claude skills directory:

```bash
git clone https://github.com/<you>/easyeda-pro-claude-skill.git
mkdir -p ~/.claude/skills/easyeda-pro
cp easyeda-pro-claude-skill/SKILL.md ~/.claude/skills/easyeda-pro/
```

It activates automatically when you ask Claude to work on an EasyEDA Pro / LCEDA Pro / 嘉立创EDA
schematic. (Global skills live in `~/.claude/skills/`; project skills in `<project>/.claude/skills/`.)

## Usage

Just ask, in plain language, e.g.:

- "Fix the connectivity defects in this EasyEDA Pro schematic."
- "Connect the DRV8316's PGND and AGND pins to GND and verify with the netlist."
- "Swap U10 to the SPI variant and wire SCLK/SDI/SDO/nSCS."
- "Audit this 嘉立创EDA project's power rails — are 5V_A/5V_B/5V_SYS still separate?"

Claude will drive the app (computer-use), export the netlist, and verify each change against it — or,
for big jobs, hand you a precise change list and verify your netlist exports.

## Requirements

- **Claude Code** (or Claude desktop) with **computer-use** available.
- **EasyEDA Pro / LCEDA Pro / 嘉立创EDA专业版** installed.
- macOS (the computer-use gotchas are macOS-specific; the methodology is OS-agnostic).

## Suggested GitHub topics (for discoverability)

`easyeda` · `easyeda-pro` · `lceda` · `lceda-pro` · `jlceda` · `嘉立创eda` · `立创eda` · `claude` ·
`claude-code` · `claude-skill` · `computer-use` · `eda` · `pcb` · `schematic` · `netlist` ·
`hardware-design` · `ai-eda` · `electronics`

## Contributing

This started from one real engagement (fixing a BLDC motor-control board's schematic). The recipes
and gotchas are what actually worked. If you hit a new EasyEDA Pro quirk or a better workflow, open an
issue or PR — especially Windows/Linux notes, English-UI menu paths, and additional netlist checks.

## License

[MIT](LICENSE) — do whatever you want, no warranty.
</content>

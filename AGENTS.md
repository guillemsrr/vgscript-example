# VGScript — Agent Guide

This is the **single, IDE-agnostic guide** for AI agents (Claude Code, Cursor,
Copilot, etc.) working in a VGScript **content repo** — a project full of `.vgs`
scripts authored with this plugin.

> **Is the same guide valid in every IDE?** Yes. The VS Code extension and the
> JetBrains plugin are thin wrappers over the **same `vgs` engine and the same
> grammar** (both live in `core/`). Everything an agent needs — the language and
> the `vgs` CLI — is editor-agnostic, and agents drive the CLI directly rather
> than clicking menus. Both plugins even expose the **identical command set**
> (`Fix + Reconcile`, `Validate`, `Fix + Reconcile All`, `Resolve Links`,
> `Reorder Scenes`) under a `VGScript` menu, so the human-facing steps match too.
>
> **How to use it:** drop this file into your content repo as `AGENTS.md` (or
> `CLAUDE.md`). The *Language* and *Tooling* sections below are universal — copy
> them verbatim. Only the *Project context* section is per-project; fill it in.

---

## Project context (fill this in for your repo)

> This section is a placeholder. Replace the prompts below with your project's
> own details, but keep the headings so agents always know where to look.

- **Premise** — one or two sentences on what the game/story is about.
- **Authoring language** — the natural language the prose is written in (and any
  rule, e.g. "keep tags in English, prose in `<language>`").
- **Conventions** — anything non-obvious about tone, naming, or structure an agent
  should preserve.

### Workspace structure (typical — adapt to your repo)

- Scenes are `.vgs` files, each with a matching `<file>.vgs.meta` sidecar.
- Scenes are usually grouped into directories (acts, chapters, etc.) and numbered
  sequentially, e.g. `S1_Intro.vgs`.
- `Scenes.txt` at the project root lists scenes in play order; several `vgs`
  commands read it.

---

## The `vgscript` language

`vgscript` is a lightweight bracket-tagged DSL for narrative scenes: dialogue,
named actions, timed/parallel events, and branching decisions. Every tag sits at
the **start of its line**, in square brackets. The descriptions below match what
the `vgs` parser actually does (`core/tool/src/reconcile.rs`); where a feature is
interpreted by the *game engine* rather than the tool, it is marked
**(engine-level)**.

### Entries vs. context tags

The tool parses a script into **entry groups**. Only four tags open an entry:

| Tag | Entry | Notes |
| --- | --- | --- |
| `[A]` | **Action** | Engine actions/events. Often a scene's `SceneHandler` (an action called via events/triggers). |
| `[T]` | **Talk** | A block of visible narrative / dialogue lines. |
| `[D]` | **Decision** | A player choice with numbered options. |
| `[C]` | **Condition** | A condition block that gates flow **(engine-level)**. |

Two more tags carry **context** but do **not** open an entry of their own:

- **`[S]` — Scene header.** Defines the scene key + name. Written as either
  `[S1_Intro]` (key/name split on the first `_`) or `[S1] Intro` (name after the
  bracket). With neither, the key/name fall back to the filename.
- **`[P]` — Speaker (persona).** ⚠️ This is **who is speaking**, *not* the spoken
  text. The text after `[P]` names the character; an arrow gives a target:
  `[P] Hero` or `[P] Hero -> Villain`. The current speaker (and optional target)
  are attached to the **following `[T]` talks** as `character` / `directedTo` in
  the `.meta`. An empty `[P]` clears the speaker. The actual lines of dialogue
  live in the `[T]` block beneath it.

### Sublines

Numeric sublines `[1] [2] [3] …` belong to the most recent parent entry above
them. A subline may also be written in long form `[D1-2]` — reconcile normalizes
it to `[2]`. `[X]` (or `[x]`) is a placeholder/first subline.

```vgscript
[P] Character
[T1]
[1] So this is where it begins.
[2] Nothing here yet. Only my voice.
```

### Flow control and timing

- **Sequential** — `->(X)`: after this line/action finishes, go to entry `X`.
  Example: `[A1] Intro ->(A2)`.
- **Parallel** — `||(X)`: run `X` concurrently without waiting.
  Example: `[A1] FadeIn ||(Timer{S})`.
- **Cross-script jump** — `->(Sx-Code)`: jump to an entry in another scene. Run
  `vgs resolve-links` to bind these to stable GUIDs (see *Tooling*).
- Everything after `->` or `||` on a line is flow, not visible text — the tool
  excludes it when computing a line's displayed text.

The following are **engine-level** conventions. To the `vgs` tool, any `{…}` is an
opaque inline tag that it strips from visible text; the tool does **not** validate
their meaning:

- **Timers** — `Timer{XS}`, `Timer{S}`, `Timer{M}`, `Timer{L}`: engine-defined
  durations (extra-small … large) for easy global tuning.
- **Events** — an action that calls a sub-action by name, e.g.
  `[A2] Event{A1, "UnlockDoor"}` ("A2 calls the `UnlockDoor` event from A1").
- **Inline markers** — e.g. `{!repeat}`: metadata on the line, stripped from the
  visible/`.meta` text.
- **Magnitude suffixes** — `XS, S, M, L` denote size/scale for timers, text
  blocks, and other magnitude-based parameters.

### Decisions and branching

A decision lists numbered options; each option's arrow says what to start when
chosen:

```vgscript
[D1]
[1] Accept  ->(A_yes)
[2] Decline ->(A_no)
```

The tool understands two things inside options:

- **Copy text from a talk** — `=(Tn-1)` pulls the visible text of `[Tn]` line `[1]`
  into the option. Reconcile materializes it: a line like
  `[1] {!repeat} =(T1-1) 'Old text' ->(T1)` is rewritten to
  `[1] {!repeat} 'Actual talk text' ->(T1)`, and the `.meta` stores the resolved
  text. Author-written quoted/plain text that you *didn't* reference is left
  alone.
- **Conditions** — an option may start with `if(...)`; the `if(...)` is not counted
  as visible option text. (Broader decision queries such as
  `DecisionsCountLess{...}` and `=(Dn-x)` echoes are **engine-level**.)

#### Writing decisions that *feel* like decisions

A decision's job is to give the player a **sensation of agency** — the feeling
they are authoring their own route — even when every branch reconverges on the
same beat. The narrative can (and usually should) funnel back together; what must
**not** collapse is the felt difference between options at the moment of choosing.

The failure mode to avoid is **synonym choices**: two or more options that say the
same thing in different words. They read as a fake decision and break immersion.

> ✗ Anti-pattern:
> ```vgscript
> [D1]
> [1] We should leave.
> [2] It's time to go.
> [3] There's nothing left for us here.
> ```
> All three are the *same intent* reworded, and the flow merges regardless. There
> is no real choice here.

Make options differ in **kind**, not just wording. Good axes of divergence:

- **Attitude / value** — pride vs. doubt, tenderness vs. cruelty, awe vs.
  contempt. The character *reveals who they are* by what they pick.
- **Approach / method** — what they'll *do* next differs, even if it lands in the
  same place (get past the guard *by force* vs. *by persuasion*).
- **Tone / register** — clinical vs. poetic, defiant vs. resigned.
- **What they notice** — each option fixes on a different facet of the scene.

When branches must reconverge, let the choice still leave a **trace**: a changed
line later, a remembered stance, or a different transition into the shared beat.
Reconvergence is fine; an *invisible* choice is not.

Quick self-check before committing a `[Dn]`: *if I swapped two options, would the
player notice a different meaning, mood, or consequence?* If not, rewrite or cut
until each option is a genuinely distinct stance.

### Actions

- Actions are meant to be reusable. **Ask the user before adding a new one.**
- Parameters are usually key-value pairs separated by colons; keep the syntax
  consistent with existing cases.
- A scene normally has at least one `SceneHandler` action — an action designed to
  be called via events and triggers.

### Comments

- `// line comment` — to end of line.
- `/* block comment */` — may span lines.

Both are stripped before parsing, so they never affect entry text or hashes.

### Anchors and references (tool-managed — never hand-write)

- `{#guid:…}` — a stable per-entry ID the tool injects so references survive
  renumbering.
- `{#dst:<guid>/<hash>}` — a resolved cross-script destination (written by
  `resolve-links`).
- A bare 32-char hex hash gets annotated `/*Scene Code*/` (or `/*MISSING*/` if the
  target is gone). Leave these to the tool.

### Meta files

Each `.vgs` has a `<file>.vgs.meta` (JSON) holding scene key/name and per-entry
records (`currentCode`, stable `hash`, visible `lines`, and for talks the
`character`/`directedTo`). **Never hand-edit `.vgs.meta`** — edit the `.vgs` and
reconcile.

---

## Tooling: the `vgs` CLI (how to "reconcile")

**Do not write your own parser or fix scripts, and never hand-edit `.vgs.meta`.**
All `.meta` generation, entry renumbering, decision-text filling, cross-script
link resolution, and validation are owned by the bundled `vgs` CLI. Drive it.

### What "reconcile" means

Reconcile = regenerate a script's `<file>.vgs.meta` so it matches the `.vgs`
source. One pass:

- Parses the script into entry groups (`[A] [T] [D] [C]` with numeric sublines).
- **Preserves stable per-entry IDs** by matching, in order: a `{#guid:…}` anchor →
  the exact entry code → a content hash; a new ID is minted only when nothing
  matches. This is why renumbering a scene does not break references to it.
- Auto-fills decision options that reference a talk via `=(Tn-1)`.
- Annotates cross-script hash references (`/*Scene Code*/` or `/*MISSING*/`).
- Writes/updates `<file>.vgs.meta`, and may lightly rewrite the `.vgs` itself
  (inject `{#guid:…}` anchors, normalize sublines to numeric).

Workflow is always: **edit the `.vgs`, then reconcile to regenerate the `.meta`.**

### How to run it

**In the IDE (JetBrains *or* VS Code, identical):** right-click the `.vgs` (or a
folder, for the project-wide actions) → **VGScript → Fix + Reconcile** (or
**Fix + Reconcile All (Scenes.txt)** for the whole project).

**From a terminal:** `vgs` is the compiled binary. The plugin finds it via
**Settings → Tools → VGScript** (explicit path), then
`core/tool/target/{release,debug}/vgs[.exe]` under the open project, then `PATH`.
A content repo with no Rust sources won't have a local build — set the explicit
path in plugin settings, or put `vgs` on your `PATH`.

| Goal | Command (alias) |
| --- | --- |
| Reconcile one script (renumber, fill, (re)write its `.meta`) | `vgs fix <file.vgs>` |
| Reconcile every scene, in `Scenes.txt` order | `vgs fix-all [dir]` |
| Find mistakes in one script | `vgs find-mistakes <file.vgs>` (`vgs validate`) |
| Find mistakes across all scenes | `vgs check-all [dir]` |
| Resolve cross-script `->(Sx-Code)` links to GUIDs | `vgs resolve-links <dir>` |
| Renumber / reorder scenes from `Scenes.txt` | `vgs reorder-scenes <dir> [--dry-run]` |

`[dir]` defaults to the current directory; pass the folder containing
`Scenes.txt`. `vgs fix` *is* the editor's "Fix + Reconcile" (it enumerates, then
reconciles with subline normalization) — prefer it over the lower-level
`enumerate` / `reconcile` / `batch` subcommands. Run `vgs --help`,
`vgs <command> --help`, or `vgs help-all` for the full spec.

### Typical workflow after writing or editing a scene

1. `vgs fix <file.vgs>` — normalize entries and (re)generate `<file>.vgs.meta`.
2. `vgs find-mistakes <file.vgs>` — confirm it is clean.

For a whole-project pass use `vgs fix-all` then `vgs check-all`.

### Exit codes (branch on these instead of parsing output)

- `0` — success / no problems found.
- `1` — validation problems found (`find-mistakes`, `check-all`).
- `2` — the script or `Scenes.txt` could not be found.

---

## Golden rules for agents

1. **Edit `.vgs`, never `.vgs.meta`** — let `vgs fix` regenerate the sidecar.
2. **Don't reimplement the tool** — no hand-rolled parsers or "fix" scripts.
3. **Reconcile, then validate** after any edit (`vgs fix` → `vgs find-mistakes`).
4. **Ask before adding a new action**; reuse existing ones.
5. **Keep style**: descriptive action names, readable indentation, consistent tag
   casing, no trailing spaces on tag lines.
6. **Make decisions genuine** — distinct in kind, not synonyms (see above).
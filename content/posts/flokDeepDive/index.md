---
title: "flok Deep Dive: Supervising AI Coding Agents Inside tmux"
summary: "How flok layers an agent sidebar onto an existing tmux server without touching it: the two-server design, the hook-driven state machine, the authority merge that reconciles four state sources, and the engineering that keeps it at near-zero CPU across tmux 2.7 to 3.7."
description: "A technical deep dive into flok, W4J's open-source agent sidebar for tmux: architecture, state pipeline, hooks, screen rules, tmux version gating, and why the approach matters for teams running many AI coding agents."
categories: ["Engineering"]
tags: ["Go", "tmux", "AI Agents", "Claude Code", "Developer Tools", "Open Source"]
date: 2026-09-20
draft: false
---

![flok, an agent sidebar for tmux](flok-hero.png)

Teams that adopt AI coding agents seriously end up running several of them at once. A single engineer with Claude Code or GitHub Copilot CLI quickly moves from one conversation to five: one per repository, one per ticket, one doing a long migration in the background. The agents are good at the work. What breaks down is **supervision**: which of the five is waiting for a permission approval, which one finished ten minutes ago, which one is still grinding through a test suite.

[flok](https://github.com/w4jnl/flok) is our answer for the environment we actually work in, which is tmux, on macOS and on Linux servers over ssh. It is a single Go binary (MIT, no cgo) that adds a sidebar to an existing tmux setup, tracks every agent pane through the agents' own hook systems, and surfaces the one signal that matters: *who needs you right now*.

This article is the technical account. If you want the short, user-facing version, it is on [Jaro's personal site](https://jaro.w4j.nl/posts/flokagentsidebar/). Here we go through the architecture, the state pipeline and the engineering decisions, and close with why we think the approach is the right one for organisations, not only individuals.

## Design constraints

flok was built against four hard constraints. They explain nearly every decision below.

1. **Zero footprint on the user's tmux.** No plugin, no injected pane, no config change. Engineers have years of tmux configuration; a tool that requires rewriting it does not get adopted.
2. **Exact state where exact state exists.** Claude Code and Copilot CLI emit lifecycle hooks. If the agent tells us it is waiting for permission, we should not be guessing from screen scraping.
3. **Never lie about attention.** A false "needs you" is worse than a late one. A finished agent that the user was already watching must not beep or show as unread.
4. **Runs everywhere we run.** macOS with a modern terminal, but also RHEL 8 with tmux 2.7 over ssh, with no sound device and no window manager.

## Architecture: two tmux servers

The zero-footprint constraint rules out the obvious design (a tmux plugin or a status-line integration). flok instead runs a second, private tmux server, the **outer**, whose only job is to frame the user's normal server, the **inner**.

{{< mermaid >}}
%%{init: {"theme":"base","themeVariables":{"primaryColor":"#e3f4f5","primaryTextColor":"#14181A","primaryBorderColor":"#12999D","secondaryColor":"#fbe9d4","secondaryBorderColor":"#E8963C","tertiaryColor":"#eef1f1","tertiaryBorderColor":"#8A999C","lineColor":"#8A999C","textColor":"#14181A","nodeTextColor":"#14181A","clusterBkg":"#f3f6f6","clusterBorder":"#8A999C","titleColor":"#14181A","edgeLabelBackground":"#eef1f1","labelBackgroundColor":"#eef1f1","labelColor":"#14181A"}}}%%
flowchart TB
    T["Terminal emulator"]
    subgraph OUTER["outer tmux server (socket flok)"]
        SB["left pane<br/><code>flok sidebar</code><br/>(Bubble Tea)"]
        AL["right pane<br/><code>flok _attach-loop</code> → <code>tmux attach</code>"]
    end
    subgraph INNER["inner tmux server (the user's)"]
        P1["pane: claude"]
        P2["pane: copilot"]
        P3["pane: zsh"]
    end
    T --> OUTER
    AL -- "every key and mouse event, unchanged" --> INNER
    SB -. "reads: list-sessions ; list-panes ; list-clients · capture-pane · list-keys<br/>steers: switch-client · select-window · select-pane" .-> INNER
{{< /mermaid >}}

The outer server has `prefix None`, `status off` and `mouse on`, so it is transparent: every keystroke and mouse event falls through to the inner client running in the right pane. The user's prefix, bindings, copy mode and plugins are untouched because they live in the inner server, which flok only ever *reads* (`list-sessions`, `list-panes`, `list-clients`, `capture-pane`, `list-keys`) and *steers* (`switch-client`, `select-window`, `select-pane`).

The right pane runs an attach loop rather than a bare `tmux attach`, because a client exit is ambiguous. The loop distinguishes the two cases:

| the inner client exited because | attach loop does |
|---|---|
| the session was destroyed (`tmux kill-session`) | re-attaches to another session |
| the user detached deliberately (`prefix d`) | tears the outer server down, `flok up` exits 0 |

That exit code matters for launcher integration: `flok up || tmux attach || tmux new-session` in a terminal profile falls back to plain tmux only when flok genuinely cannot start.

The outer configuration is rendered from a template per tmux version and written to the state directory on every `flok up`. Width pinning is a good example of the kind of detail the template carries: tmux scales panes proportionally on every window resize, which would turn a 28-column sidebar into 100 after a window manager moves the window. The sidebar is the main pane of a `main-vertical` layout, and a `window-resized` hook (or `client-resized` before tmux 3.3) re-applies the layout unless the work pane is zoomed.

### One binary, several roles

The same executable plays every part, selected by subcommand:

| role | command | lifetime |
|---|---|---|
| launcher | `flok up` / `flok down` | renders the outer config, creates the session, writes `runtime.json`, execs `tmux attach` |
| sidebar | `flok sidebar` | long-lived Bubble Tea program in the left pane |
| hook receiver | `flok hook claude` / `flok hook copilot` | one short process per agent event |
| navigation | `flok jump\|next\|prev\|toggle\|hide\|focus` | one-shot, bound in the user's tmux.conf |
| menu bar | `flok-bar` (macOS, separate binary) | reads the published snapshot, forwards clicks to `flok goto` |

`runtime.json` is the contract between them: the outer panes, both sockets, the inner client tty, the terminal app that ran `flok up` and the tmux version. One-shot commands read it instead of forking `tmux -V` or searching for the outer server on every call.

## The state pipeline

Everything the sidebar shows comes out of one function, `merge.Tracker.Build`, fed by four sources of unequal authority.

{{< mermaid >}}
%%{init: {"theme":"base","themeVariables":{"primaryColor":"#e3f4f5","primaryTextColor":"#14181A","primaryBorderColor":"#12999D","secondaryColor":"#fbe9d4","secondaryBorderColor":"#E8963C","tertiaryColor":"#eef1f1","tertiaryBorderColor":"#8A999C","lineColor":"#8A999C","textColor":"#14181A","nodeTextColor":"#14181A","clusterBkg":"#f3f6f6","clusterBorder":"#8A999C","titleColor":"#14181A","edgeLabelBackground":"#eef1f1","labelBackgroundColor":"#eef1f1","labelColor":"#14181A"}}}%%
flowchart LR
    H["<b>Hooks</b><br/>Claude Code / Copilot CLI<br/>→ <code>flok hook</code><br/>→ one JSON record per pane"]
    R["<b>Registry</b><br/><code>claude agents --json</code><br/>busy / idle, matched<br/>pid → tty → pane"]
    Ti["<b>Pane title</b><br/>spinner glyph = working"]
    Sc["<b>Screen rules</b><br/>herdr manifests over<br/>capture-pane + title + OSC 9;4"]
    Tm["<b>tmux snapshot</b><br/>sessions ; panes ; clients<br/>in one call"]
    M["merge.Build"]
    Snap["Snapshot<br/>Spaces · Agents · Focus · NewlySeen"]
    UI["sidebar view"]
    Bar["snapshot.json → flok-bar"]
    Seen["seen marks"]
    Tm --> M
    H -- "fsnotify, instant" --> M
    R -- "10 s, only while useful" --> M
    Ti --> M
    Sc -- "2 s" --> M
    M --> Snap
    Snap --> UI
    Snap --> Bar
    Snap -- "NewlySeen" --> Seen
{{< /mermaid >}}

### Hooks: the source of truth

`flok install` registers the receiver in Claude Code's `settings.json` for ten events, and in `~/.copilot/hooks/flok.json` for Copilot's equivalents:

```
SessionStart  UserPromptSubmit  PreToolUse  PostToolUse  PostToolUseFailure
PermissionRequest  Notification  Stop  StopFailure  SessionEnd
```

Each entry is written idempotently (the installer finds its own entries by the ` hook claude` marker in the command string, repairs a stale binary path, and backs the file up before changing it) and marked `async` with a five-second timeout, so a slow hook can never stall the agent.

The receiver has three rules: **fast, silent, exit 0**. Copilot CLI denies the tool call when a hook fails, so a panic inside the receiver is recovered and still exits 0. On each call it:

1. reads the pane from `TMUX_PANE` (no pane means no tmux, and nothing to track);
2. maps the raw JSON through the agent adapter into an agent-neutral event;
3. takes a per-pane `flock`, applies the pure state machine, writes the record with temp-and-rename;
4. plays a sound if the effects say so, and appends a line to `events.log`.

The adapter is where agent-specific knowledge lives. For Claude Code, for instance, a `PreToolUse` for `AskUserQuestion` is a *block* with reason `question`, not a tool start; a `Notification` of type `permission_prompt` is a block, `idle_prompt` is a nudge, and `agent_completed` is ignored because it describes a background session rather than this pane. Sub-agent events (payloads carrying an `agent_id`) never change a pane's state.

### The state machine

The machine itself is a pure function, `Apply(agent, event, focused, now) → Effects{Sound, Delete}`. It never touches files, which is what makes it testable without tmux. The interesting transitions:

{{< mermaid >}}
%%{init: {"theme":"base","themeVariables":{"primaryColor":"#e3f4f5","primaryTextColor":"#14181A","primaryBorderColor":"#12999D","secondaryColor":"#fbe9d4","secondaryBorderColor":"#E8963C","tertiaryColor":"#eef1f1","tertiaryBorderColor":"#8A999C","lineColor":"#8A999C","textColor":"#14181A","nodeTextColor":"#14181A","clusterBkg":"#f3f6f6","clusterBorder":"#8A999C","titleColor":"#14181A","edgeLabelBackground":"#eef1f1","labelBackgroundColor":"#eef1f1","labelColor":"#14181A"}}}%%
flowchart LR
    start(( )) -- "SessionStart" --> idle
    idle -- "prompt submitted" --> working
    working -- "tool start / end<br/>Stop with sub-agents in flight" --> working
    working -- "permission · question · elicitation" --> blocked
    blocked -- "tool end · next prompt" --> working
    working -- "Stop, pane <b>not</b> focused<br/>sound · unread +1" --> done
    working -- "Stop, pane focused<br/>silent" --> idle
    blocked -- "Stop, not focused" --> done
    blocked -- "Stop, focused" --> idle
    done -- "the user looks at the pane" --> idle
    idle -- "SessionEnd<br/>record deleted" --> stop(( ))
    classDef working fill:#d9f5f6,stroke:#3FD0D4,stroke-width:2px,color:#14181A
    classDef blocked fill:#fbe9d4,stroke:#E8963C,stroke-width:2px,color:#14181A
    classDef done fill:#dff3e5,stroke:#5FC47A,stroke-width:2px,color:#14181A
    classDef idle fill:#eef1f1,stroke:#8A999C,stroke-width:2px,color:#14181A
    classDef term fill:#8A999C,stroke:#8A999C
    class working working
    class blocked blocked
    class done done
    class idle idle
    class start,stop term
{{< /mermaid >}}

Three details carry most of the "never lie about attention" constraint:

- **`done` exists only when the pane is not focused.** `focused` is passed as a lazy callback because answering it costs a tmux call (`session_attached window_active pane_active` on the inner server plus a terminal-focus check), and most events do not need it.
- **Duplicate blocks are collapsed.** A `PermissionRequest` and the `Notification` for the same `tool_use_id` count as one block: one sound, one unread mark.
- **A Stop with background sub-agents still running stays `working`** with reason `waiting`. The turn is not over from the user's point of view until the sub-agents report back.

Each block or unseen completion appends a timestamped notification to the record; the unread count on a `done` row is simply the number of notifications newer than the pane's *seen* mark.

### The authority merge

Hooks are exact but not complete. A turn interrupted with `Esc`, a usage-limit stop, a crash, or an agent started before `flok install` all leave the record saying `working` with no event to close it. The merge exists to reconcile that, with strict rules about who may overrule whom.

| pane has | state comes from | other sources may |
|---|---|---|
| hook record | the record | clear a stale `working` after **two** consecutive idle registry samples, or **three** idle-screen polls (**six** while the registry still says busy); clear a stale `blocked` after **two** idle-screen polls |
| no hooks | title → registry → screen rules, first that answers | nothing to overrule |

Two guards keep the escape hatches narrow. Evidence must be *newer* than the state it overrules: samples taken before the record's `StateSince` (plus a small grace period for the registry and the screen to catch up) do not count, and a new prompt resets every counter. And the idle *title* is deliberately not evidence at all: inside tmux, Claude Code keeps the `✳` glyph in the title while busy, so only the spinner glyphs mean anything.

The counts were tuned against real captures. The 0.4.3 release, for example, raised the screen-idle threshold to six samples while the registry reports busy because Claude Code's own spinner sometimes draws `✳`, which three unlucky captures in a row read as a bare prompt box. The bundled manifest was updated to the herdr version that knows the glyph at the same time.

The merge also handles pane reuse. If a pane whose record says `idle` or `done` is now running `ssh` or `nvim` in the foreground, the agent exited without a `SessionEnd`; the record is reported stale and removed rather than shown as a ghost row. While `working` or `blocked` the record is trusted, because an agent may legitimately put any tool in the foreground.

### Screen rules

For agents without hooks (Codex, Gemini, OpenCode) and as the fallback above, flok embeds a port of herdr's manifest engine. The manifests are TOML files, copied verbatim from herdr under Apache-2.0, describing regions of the screen and matchers over the visible pane text, the pane title and tmux's OSC 9;4 progress state. They load in layers, bundled → herdr's own cache (opt-in) → `~/.config/flok/agents/`, later replacing earlier by id, so a user can override one agent's rules without forking.

Captured screens live as test fixtures, and `TestClaudeFixtures` pins which rule wins for each of them. `flok explain <pane>` runs the same evaluation against a live pane, which is how a misdetection is diagnosed in the field.

## Keeping it at zero CPU

A sidebar that polls tmux once a second, captures every agent pane and forks `claude agents --json` would be visible in `top`. The sidebar's poll model was rewritten in 0.2.2 and 0.2.3 around a few rules that are easy to undo by accident, so they are worth stating explicitly.

{{< mermaid >}}
%%{init: {"theme":"base","themeVariables":{"primaryColor":"#e3f4f5","primaryTextColor":"#14181A","primaryBorderColor":"#12999D","secondaryColor":"#fbe9d4","secondaryBorderColor":"#E8963C","tertiaryColor":"#eef1f1","tertiaryBorderColor":"#8A999C","lineColor":"#8A999C","textColor":"#14181A","nodeTextColor":"#14181A","clusterBkg":"#f3f6f6","clusterBorder":"#8A999C","titleColor":"#14181A","edgeLabelBackground":"#eef1f1","labelBackgroundColor":"#eef1f1","labelColor":"#14181A"}}}%%
flowchart LR
    tick["1 s tick"] --> snap["one tmux call:<br/>sessions ; panes ; clients"]
    snap --> fp{"fingerprint of raw output<br/>+ hook records changed?"}
    fp -- no --> hb["keep cached frame<br/>heartbeat snapshot.json"]
    fp -- yes --> build["merge.Build"]
    fs["fsnotify:<br/>hook record written"] --> rebuild["rebuild() over the<br/>cached tmux snapshot"]
    scr["2 s: captureAll<br/>(one tmux fork for all panes)"] --> rebuild
    reg["10 s: registry,<br/>only while a turn is open<br/>or a pane lacks hooks"] --> rebuild
    rebuild --> build
    build --> render["render at 15 fps cap"]
{{< /mermaid >}}

- **Only the 1 s tick spawns tmux.** Screen results, registry samples and hook changes go through `rebuild()`, which re-merges the cached tmux snapshot instead of taking a new one.
- **An unchanged poll costs no merge.** The fingerprint covers the raw tmux output, the hook records, the seen marks and the terminal focus flag. If nothing changed, the cached frame stays and only the menu bar heartbeat fires.
- **All screen captures go in one tmux invocation.** One fork for N panes instead of N forks.
- **The registry is queried only when the answer can be used.** `claude agents --json` costs about 0.2 s of CPU; it runs every 10 s only while a Claude turn or prompt is open or a pane lacks hooks, and once a minute otherwise.
- **The renderer is capped at 15 fps** instead of Bubble Tea's default 60, and the spinner runs at 4 fps.
- **Idle mode.** When the sidebar is hidden (`prefix B`), the spinner stops and polls and captures stretch to `idle_poll_ms`. An unfocused terminal window is deliberately *not* idle: it is usually still visible on another monitor.
- **Sounds play in the hook process by default**, not the sidebar, so a notification never waits for a render cycle.

`FLOK_CPUPROFILE` in the sidebar's environment writes a 30 s pprof profile after start, which is how these numbers were obtained and are re-checked.

## Running on every tmux we have to support

The fleet we operate spans tmux 2.7 (RHEL 8) through 3.7 (Homebrew). tmux does not fail on an unknown option in a config file; it dumps the error into a view-mode overlay on the pane, which the user has to dismiss. That makes version gating a correctness issue, not a nicety.

flok parses `tmux -V` once into a `Features` struct, records it in `runtime.json`, and renders the outer config template against it:

| feature | needs | fallback below |
|---|---|---|
| `display-popup` for the keybinds help | 3.2 | help opens in a new window |
| popup border and title | 3.3 | bare popup |
| `allow-passthrough` for the inner server's apps | 3.3 | none |
| `client-focus-in/out` hooks | 3.3 | done/idle assumes the terminal is focused |
| `window-resized` hook | 3.3 | `client-resized` |
| vis(3)-escaped `list-*` output | 3.4 | raw output; `tmux.Decode` undoes the escaping only when this is set |
| `extended-keys-format csi-u` | 3.5 | none |

Two things bit us during this work. tmux 3.4 started escaping non-printable bytes in `list-*` output, so the field separator flok used arrived as the octal string `\037`; every snapshot now goes through a decoder gated on the version, and nothing splits raw output directly. And `-e VAR=val` on pane and popup commands exists only in 3.0 to 3.3, so environment for the sidebar and the popup goes through an `env …` prefix instead. Each rendered template is pinned by a test, and CI runs the end-to-end suites on macOS 3.7, Ubuntu 3.4, Rocky 9 3.2a and Rocky 8 2.7.

The end-to-end suites deserve a mention because they are what made the version matrix tractable. A library script builds `bin/flok` and a fake agent binary named `claude` that paints scripted screens, starts isolated tmux servers with a private state and config directory, replays hook payloads against the fake agent's pane, and asserts on `capture-pane` output of the sidebar. Seven suites cover title-driven states, hook-driven states, navigation and keys, screen rules, the launcher lifecycle, the snapshot and menu bar plumbing, and the terminal bell. The developer's real tmux server is never touched.

## Notifications that reach you over ssh

The sound path is a small composition: a `Player` that picks the first of `afplay`, `mpv`, `ffplay`, `pw-play`, `paplay`, `play` on `PATH` (or a configured command), a `Bell` that writes `BEL` into the outer's work pane, and a `Compose` that combines them per mode. With `[sounds] bell = "auto"`, a headless or remote host with no player rings the bell; the outer server has `bell-action any`, so the BEL reaches the terminal emulator, which turns it into the user's local notification, across ssh, with nothing installed remotely. The bundled sounds are WAV because the PulseAudio players on RHEL 9 cannot decode mp3.

## The menu bar companion

On macOS a separate binary, `flok-bar`, puts the flok mark in the menu bar: a spinner while an agent works, an orange badge with a count while one waits, and a dropdown of agents. It is the only cgo package in the tree (`fyne.io/systray`) and is kept out of `internal/` so the sidebar remains a pure, static Go binary everywhere else.

It never talks to tmux. The sidebar publishes its merged view to `snapshot.json` after every merge (on change, plus a 5 s heartbeat), the bar renders it through pure, tested functions, and a click runs `flok goto <pane>`: switch the inner client, mark the pane seen, and bring the terminal window to the front through AeroSpace when installed or AppleScript otherwise. The bar quits by itself 30 s after `runtime.json` and the snapshot disappear, so `flok down` leaves no orphan.

## Restoring agents with tmux-resurrect

tmux-resurrect restores panes and directories but only relaunches allow-listed programs, and adding `claude` to the list restarts the CLI without selecting the same conversation. flok's opt-in integration hooks tmux-resurrect's post-save event and rewrites only the Claude and Copilot process fields in the new state file:

- Claude Code becomes `claude --resume <exact-session-id>`, taken from `claude agents --json` (which describes the live process) and from the hook record as a fallback.
- Copilot CLI becomes `copilot --resume=<exact-session-id>` from its hook record.
- A recognised agent without a usable ID is saved with **no** process, so the restored pane is a shell rather than the wrong conversation.

tmux-resurrect still owns pane creation, layout and process launch, so there is no post-restore race and no dependency on pane ids being reused. `flok up` also starts the inner server with a throwaway session named `~flok`, a name no saved session can be mistaken for, so a tmux-continuum restore finds every pane free.

## Why this matters for an organisation

The individual benefit is obvious: nobody leaves an agent waiting on a permission prompt for twenty minutes. The organisational case is about the shape of the design rather than the sidebar itself.

- **Adoptable without a migration.** flok wraps the tmux an engineer already has. There is no new terminal, no new multiplexer, no config rewrite. Trying it costs `brew install` and `flok up`; removing it costs `flok down`.
- **Exact state, from the agents' own contract.** Hooks are a supported interface of Claude Code and Copilot CLI. Screen scraping is the fallback, not the foundation, and the fallback is the same rule engine herdr maintains, so new agents arrive as manifests rather than code.
- **Auditable.** Every hook event and the state it produced is appended to `events.log` as JSON lines; every pane has a readable record; `flok status --json` and `flok explain` show what the sidebar sees and why. Debugging a misdetection is reading a file.
- **Works where the work is.** The same binary, the same behaviour, on a developer's Mac and on a RHEL 8 jump host over ssh, with notifications that reach the local terminal through the bell when there is no sound device.
- **Cheap to run.** Near-zero CPU at rest, one tmux fork per second, no daemon beyond the sidebar pane. It sits in the background all day without being noticed.
- **Small and inspectable.** One Go module, no cgo outside the optional menu bar, MIT licensed, with unit tests that construct the merge inputs directly and end-to-end suites on isolated servers across four tmux versions.

flok is a personal tool published because there was no reason not to, and the README says so plainly: features follow our workflow, macOS is first-class, Linux is the terminal sidebar without the menu bar, and there is no compatibility promise between versions yet. What we can say is that the approach, a transparent outer server plus hook-driven state with a disciplined merge, has held up across a year of daily use with many agents in parallel, and that it is the pattern we would reach for again.

```sh
brew install w4jnl/tap/flok      # macOS; Linux tarballs on the releases page
flok install && flok doctor      # hooks, config, tmux snippet, version report
flok up                          # from a plain terminal
```

{{< github repo="w4jnl/flok" >}}

If your team is putting AI coding agents into production workflows and wants help with the tooling around them, from supervision to CI integration and the infrastructure they run on, [talk to us](mailto:info@w4j.nl).

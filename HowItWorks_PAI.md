# How It Works: Personal AI Infrastructure Repository

## Scope of this analysis

This repository is not a single app source tree. It is primarily a release catalog of complete `.claude/` distributions for multiple PAI versions, plus root-level docs and maintenance utilities.

I read the full tracked repository (4,404 files), then traced executable entrypoints and runtime wiring.

---

## Repository structure

Top-level:

- `.claude/` (only a small root helper script in this repo snapshot)
- `Releases/` (the actual product payloads by version)
- `Tools/` (repo maintenance/security utilities)
- `images/` (documentation assets)
- root docs and policies (`README.md`, `SECURITY.md`, `PLATFORM.md`, `.pai-protected.json`, etc.)

Release payload sizes:

- `Releases/v2.3`: 950 files
- `Releases/v2.4`: 1,052 files
- `Releases/v2.5`: 1,111 files
- `Releases/v3.0`: 1,250 files

The active/current architecture is centered on `Releases/v3.0/.claude/`.

---

## What this system is

PAI is a Claude Code-centric infrastructure layer that adds:

- installation workflow and bootstrapping
- persistent settings/config
- lifecycle hooks (session/tool events)
- skill system (`skills/*`)
- agent definitions (`agents/*`)
- memory directories (`MEMORY/*`)
- optional voice server
- status line and terminal orchestration

At runtime, Claude Code is still the host. PAI configures and extends it.

---

## Startup flow (v3.0, current)

### 1) Install entry

Typical documented install path:

- Copy `Releases/v3.0/.claude` into `~/.claude`
- Run installer shell entrypoint:
  - `~/.claude/install.sh` (forwarder)
  - which executes `~/.claude/PAI-Install/install.sh`

Key files:

- `Releases/v3.0/.claude/install.sh`
- `Releases/v3.0/.claude/PAI-Install/install.sh`
- `Releases/v3.0/.claude/PAI-Install/main.ts`

### 2) Bootstrap installer

`PAI-Install/install.sh`:

- detects OS
- checks/install prerequisites (`curl`, `git`, `bun`, `claude` checks)
- resolves installer directory
- launches Bun entrypoint:
  - `bun run .../main.ts --mode gui`

### 3) Installer mode router

`PAI-Install/main.ts` routes execution:

- `--mode cli` -> terminal wizard
- `--mode web` -> Bun HTTP/WebSocket server
- default `gui` -> Electron wrapper that runs web mode

### 4) Web installer orchestration

`PAI-Install/web/server.ts` hosts UI + websocket.

`PAI-Install/web/routes.ts` handles messages and runs sequential install actions:

1. `runSystemDetect`
2. `runPrerequisites`
3. `runApiKeys`
4. `runIdentity`
5. `runRepository`
6. `runConfiguration`
7. `runVoiceSetup`
8. `runValidation`

Action implementations are in:

- `PAI-Install/engine/actions.ts`
- `PAI-Install/engine/detect.ts`
- `PAI-Install/engine/validate.ts`
- `PAI-Install/engine/state.ts`
- `PAI-Install/engine/steps.ts`

---

## What installer writes/configures

Primarily via `runConfiguration` and `runVoiceSetup`:

- merges user-specific fields into `~/.claude/settings.json` (preserves template hooks/statusline/etc.)
- creates/updates env file in `~/.config/PAI/.env`
- creates symlink targets for env visibility (`~/.claude/.env`, `~/.env` when safe)
- updates algorithm `LATEST` file in skill tree
- initializes counts used by banner/statusline
- writes shell alias for `pai` (zsh and fish handling)
- starts/configures voice server if enabled

---

## Post-install launch flow

### `pai` command

Installer config points alias `pai` to:

- `~/.claude/skills/PAI/Tools/pai.ts`

`pai.ts` then:

- displays banner
- optionally sends startup voice notification
- launches Claude Code (`claude` process)

So control passes:

`pai` alias -> `pai.ts` -> `claude`

---

## Runtime control plane: settings + hooks

The central runtime wiring is in:

- `Releases/v3.0/.claude/settings.json`

This file configures:

- permissions (`allow`, `ask`)
- hook subscriptions by lifecycle event
- status line command
- environment variables

### Lifecycle event chain (v3.0)

When Claude runs a session, configured hooks fire in this order by event type.

#### SessionStart

- `StartupGreeting.hook.ts`
- `LoadContext.hook.ts`
- `CheckVersion.hook.ts`

#### UserPromptSubmit

- `RatingCapture.hook.ts`
- `AutoWorkCreation.hook.ts`
- `UpdateTabTitle.hook.ts`
- `SessionAutoName.hook.ts`

#### PreToolUse

- `VoiceGate.hook.ts` (Bash voice curl gate)
- `SecurityValidator.hook.ts` (Bash/Edit/Write/Read)
- `SetQuestionTab.hook.ts` (AskUserQuestion)
- `AgentExecutionGuard.hook.ts` (Task tool)
- `SkillGuard.hook.ts` (Skill tool)

#### PostToolUse

- `QuestionAnswered.hook.ts` (AskUserQuestion)
- `AlgorithmTracker.hook.ts` (Bash/TaskCreate/TaskUpdate/Task)

#### Stop

- `StopOrchestrator.hook.ts`
  - delegates to handlers:
    - `VoiceNotification.ts`
    - `TabState.ts`
    - `RebuildSkill.ts`
    - `AlgorithmEnrichment.ts`
    - `DocCrossRefIntegrity.ts`

#### SessionEnd

- `WorkCompletionLearning.hook.ts`
- `SessionSummary.hook.ts`
- `RelationshipMemory.hook.ts`
- `UpdateCounts.hook.ts`
- `IntegrityCheck.hook.ts` (runs system/doc integrity handlers)

This is the core "what gets called next by what" chain for normal operation.

---

## Core subsystems and how they connect

### 1) Skill system

Primary skill entry in current release:

- `skills/PAI/SKILL.md`

Related structure:

- `skills/*` (many domain skills)
- `skills/PAI/Components/*` + `Tools/RebuildPAI.ts`
  - components are source fragments
  - rebuild tool assembles generated `SKILL.md`

`LoadContext.hook.ts` injects core context from settings-configured files, making skill docs operational at session start.

### 2) Algorithm/session state subsystem

State file model:

- `MEMORY/STATE/algorithms/{sessionId}.json`

Managed by:

- `hooks/lib/algorithm-state.ts`
- writers:
  - `AlgorithmTracker.hook.ts` (real-time transitions/criteria/agents)
  - `AlgorithmEnrichment.ts` (stop-time enrichment/finalization)

Also connected to:

- `skills/PAI/Tools/algorithm.ts` (loop/interactive algorithm CLI and PRD operations)

### 3) Work/memory subsystem

Key flow:

- `AutoWorkCreation.hook.ts` creates session/task workspace under `MEMORY/WORK/*`
- `SessionSummary.hook.ts` finalizes/cleans state on session end
- `WorkCompletionLearning.hook.ts` converts finished work signals into learning artifacts in `MEMORY/LEARNING/*`
- ratings and signals feed statusline/learning trend calculations

### 4) Terminal/UI subsystem

- status line command: `statusline-command.sh`
- tab color/title control through `hooks/lib/tab-setter.ts` and tab handlers
- startup banner via `StartupGreeting` -> PAI banner tools

### 5) Voice subsystem

Server:

- `VoiceServer/server.ts` (Bun HTTP server, default port 8888)

Primary endpoints:

- `POST /notify`
- `GET /health`

Control scripts:

- `VoiceServer/install.sh`, `start.sh`, `stop.sh`, `restart.sh`, `status.sh`, `uninstall.sh`

Hooks/CLI call into voice server via HTTP, not by embedding TTS logic directly.

---

## Version-to-version architecture progression

### v2.3

- installer via `install.ts`
- CORE naming (`skills/CORE/...`)
- hook set includes format/rating capture split and subagent output capture
- voice server TypeScript/Bun stack in release payload

### v2.4

- installer renamed to `PAIInstallWizard.ts`
- algorithm-focused docs and format reminder evolution
- similar hook topology, still CORE-centered in many paths

### v2.5

- installer renamed to `INSTALL.ts`
- adds hooks like `RelationshipMemory` and `SoulEvolution`
- voice stack in this release is Python-based files (`server.py`, `tts_engine.py`, etc.)
- skill naming converges toward `PAI` core path

### v3.0 (current architecture)

- full installer platform (`PAI-Install/` with engine/web/cli/electron)
- richer hook mesh with guards, algorithm tracking, integrity checks
- consolidated stop orchestration + handler model
- stronger algorithm state persistence and PRD-oriented loop tooling
- current voice server back to Bun/TS implementation in release payload

---

## Root-level files: purpose in whole-app design

- `README.md`: high-level concept + install instructions pointing to releases
- `Releases/README.md`: release catalog and recommended install path
- `SECURITY.md`: public-repo safety and injection defense guidance
- `PLATFORM.md`: platform support matrix and compatibility notes
- `.pai-protected.json`: manifest/pattern policy for protected/sensitive content scanning
- `Tools/validate-protected.ts`: pre-commit style scanner against sensitive patterns
- `Tools/BackupRestore.ts`: backup/restore/migration utility for `~/.claude`
- `.github/workflows/*.yml`: CI automations using Claude Code GitHub action

These are governance/operations scaffolding around the release artifacts.

---

## Embedded sub-apps/tooling islands inside releases

Across versions, there are additional standalone apps/templates:

- Observability dashboard (client/server apps)
- Telos Next.js templates (dashboard/report)
- skill-specific tool packages (Browser, Apify, Prompting templates, Remotion tools, WebAssessment bug bounty tool, etc.)

They are mostly optional capability modules packaged within release trees, not required for the core install/boot path.

---

## End-to-end execution summary (current v3.0)

1. User installs release to `~/.claude`.
2. `install.sh` -> `PAI-Install/install.sh` -> `main.ts`.
3. Installer engine runs 8 setup phases; writes `settings.json`, env, alias, optional voice config.
4. User runs `pai`.
5. `pai.ts` renders banner, then spawns `claude`.
6. Claude runtime loads hooks from `settings.json`.
7. Hooks govern each lifecycle event:
   - context injection
   - security gating
   - tab/voice updates
   - algorithm state tracking
   - work/memory/learning persistence
   - integrity/counts maintenance
8. Voice server and statusline remain external services/scripts called by hooks/CLI.

That is the operational design and call graph of this repository’s application model.

---

## Subsystem Deep Dive Addendum

This section expands the four subsystems requested, with implementation-level flow and what each is used for.

### 1) Skill system

#### What it is used for

The skill system is the capability layer. It stores structured operational knowledge that tells Claude/PAI:

- what domain capability exists
- when to use it
- what workflows/tools/docs to follow
- how behavior should stay consistent

In practice, skills replace ad-hoc prompting with reproducible, file-backed guidance.

#### Primary entry and composition

Current core entry:

- `skills/PAI/SKILL.md`

This file is the primary injected context anchor for the system and includes the global algorithm/process contract used during response generation.

Related structure:

- `skills/*` contains many domain skills (Research, Browser, RedTeam, etc.)
- each skill typically has:
  - `SKILL.md` as entry spec
  - optional `Workflows/*`
  - optional `Tools/*`
  - optional references/templates/data

#### Generated core skill model (PAI Components)

For the core PAI skill, `SKILL.md` is generated from component fragments:

- source fragments: `skills/PAI/Components/*`
- assembler: `skills/PAI/Tools/RebuildPAI.ts`

How this works:

1. Rebuild tool reads component markdown files in numeric order.
2. It resolves template variables from `settings.json` (DA/principal names, voice ids, etc.).
3. It loads versioned algorithm content from `Components/Algorithm/LATEST`.
4. It writes the assembled output to `skills/PAI/SKILL.md`.

This gives a clean source-of-truth split:

- components are editable source
- `SKILL.md` is generated artifact used at runtime

#### Runtime activation path

Skill context becomes active via startup hook:

- `hooks/LoadContext.hook.ts`

Startup behavior:

1. Reads `settings.json` `contextFiles` list.
2. Loads files in that order (default includes core skill and steering rules).
3. Emits merged content as system reminder context for Claude session start.

So operationally:

- skills are inert files on disk until `LoadContext` injects selected entries into live session context.

#### Why this design exists

- deterministic boot context instead of manually pasting instructions
- editable modular source with generated unified runtime file
- easy versioning and diffing of behavior contracts
- allows domain extension by adding new skills without rewriting core runtime

---

### 2) Algorithm/session state subsystem

#### What it is used for

This subsystem tracks the state of algorithmic execution per Claude session:

- current phase (OBSERVE, THINK, PLAN, etc.)
- tracked criteria/anti-criteria
- active/completed agents
- effort level/SLA
- phase history and completion summaries

It enables:

- real-time UI/state awareness
- continuity through long or iterative tasks
- post-response enrichment and lifecycle closure

#### State model

State files are session-scoped:

- `MEMORY/STATE/algorithms/{sessionId}.json`

This avoids one-global-file races and allows concurrent session tracking.

#### Core state library

Single source of write logic:

- `hooks/lib/algorithm-state.ts`

It owns:

- read/write helpers
- phase transitions
- criteria add/update logic
- agent registration
- end-of-algorithm enrichment/finalization helpers
- stale-session sweep logic

This is important because multiple hooks/handlers interact with the same state.

#### Live writer: `AlgorithmTracker.hook.ts`

Trigger points:

- PostToolUse on Bash, TaskCreate, TaskUpdate, Task

What it does:

1. Detects phase transitions from structured tool input patterns (not transcript rescans).
2. Detects criteria creation via `TaskCreate`.
3. Detects criteria status changes via `TaskUpdate`.
4. Tracks agent spawn/use via Task tool data.
5. Ensures session state is active/reactivated as needed.

This gives near-real-time algorithm state mutation as tools execute.

#### Stop-time writer: `AlgorithmEnrichment.ts`

Called from `StopOrchestrator`.

What it does:

1. Parses completed response content for richer metadata:
   - task description
   - summary
   - effort level
   - quality gate/capability hints
2. Calls `algorithmEnd(...)` style finalization in state library.
3. Runs stale-session cleanup/sweep.
4. Has compaction guards so mid-session compaction does not falsely finalize active work.

This complements tracker writes by adding semantic summary at response boundaries.

#### CLI integration: `skills/PAI/Tools/algorithm.ts`

`algorithm.ts` is not just docs tooling; it is an active controller for PRD-based execution:

- loop mode
- interactive mode
- PRD status/pause/resume/stop

Connection to state subsystem:

- creates/updates loop session state entries in `MEMORY/STATE/algorithms/*`
- syncs criteria progress from PRD checkboxes back into algorithm state
- writes session naming and loop metadata used by dashboards/status surfaces

Net effect:

- hooks drive reactive session-state tracking
- `algorithm.ts` drives explicit orchestrated runs
- both converge on the same state store

---

### 3) Work/memory subsystem

#### What it is used for

This subsystem captures operational history and turns work into persistent learning signals.

It supports:

- session/task continuity
- structured work artifacts
- end-of-session summarization
- extraction of reusable learnings
- sentiment/ratings trend tracking

#### Key workflow path

##### A) Session/task scaffolding

- `AutoWorkCreation.hook.ts` on `UserPromptSubmit`

It creates/maintains workspace under `MEMORY/WORK/*`, including:

- session folder
- task subfolders
- metadata files
- task pointer/current task state
- PRD template creation hooks in newer architecture

It writes state files in `MEMORY/STATE/*` so later hooks know active work context.

##### B) End-of-session closure

- `SessionSummary.hook.ts` on `SessionEnd`

It finalizes work status, updates completion markers, and cleans/rolls state references.

##### C) Learning extraction

- `WorkCompletionLearning.hook.ts` on `SessionEnd`

It evaluates completed work metadata and writes learning artifacts into:

- `MEMORY/LEARNING/*`

These records capture what worked/failed and categorize signals (e.g., algorithm vs system learnings).

##### D) Ratings/sentiment signal stream

User feedback and inferred sentiment are written into signal files (notably ratings jsonl streams). Those signals feed:

- statusline trend display
- learning signal summaries
- quality feedback loops for future sessions

#### Important design choices

- session-scoped state files reduce collisions
- work files are explicit and inspectable on disk
- learnings are generated from evidence, not just ephemeral chat context
- session end is the primary consolidation checkpoint

---

### 4) Terminal/UI subsystem

#### What it is used for

This subsystem provides operational visibility while work happens:

- live statusline metrics
- tab title/color phase signaling
- startup identity/banner output

It is not a separate GUI app; it is terminal-first instrumentation around Claude sessions.

#### A) Status line: `statusline-command.sh`

Configured in `settings.json` under `statusLine`.

What it does:

1. Reads Claude-provided JSON context from stdin.
2. Reads local state/settings/memory caches.
3. Computes display mode by terminal width.
4. Renders multi-line operational status including:
   - greeting/identity
   - model/version/context
   - git/work stats
   - learning/rating trends
   - optional environment signal slices (location/weather/usage caches where configured)

It aggressively uses short-lived caches to avoid expensive calls on every render.

#### B) Tab state control: `hooks/lib/tab-setter.ts` + handlers

Primary writers:

- `UpdateTabTitle.hook.ts` (working-phase updates after user prompt)
- `SetQuestionTab.hook.ts` (question state before AskUserQuestion)
- `QuestionAnswered.hook.ts` (returns to working state)
- `handlers/TabState.ts` via `StopOrchestrator` (completion state)
- `LoadContext`/`Startup` flows reset or preserve state depending on compaction/session logic

Mechanics:

- uses Kitty remote control APIs when available
- persists per-window tab state files under `MEMORY/STATE/tab-titles/*`
- maps algorithm phases to color/symbol constants in `tab-constants.ts`

Design goal:

- at-a-glance understanding of what each terminal tab/session is currently doing.

#### C) Startup banner path

Startup flow:

1. `StartupGreeting.hook.ts` runs on SessionStart.
2. It invokes banner tooling (PAI banner script/tool in skill tools).
3. Banner prints identity/system counts/version-oriented startup display.

This confirms initialization and exposes key stats immediately when a session opens.

#### Why this subsystem matters operationally

- improves situational awareness in multi-tab/multi-session workflows
- reduces need to inspect logs/files manually for routine state
- turns algorithm/work state into visible UI cues without building a heavyweight frontend



› How should my code look like to emulate hooks?


• import asyncio
  import json
  from dataclasses import dataclass, field
  from pathlib import Path
  from typing import Any, Awaitable, Callable, Dict, List, Optional, Tuple

  MODEL = "gpt-5-codex"
  SANDBOX = "workspace-write"
  APPROVAL = "default"

  HookFn = Callable[[Dict[str, Any]], Awaitable[None]]


  @dataclass
  class HookBus:
      handlers: Dict[str, List[HookFn]] = field(default_factory=lambda: {
          "on_start": [],
          "on_prompt": [],
          "on_tool_pre": [],
          "on_tool_post": [],
          "on_stop": [],
          "on_end": [],
          "on_error": [],
      })

      def on(self, event: str, fn: HookFn) -> None:
          self.handlers.setdefault(event, []).append(fn)

      async def emit(self, event: str, payload: Dict[str, Any]) -> None:
          for fn in self.handlers.get(event, []):
              await fn(payload)


  def _map_event(raw: Dict[str, Any]) -> Optional[Tuple[str, Dict[str, Any]]]:
      """
      Map codex JSON events -> hook events.
      Adjust these keys to match actual --experimental-json output.
      """
      t = raw.get("type") or raw.get("event")

      # examples; tune to real schema
      if t == "prompt_submitted":
          return "on_prompt", raw
      if t == "tool_call_started":
          return "on_tool_pre", raw
      if t == "tool_call_finished":
          return "on_tool_post", raw
      if t in ("response_completed", "run_stopped"):
          return "on_stop", raw
      return None


  async def run_codex(
      prompt: str,
      *,
      hooks: HookBus,
      cwd: Optional[Path] = None,
      extra_args: Optional[List[str]] = None,
  ) -> Tuple[str, Dict[str, Any]]:
      cmd = [
          "codex",
          "exec",
          "--experimental-json",
          "--model", MODEL,
          "--sandbox", SANDBOX,
          "--config", APPROVAL,
          "--skip-git-repo-check",
      ]
      if extra_args:
          cmd.extend(extra_args)
      cmd.append(prompt)

      context: Dict[str, Any] = {
          "prompt": prompt,
          "cmd": cmd,
          "cwd": str(cwd) if cwd else None,
          "events": [],
          "stdout_lines": [],
          "stderr_lines": [],
          "final_text": "",
          "returncode": None,
      }

      await hooks.emit("on_start", context)

      proc = await asyncio.create_subprocess_exec(
          *cmd,
          cwd=str(cwd) if cwd else None,
          stdout=asyncio.subprocess.PIPE,
          stderr=asyncio.subprocess.PIPE,
      )

      async def read_stdout() -> None:
          assert proc.stdout
          while True:
              line = await proc.stdout.readline()
              if not line:
                  break
              s = line.decode("utf-8", errors="replace").rstrip("\n")
              context["stdout_lines"].append(s)

              try:
                  evt = json.loads(s)
                  context["events"].append(evt)
                  mapped = _map_event(evt)
                  if mapped:
                      name, payload = mapped
                      await hooks.emit(name, payload)

                  # optional: collect assistant text if present
                  if "output_text" in evt:
                      context["final_text"] += evt["output_text"]
              except json.JSONDecodeError:
                  # if some lines are plain text
                  context["final_text"] += s + "\n"

      async def read_stderr() -> None:
          assert proc.stderr
          while True:
              line = await proc.stderr.readline()
              if not line:
                  break
              context["stderr_lines"].append(line.decode("utf-8", errors="replace").rstrip("\n"))

      try:
          await asyncio.gather(read_stdout(), read_stderr())
          rc = await proc.wait()
          context["returncode"] = rc
          await hooks.emit("on_end", context)
          return context["final_text"], context
      except Exception as e:
          context["error"] = str(e)
          await hooks.emit("on_error", context)
          raise


  # ----------------- Example hooks -----------------

  async def log_start(ctx: Dict[str, Any]) -> None:
      print(f"[start] {ctx['cmd']}")

  async def log_tool_pre(evt: Dict[str, Any]) -> None:
      print(f"[tool pre] {evt}")

  async def log_tool_post(evt: Dict[str, Any]) -> None:
      print(f"[tool post] {evt}")

  async def log_end(ctx: Dict[str, Any]) -> None:
      print(f"[end] rc={ctx['returncode']}")

  # Usage:
  # hooks = HookBus()
  # hooks.on("on_start", log_start)
  # hooks.on("on_tool_pre", log_tool_pre)
  # hooks.on("on_tool_post", log_tool_post)
  # hooks.on("on_end", log_end)
  # text, meta = await run_codex("fix failing tests", hooks=hooks)

  If you want hard pre/post control even when Codex JSON lacks tool events, the next step is a proxy command
  (safe_bash) that emits on_tool_pre/on_tool_post itself.
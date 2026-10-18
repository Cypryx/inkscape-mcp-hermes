# inkscape-mcp for AI Agents — Master Roadmap

**Goal:** give any MCP-capable AI agent the ability to create, edit, analyze, and export vector graphics through Inkscape, using the open-source [`sandraschi/inkscape-mcp`](https://github.com/sandraschi/inkscape-mcp) server.

**Scope:** a local **stdio** MCP server running on the same machine as the agent. One machine per pass through this roadmap; repeat it for each machine.

**Framework:** framework-agnostic. Any agent framework that can launch a stdio MCP server works. Examples use **Hermes Agent**, the framework this roadmap is tested with, plus the common JSON `mcpServers` format used by many other clients.

| | |
|---|---|
| Upstream | `sandraschi/inkscape-mcp` (MIT) |
| Pinned commit | `b1f5d64fb0d2f7c61e584205c07d0c44dc64eead` (v2.7.1, 2026-10-12) — **candidate**: becomes the tested pin when Phase 5 passes |
| Tested with | Hermes Agent on Windows (an older upstream commit, July 2026). The pinned commit is not yet tested. Linux not yet tested. macOS **untested** |

### Document conventions

- This roadmap is the **plan**. Once a build starts, the roadmap is not edited. What actually happened, including every deviation, goes in a separate **build log** for that build.
- Placeholders are written as `<like-this>`:

| Placeholder | Meaning |
|---|---|
| `<repo-path>` | Absolute path to your clone of inkscape-mcp |
| `<agent-config>` | Absolute path to your agent framework's MCP config file |
| `<server-name>` | The name you give the server in your agent config (this roadmap uses `inkscape`) |
| `<inkscape-path>` | Absolute path to the Inkscape executable (only if auto-detection fails) |
| `<output-dir>` | A dedicated folder for files the agent creates |

- ⚠ marks a step or fact that has not yet been verified against the pinned commit. Confirm it on your first run and record the result in your build log.

---

## Phase 0 — Prerequisites

**Goal:** confirm the machine is ready before installing anything.

### 0.1 Inkscape 1.2+ (1.4+ recommended)

Inkscape 1.2+ is needed for the Actions API; 1.4+ is recommended.

| OS | Version check |
|---|---|
| Windows | `& "C:\Program Files\Inkscape\bin\inkscape.com" --version` |
| Linux | `inkscape --version` |
| macOS ⚠ | `/Applications/Inkscape.app/Contents/MacOS/inkscape --version` |

> **Windows:** check with `inkscape.com`, not `inkscape.exe`. The `.exe` is the GUI launcher and prints **nothing** for `--version` or `--help`, which looks like a broken install but isn't.

**Pass:** a version string such as `Inkscape 1.4.2 (...)`.

### 0.2 Python 3.12+, `uv`, and `git`

```bash
uv --version
git --version
uv python list          # shows interpreters uv can see or manage
```

If `uv` is missing, install it from <https://docs.astral.sh/uv/>. You don't need a system Python 3.12: `uv python install 3.12` gives you one that uv manages. This avoids conflicts on machines that already have several Pythons (conda, the Microsoft Store, system packages).

### 0.3 libmagic (Linux and macOS only) ⚠

The server depends on `python-magic`, which needs the system `libmagic` library.

| OS | Package |
|---|---|
| Debian/Ubuntu | `libmagic1` |
| Arch-based | `file` (usually already installed) |
| macOS | `brew install libmagic` |

Windows uses a bundled binary instead (`python-magic-bin`), so nothing is needed there.

### 0.4 Your agent framework

- Confirm it supports **stdio MCP servers** (a `command`, `args`, and `env` definition).
- Find its MCP config file, which becomes `<agent-config>`.
  - **Hermes Agent:** the docs say `~/.hermes/config.yaml`. On at least one Windows install it was at `%LOCALAPPDATA%\hermes\config.yaml`. If the documented path doesn't exist, search for `config.yaml` under your Hermes install folder.
- Find out how your framework **reloads** MCP servers (for Hermes, `/reload-mcp` in a session, or restart the gateway).

### 0.5 Back up the agent config

```bash
# Linux/macOS
cp "<agent-config>" "<agent-config>.bak-$(date +%Y%m%d-%H%M%S)"
```
```powershell
# Windows
Copy-Item "<agent-config>" "<agent-config>.bak-$(Get-Date -Format yyyyMMdd-HHmmss)"
```

**Phase 0 done when:** Inkscape, uv, and git all report versions, the agent config is located, and a backup exists.

---

## Phase 1 — Install the server

**Goal:** a working server you've tested on its own, before any agent touches it.

### 1.1 Clone and check out the pinned commit

```bash
git clone https://github.com/sandraschi/inkscape-mcp.git <repo-path>
cd <repo-path>
git checkout b1f5d64fb0d2f7c61e584205c07d0c44dc64eead
```

This leaves the clone in "detached HEAD" state. That's expected for a pinned build.

### 1.2 Pin Python and sync

```bash
uv python pin 3.12      # writes .python-version so every uv run uses the same interpreter
uv sync
```

**Why pin:** on machines with several Pythons, an unpinned `uv run` can pick a different interpreter from the one the venv was built with, and then rebuild the venv on every launch. Pinning once in the repo avoids that. It also means you don't need a `UV_PYTHON` variable in the agent config. If you set one anyway, it must match this pin.

### 1.3 Smoke test the CLI

```bash
uv run --directory <repo-path> inkscape-mcp --help
```

**Pass:** usage text showing `--mode {stdio,http,dual}`, `--config`, `--port`, `--host`, and `--log-level`.

### 1.4 Stdio handshake test (no agent involved) ⚠

This proves the server speaks MCP over stdio and lists its tools. If this fails, the problem is the server. If this passes but the agent can't see the tools, the problem is the agent config.

Save as `smoke_stdio.py` **outside the repo**:

```python
import asyncio, sys
from fastmcp import Client
from fastmcp.client.transports import StdioTransport

async def main():
    transport = StdioTransport(
        command=sys.executable,
        args=["-m", "inkscape_mcp.main", "--mode", "stdio"],
    )
    async with Client(transport) as client:
        tools = await client.list_tools()
        for tool in sorted(t.name for t in tools):
            print(tool)
        print(f"TOTAL: {len(tools)}")

asyncio.run(main())
```

Run it with the server's own environment:

```bash
uv run --directory <repo-path> python /path/to/smoke_stdio.py
```

**Pass:** a list of tool names ending in `TOTAL: <n>`. Record the list in your build log; Phase 4 uses it.
**Fail:** read the server's log output (it writes to stderr), then check Phase 0.3 (libmagic) and Phase 2.

**Phase 1 done when:** `--help` works and the handshake lists the tools.

---

## Phase 2 — Inkscape detection

**Goal:** the server finds the right Inkscape executable.

### 2.1 How detection works (at the pinned commit)

1. If `INKSCAPE_PATH` is set and points to a valid executable, it's used.
2. Otherwise the server checks per-OS locations:
   - **Windows:** registry → `Program Files` → per-user installs → Microsoft Store → `PATH`
   - **Linux:** `PATH` → `/usr/bin`, `/usr/local/bin`, `/snap/bin`, Flatpak, `~/.local/bin`
   - **macOS:** `/Applications/Inkscape.app/...` → Homebrew paths → `PATH`

> **Note:** the repo's `config.yaml` is only read when the server is launched with `--config <file>`. In the default launch it's ignored. To override the Inkscape location, use `INKSCAPE_PATH`, not `config.yaml`.

### 2.2 Decide whether you need `INKSCAPE_PATH`

You need it only if Inkscape is installed somewhere not listed above (a portable install, a custom prefix, or several versions installed side by side). If so, note `<inkscape-path>` for Phase 3.

### 2.3 Confirm detection

Start the server briefly and look for the detection line in its log output:

```bash
uv run --directory <repo-path> inkscape-mcp --mode stdio
```

**Pass:** a log line like `Found Inkscape at: <path>`. Stop the server with Ctrl+C.
**Fail:** the line `Inkscape not found. Running in limited functionality mode` means you need `INKSCAPE_PATH`. Set it in your shell and repeat the test.

---

## Phase 3 — Connect to the agent

**Goal:** the agent launches the server and registers its tools.

### 3.1 The server definition (any framework)

| Field | Value |
|---|---|
| command | `uv` |
| args | `run`, `--directory`, `<repo-path>`, `inkscape-mcp`, `--mode`, `stdio` |
| env | `INKSCAPE_PATH: <inkscape-path>` **only if** Phase 2 required it |
| timeout | 120 seconds or more (Inkscape exports can be slow) |

> **Why `--mode stdio` explicitly:** without it, the server reads `MCP_TRANSPORT` and defaults to stdio. Passing the flag removes any dependence on environment variables, which some frameworks don't forward (next note).

> **Environment isolation:** many frameworks (including Hermes) pass only the env vars you declare, plus a small safe baseline, to stdio servers. Your shell's environment is **not** inherited. Anything the server needs must go in `env`.

### 3.2 Edit the config safely

1. Confirm the Phase 0.5 backup exists.
2. **Edit by hand in a text editor.** Don't use regex replacements or scripts that insert text into the file: they can create duplicate or orphaned blocks. Don't round-trip the whole file through a YAML library either: that can drop comments and reorder keys.
3. **If your framework watches its config file** (a running Hermes gateway applies MCP changes within about a minute), edit a **copy**, validate it, and then replace the original. That way a half-finished edit is never live.
4. **Validate the syntax** using the server's own environment (`pyyaml` is already installed there):

   ```bash
   # YAML
   uv run --directory <repo-path> python -c "import sys,yaml; yaml.safe_load(open(sys.argv[1], encoding='utf-8')); print('YAML OK')" "<agent-config>"
   # JSON
   uv run --directory <repo-path> python -m json.tool "<agent-config>" > /dev/null && echo "JSON OK"
   ```

5. **Check for duplicate keys.** YAML parsers usually accept a duplicated key silently and keep only the last one, so step 4 won't catch it. Count occurrences of the server name; there must be exactly one.

   ```bash
   grep -cE '^\s+inkscape:' "<agent-config>"                                         # Linux/macOS → 1
   ```
   ```powershell
   (Select-String -Path "<agent-config>" -Pattern '^\s+inkscape:').Count            # Windows → 1
   ```

**Windows paths in YAML:** use single quotes (`'C:\path\to\repo'`) or forward slashes (`C:/path/to/repo`). In double quotes, every backslash must be doubled.

### 3.3 Example — Hermes Agent (tested)

```yaml
mcp_servers:
  inkscape:
    command: uv
    args:
      - run
      - --directory
      - <repo-path>
      - inkscape-mcp
      - --mode
      - stdio
    # env:
    #   INKSCAPE_PATH: <inkscape-path>    # only if Phase 2 required it
    timeout: 120
```

Add the `inkscape:` block **under the existing `mcp_servers:` key**. Never add a second `mcp_servers:` key.

> Avoid `hermes mcp add` with `--env` flags for this server. It was observed dropping `--env` values (on another server's install). Edit the file directly.

### 3.4 Example — JSON `mcpServers` clients ⚠

Many other clients use this shape. Check your client's docs for its exact file location and keys.

```json
{
  "mcpServers": {
    "inkscape": {
      "command": "uv",
      "args": ["run", "--directory", "<repo-path>", "inkscape-mcp", "--mode", "stdio"]
    }
  }
}
```

### 3.5 Reload and confirm registration

Reload your agent's MCP servers (for Hermes: restart the gateway if needed, then `/reload-mcp`). Then ask the agent:

> List the tools you have from the inkscape MCP server.

**Pass:** the agent lists the same tools as Phase 1.4, possibly with a framework prefix. Hermes, for example, names them `mcp_inkscape_<tool>`.
**Fail →** work through these in order:
- Re-run Phase 1.4. If it fails, this is a server problem; if it passes, it's a config problem.
- Re-validate the config (Phase 3.2, steps 4–5).
- Check the agent's logs for a connect error. Hermes, for example, names failed servers in its startup summary in `agent.log`, and `hermes mcp test <server-name>` reports what the server returned.

---

## Phase 4 — Tool surface review and restriction

**Goal:** expose only the tools the agent should have. Do this **before** heavy use.

### 4.1 Inventory

Compare the tool list you recorded in Phase 1.4 against this classification for the pinned commit. If a tool isn't listed here, classify it yourself before allowing it.

| Risk class | Tools | Why it matters |
|---|---|---|
| **Read-only** | `inkscape_analysis`, `inkscape_validation` | Inspect files; no writes |
| **File-writing** | `inkscape_file`, `inkscape_vector`, `inkscape_render`, `inkscape_layers`, `inkscape_animation`, heraldry tools | Create or overwrite files. `inkscape_file` and `inkscape_vector` are marked destructive |
| **System / process control** | `inkscape_system`, `inkscape_shutdown`, `llm_ops` | `inkscape_system` can download and install third-party Inkscape extensions, and has a self-terminate operation. `inkscape_shutdown` kills the server. `llm_ops` can switch or **unload local Ollama models**, which affects other workloads on the machine |
| **External network** | `inkscape_fleet`, `inkscape_fab_art`, `inkscape_sim_art`, `list_local_models` | Push files to external services (GIMP, Blender, Unity, robotics endpoints) or probe localhost model servers |
| **Sampling-dependent** ⚠ | agentic and generation tools (names vary by version, e.g. `generate_svg`) | Ask *your agent's* model to generate content. This costs tokens and depends on the client supporting MCP sampling |

### 4.2 Recommended baseline allowlist

Use an **allowlist** (include), not a blocklist. Tools that upstream adds later then stay hidden until you review them.

```
inkscape_file
inkscape_vector
inkscape_analysis
inkscape_render
inkscape_validation
inkscape_layers
inkscape_animation
```

The sampling-dependent tools are **opt-in**. If you enable them, also apply your framework's sampling limits (rate, tokens, timeout).

**Hermes example:**

```yaml
  inkscape:
    # ...command/args from Phase 3.3...
    tools:
      include:
        - inkscape_file
        - inkscape_vector
        - inkscape_analysis
        - inkscape_render
        - inkscape_validation
        - inkscape_layers
        - inkscape_animation
      prompts: false      # optional: hide prompt utility wrappers
      resources: false    # optional: hide resource utility wrappers
```

Other frameworks: use your client's per-server tool filter. If it has none, consider an MCP proxy that filters tools.

### 4.3 Diagnostics without permanent exposure

`inkscape_system` is useful for troubleshooting (`status`, `diagnostics`, `version`). Add it to the allowlist **temporarily** when diagnosing a problem, then remove it.

**Phase 4 done when:** after a reload, the agent lists only the allowlisted tools.

---

## Phase 5 — Verification

**Goal:** prove the full path works: agent → MCP → server → Inkscape → file on disk. Run each test as a separate prompt.

| # | Test (prompt to the agent) | Pass | Fail → fix |
|---|---|---|---|
| 5.1 | "List your inkscape tools." | Exactly the Phase 4 allowlist | Recheck the `include` list and reload |
| 5.2 | "Create a 100×100 red square SVG at `<output-dir>/test_square.svg`." | The file exists and opens in Inkscape | The agent picked the wrong tool, or path permissions → check that `<output-dir>` exists and is writable |
| 5.3 | "Export `<output-dir>/test_square.svg` to PNG at 96 DPI." | A PNG exists and shows a red square | This proves Inkscape is reachable. If it fails: Phase 2 detection, the CLI flag issue in Appendix A, or the server's log output |
| 5.4 | "Analyze `<output-dir>/test_square.svg` and report its dimensions." | Reports 100×100 | Read-only path broken → check the server's log output |
| 5.5 | "Analyze `<output-dir>/does_not_exist.svg`." | A clean error, and the agent carries on | A hang or crash → note it in the build log as a server robustness issue |
| 5.6 | *(you, not the agent)* After the tests, check for leftover Inkscape processes | None left, other than any Inkscape window you opened yourself | Note it in the build log; the server may be orphaning CLI processes |

Process check for 5.6:

```powershell
Get-Process | Where-Object { $_.Name -like "inkscape*" }      # Windows
```
```bash
pgrep -fl inkscape                                             # Linux/macOS
```

**Phase 5 done when:** all six pass. Record the pinned commit as **tested** for this OS in your build log.

---

## Phase 6 — Agent guidance

**Goal:** the agent uses the tools well, not just correctly.

Add guidance to your agent's system prompt or skills covering:

- **When to use Inkscape:** precise vector work, exports, format conversion, and SVG inspection. When not to: photographic or raster image generation, which belongs to image-generation tools.
- **Where files go:** always write into `<output-dir>` and never overwrite source files unless told to. Some frameworks also let you set the server's working directory (Hermes has a `cwd` key); point it at `<output-dir>`.
- **Validate before mutating:** call `inkscape_analysis` or `inkscape_validation` before editing an unfamiliar file.
- **The vision loop:** after an edit, export a preview with `inkscape_render export_preview` and, if the agent's model can see images, inspect the result before reporting done.
- **Report paths:** always tell the user the absolute path of every file created.

---

## Phase 7 — Optional and deferred

These are not part of the baseline build. Each one needs its own security review before use.

- **HTTP or dual mode, and the web dashboard:** the server can also serve HTTP with a REST bridge for a browser dashboard. Note that `--host` defaults to `0.0.0.0` (all network interfaces); bind to `127.0.0.1` unless you intend network access.
- **The Tauri desktop app and the Windows installer** from upstream releases.
- **Local LLM generation through Ollama or LM Studio** (`list_local_models`, `llm_ops`).
- **Fleet pipelines:** handoff to GIMP, Blender, Unity, and robotics tooling.
- **Several machines or agents** sharing one design workflow.

---

## Phase 8 — Extending: a companion MCP server

**Goal:** add your own Inkscape-related tools without modifying upstream.

**Pattern:** keep upstream unmodified at its pinned commit. Build new tools in a **separate MCP server**, in your own repository and under your own license. The agent loads both servers side by side.

- **No merge conflicts:** upstream updates never collide with your code.
- **Clear licensing:** your code carries your license; upstream stays MIT.
- **Tool naming:** give the companion server a distinct name in the agent config (e.g. `inkscape-extras`) and distinct tool names, so tools never collide.
- **Talking to Inkscape:** call the Inkscape CLI directly (respecting the same `INKSCAPE_PATH` convention), or operate on SVG files that upstream's tools produce.
- **Limitation:** a companion server can only **add** tools. It can't change upstream's existing tools.

**When you need to change upstream's own code** (for example, to fix a bug in one of its tools): create a proper GitHub fork, make the fix on a branch, and open a pull request upstream. Keep using the pinned upstream commit until the fix is merged and you've re-verified a new pin.

The companion server gets its own roadmap when work starts.

---

## Appendix A — Known issues and limitations

Status values: **Fixed upstream**, **Open**, or **Unverified** (re-test against the pinned commit and record the result).

| Issue | Status | Notes |
|---|---|---|
| Detector rejected Inkscape: validation checked for `"gimp"` in the path | **Fixed upstream** | Fixed in the pinned commit, which checks for `"inkscape"` |
| Server forced HTTP transport, ignoring `MCP_TRANSPORT` | **Fixed upstream** | With no `--mode` flag, the pinned commit honors `MCP_TRANSPORT` and defaults to stdio. This roadmap passes `--mode stdio` anyway |
| Export failed on Inkscape 1.4.2: the server passed `--no-remote-resources`, which that Inkscape version rejected | **Unverified** | Reported in July 2026 against an older commit. Phase 5.3 re-tests it |
| `generate_svg` failed asking for a "context" parameter | **Unverified** | Seen in July 2026 against an older commit, with Hermes. Hermes supports MCP sampling by default, so the cause is unknown. Re-test only if you enable sampling tools |
| Repo `config.yaml` ignored in the default launch | **By design** | Only loaded with `--config`. Use `INKSCAPE_PATH` instead |
| Windows `inkscape.exe` prints nothing on the console | **Inkscape behavior** | Use `inkscape.com` for CLI checks |
| `--host` defaults to `0.0.0.0` in HTTP/dual mode | **By design** | Doesn't apply to stdio. See Phase 7 |

---

## Appendix B — Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| The agent lists no inkscape tools | Server failed to start, config not reloaded, or filter too strict | Phase 1.4 → Phase 3.2 checks → reload → check the `include` names |
| The agent stopped responding after a config edit | Invalid or duplicated config, applied live by a config watcher | Restore the backup, then redo Phase 3.2 on a copy |
| Tools listed, but every Inkscape operation fails | Inkscape not detected | Phase 2.3; set `INKSCAPE_PATH` |
| The venv rebuilds on every launch | Python version mismatch between the venv and the requested interpreter | Phase 1.2 pin; remove any `UV_PYTHON` that doesn't match |
| `ModuleNotFoundError` or import errors when run from an agent shell | The agent's shell exports its own `PYTHONPATH` | Run manual tests with `PYTHONPATH` unset |
| Linux/macOS crash mentioning `magic` | `libmagic` missing | Phase 0.3 |
| `inkscape --version` prints nothing (Windows) | Ran the `.exe` | Use `inkscape.com` |

---

## Appendix C — Per-OS reference

| | Windows | Linux | macOS ⚠ |
|---|---|---|---|
| Typical Inkscape path | `C:\Program Files\Inkscape\bin\inkscape.exe` | `/usr/bin/inkscape` | `/Applications/Inkscape.app/Contents/MacOS/inkscape` |
| CLI version check | `inkscape.com --version` | `inkscape --version` | full path + `--version` |
| Extra system dependency | none | `libmagic` | `libmagic` (Homebrew) |
| Process check | `Get-Process` | `pgrep -fl inkscape` | `pgrep -fl inkscape` |
| Hermes config (observed) | `%LOCALAPPDATA%\hermes\config.yaml` | `~/.hermes/config.yaml` | `~/.hermes/config.yaml` ⚠ |

---

## Appendix D — Bumping the pinned commit

Upstream is very active. Never move to a new commit without re-verifying it.

1. Read upstream's changelog and commits since the current pin. Look for new tools, renamed tools, and transport or detection changes.
2. `git fetch && git checkout <new-commit> && uv sync` in a **second clone**, not the live one.
3. Repeat Phase 1.4 and compare the tool list with the old one. Classify any new tool (Phase 4.1).
4. Point the agent at the second clone, then run Phase 5 in full.
5. Only then make the new commit the pin, and record the bump in your build log.

---

## Appendix E — Build status template

Copy this into your build log, with one row per machine. Don't use real names; label machines generically.

| Machine | OS | Pinned commit | P0 | P1 | P2 | P3 | P4 | P5 | P6 |
|---|---|---|---|---|---|---|---|---|---|
| machine-1 | | | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ |
| machine-2 | | | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ |

# Inkscape MCP × Hermes — Build Roadmap

Wiring [`sandraschi/inkscape-mcp`](https://github.com/sandraschi/inkscape-mcp) into Hermes Agent as a local stdio MCP server, on two independent installs: Windows and CachyOS.

**Scope:** MCP server only. Web dashboard / Tauri desktop app explicitly deferred — undocumented in the repo's published docs, not required for agent tool access, can be revisited later.

**Mode:** Two fully separate Hermes installs (no shared config). Each phase below is run twice, once per machine.

---

## Phase 0 — Prerequisites Check

Confirm the environment before touching any config.

- [x] Inkscape installed on Windows (`C:\Program Files\Inkscape\bin\inkscape.exe`)
- [x] Inkscape installed on CachyOS
- [ ] Confirm Inkscape version is 1.4+ on both machines (needed for full Actions API support; 1.2+ works with limitations)
- [ ] Confirm Python 3.12+ available on both machines
- [ ] Confirm `uv` available, or plan to install it
- [ ] Confirm `inkscape-mcp/config.yaml` exists and has the correct Inkscape executable path (see Phase 1, Step 1.1a)

---

## Phase 1 — Windows Setup

### Step 1.1 — Verify Inkscape CLI

On Windows, Inkscape is not necessarily on PATH. The `inkscape-mcp` server is configured via its own `config.yaml` rather than relying on PATH resolution. Verify the executable exists at the expected location:

```powershell
Test-Path "C:\Program Files\Inkscape\bin\inkscape.exe"
```

Expected: `True`

### Step 1.1a — Verify inkscape-mcp config.yaml

The repo ships with `config.yaml` at its root. This file tells the MCP server where Inkscape lives. If it's missing or has the wrong path, the server starts fine but every tool call fails with "Inkscape not found."

Check the current value:

```powershell
Get-Content "C:\Users\smeefer\inkscape-mcp\config.yaml" | Select-String "inkscape_executable"
```

Expected output:
```
inkscape_executable: "C:\\Program Files\\Inkscape\\bin\\inkscape.exe"
```

If missing or wrong, set it:

```powershell
(Get-Content "C:\Users\smeefer\inkscape-mcp\config.yaml" -Raw) -replace 'inkscape_executable:.*', 'inkscape_executable: "C:\\Program Files\\Inkscape\\bin\\inkscape.exe"' | Set-Content "C:\Users\smeefer\inkscape-mcp\config.yaml" -NoNewline
```

### Step 1.2 — Verify/install `uv`

```powershell
uv --version
```

If missing:

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

### Step 1.3 — Clone and sync the repo

> Skip clone if the repo already exists at `C:\Users\smeefer\inkscape-mcp`

```powershell
cd $env:USERPROFILE
git clone https://github.com/sandraschi/inkscape-mcp.git
cd inkscape-mcp
uv sync --python "C:\Users\smeefer\miniconda3\python.exe"
```

**Note on Python version pinning:** This machine has multiple Python installs (3.13.9 via miniconda, 3.14.0 standalone). Always pass `--python` explicitly to `uv sync` to avoid silent version drift between resyncs. Miniconda's 3.13.9 is the target for this project.

### Step 1.4 — Standalone smoke test

```powershell
uv run --directory "C:\Users\smeefer\inkscape-mcp" inkscape-mcp --help
```

Confirms the server runs before Hermes ever touches it.

### Step 1.5 — Add MCP entry to Hermes config

Hermes config lives at:
```
C:\Users\smeefer\AppData\Local\hermes\config.yaml
```

Add under the existing `mcp_servers:` key alongside `agentmail` and `walrus-memory`:

```yaml
mcp_servers:
  inkscape:
    command: "uv"
    args:
      - "run"
      - "--directory"
      - "C:\\Users\\smeefer\\inkscape-mcp"
      - "inkscape-mcp"
    env:
      UV_PYTHON: "C:\\Users\\smeefer\\miniconda3\\python.exe"
    timeout: 120
```

**Why `UV_PYTHON`:** Hermes passes only explicitly declared env vars to stdio MCP servers — it does not forward the full shell environment. Without `UV_PYTHON`, `uv run` picks an arbitrary Python on this multi-Python machine and may silently use a different version on each reload.

### Step 1.6 — Reload and verify registration

```
/reload-mcp
```

Ask Cypryx what Inkscape tools she has available.

---

## Phase 2 — CachyOS Setup

### Step 2.1 — Verify Inkscape CLI

```bash
inkscape --version
```

Note the full executable path if it is not on PATH:

```bash
which inkscape
```

### Step 2.1a — Verify inkscape-mcp config.yaml

Check the Linux config after cloning:

```bash
grep "inkscape_executable" ~/inkscape-mcp/config.yaml
```

The default value uses a Windows path and must be updated for CachyOS:

```bash
sed -i 's|inkscape_executable:.*|inkscape_executable: "/usr/bin/inkscape"|' ~/inkscape-mcp/config.yaml
```

Verify with `which inkscape` first and substitute the correct path if different.

### Step 2.2 — Verify/install `uv`

```bash
uv --version
```

If missing:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### Step 2.3 — Clone and sync the repo

```bash
cd ~
git clone https://github.com/sandraschi/inkscape-mcp.git
cd inkscape-mcp
uv sync
```

### Step 2.4 — Standalone smoke test

```bash
uv run inkscape-mcp --help
```

### Step 2.5 — Add MCP entry to Hermes config

Edit `~/.hermes/config.yaml` (dict format, consistent with existing `mcp_servers` entries like Walrus Memory):

```yaml
mcp_servers:
  inkscape:
    command: "uv"
    args:
      - "run"
      - "--directory"
      - "/home/<your-username>/inkscape-mcp"
      - "inkscape-mcp"
    timeout: 120
```

**Note:** No `UV_PYTHON` env entry needed on CachyOS assuming a single system Python. If multiple Python versions are present, add the env block as on Windows.

### Step 2.6 — Restart gateway and verify

```bash
hermes gateway restart
```

Then in Noöm's chat channel:

```
/reload-mcp
```

Ask Noöm what Inkscape tools she has available.

---

## Phase 3 — Verification Testing

Run independently on **each** machine.

### Step 3.1 — Tool discovery test

**Action:** Ask the agent to list its Inkscape capabilities.
**Pass:** Agent names tools like `inkscape_vector`, `inkscape_file`, `inkscape_analysis`, `inkscape_system`, `generate_svg`, prefixed `mcp_inkscape_*`.
**Fail → fix:**
- Confirm `/reload-mcp` was run after the config edit
- Check YAML syntax (indentation, no tabs) in `config.yaml`
- Re-run `uv run inkscape-mcp --help` standalone to rule out a broken install

### Step 3.2 — Real operation test

**Action:** Ask the agent to create a simple SVG and export it as PNG (e.g. "create a 100x100 red square SVG and export it as PNG to my desktop").
**Pass:** A real PNG file appears at the requested path.
**Fail → fix:**
- Confirm `inkscape-mcp/config.yaml` has the correct `inkscape_executable` path — this is the most likely cause on Windows
- Run `uv run inkscape-mcp --verbose` and inspect stderr
- Confirm Inkscape CLI works standalone: `& "C:\Program Files\Inkscape\bin\inkscape.exe" --version`
- Check file/path permissions in Hermes's working directory

---

## Phase 4 — Optional / Deferred

Not part of this build, captured here so they aren't lost.

- [ ] Evaluate the web dashboard (ports 10846/10847) — undocumented UI, revisit once MCP tools are confirmed working
- [ ] Evaluate the Tauri desktop app packaging of the same dashboard
- [ ] Review `inkscape_system` tool for destructive operations before exposing it unfiltered — use `tools.exclude` or `tools.include` in the Hermes config entry to limit surface if needed
- [ ] Decide whether Noöm and Cypryx should get distinct prompting around when to reach for Inkscape vs. other art/generation tools

---

## Status Tracking

| Phase | Windows (Cypryx) | CachyOS (Noöm) |
|---|---|---|
| 0 — Prerequisites | ☐ | ☐ |
| 1/2 — Setup | ☐ | ☐ |
| 3.1 — Tool discovery | ☐ | ☐ |
| 3.2 — Real operation | ☐ | ☐ |

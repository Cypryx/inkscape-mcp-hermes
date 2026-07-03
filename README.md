# Inkscape MCP — Hermes Edition

**Vector graphics tooling for AI agents, via the Model Context Protocol.**

This is a fork of [sandraschi/inkscape-mcp](https://github.com/sandraschi/inkscape-mcp), configured and patched for use with [Hermes Agent](https://hermes-agent.nousresearch.com) — but it works with any MCP-compatible client (Claude Desktop, Cursor, etc.).

## What it does

Exposes Inkscape's capabilities as MCP tools that an AI agent can call directly:

- **File operations** — load, save, convert, export SVG/PDF/PNG
- **Vector editing** — path ops, booleans, text-to-path, trace bitmap, QR/barcodes
- **SVG generation** — create SVGs from natural language descriptions
- **Analysis** — inspect SVG structure, validate, measure, count nodes
- **System** — diagnostics, version checks, extensions
- **Heraldry** — preset heraldic compositions
- **Batch processing** — intelligent multi-document vector workflows

## Quick start

```bash
git clone https://github.com/Cypryx/inkscape-mcp-hermes.git
cd inkscape-mcp-hermes
uv sync
uv run inkscape-mcp --help
```

Requires **Python 3.12+**, **Inkscape 1.4+**, and [`uv`](https://docs.astral.sh/uv/).

## Configuration

Copy or edit `config.yaml` at the project root:

```yaml
inkscape_executable: "C:\Program Files\Inkscape\bin\inkscape.exe"
```

## Usage with Hermes Agent

Add to your Hermes `config.yaml` under `mcp_servers`:

```yaml
mcp_servers:
  inkscape:
    command: uv
    args:
      - run
      - --directory
      - /path/to/inkscape-mcp-hermes
      - inkscape-mcp
    env:
      MCP_TRANSPORT: stdio
    timeout: 120
```

Then run `/reload-mcp` in a Hermes session.

## Patches in this fork

- **Fixed GIMP copy-paste in detector** — the `inkscape_detector.py` was checking for `"gimp"` in executable paths instead of `"inkscape"`, rejecting Inkscape entirely
- **Transport env override fix** — `main.py` now respects `MCP_TRANSPORT` environment variable instead of unconditionally defaulting to HTTP mode

## License

[Apache 2.0](LICENSE) — same as upstream.
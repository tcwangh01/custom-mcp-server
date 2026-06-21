# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is an MCP (Model Context Protocol) Image Tools Server that provides image processing capabilities to Claude Code. The server is implemented using the FastMCP framework and runs in a Docker container.

## Development Workflow

After any change to `server.py`, `requirements.txt`, or `Dockerfile`, rebuild the Docker image and reconnect the MCP server:

```bash
# Rebuild
docker build -t mcp-toy-image-tools-server .

# Then in Claude Code: /mcp → image-tools-server-docker → Reconnect
```

To verify the server starts cleanly before rebuilding:
```bash
docker run --rm -i \
  -v $(pwd)/images:/app/images \
  -v $(pwd)/input:/app/input \
  -v $(pwd)/output:/app/output \
  mcp-toy-image-tools-server
```

## Architecture

**Single-file server** — all MCP tools live in `server.py` using FastMCP's `@mcp.tool()` decorator pattern. Each decorated async function is automatically registered as an MCP tool; no manifest or handler routing is needed.

**Container I/O** — the container has no persistent storage except through three volume mounts defined in `.mcp.json`:
- `./images` → `/app/images` — default working directory for all tool outputs
- `./input` → `/app/input` — optional source files
- `./output` → `/app/output` — optional processed outputs

**`.mcp.json` contains absolute host paths** hardcoded to the original developer's machine. When cloning to a new machine, update the `-v` source paths to the new absolute path.

## MCP Tools

| Tool | Key parameters | Notes |
|------|---------------|-------|
| `fetch_toy_image` | `keyword`, `count`, `output_dir`, `max_search_results` | Prepends "toy " to keyword automatically; shuffles results for variety |
| `resize_image` | `image_path`, `width`, `height`, `output_path`, `maintain_aspect` | Uses Lanczos resampling; `maintain_aspect=True` uses `thumbnail()` so neither dimension exceeds the given bounds |
| `remove_background_as_png` | `image_path`, `output_path` | Forces `.png` extension; uses rembg (U2Net via ONNX); downloads ~170 MB model on first run to `/home/mcp-user/.u2net/` inside the container |

## Adding New Tools

```python
@mcp.tool()
async def your_tool(param: str, optional: int = 10) -> str:
    """One-line description shown to the MCP client."""
    # validate, process, return a string
    return "result"
```

Default output to `./images/` and call `os.makedirs(output_dir, exist_ok=True)` before writing. Follow the existing `OSError` / generic `Exception` split for error handling.

## Dependencies

Core runtime dependencies (`requirements.txt`):
- `mcp` — FastMCP framework
- `Pillow` — image I/O and manipulation
- `requests` — image downloading
- `duckduckgo-search` — image search
- `rembg` + `onnxruntime` — AI background removal (no GPU required)
- `numpy` — numerical support for rembg/scikit-image

`torch`/`torchvision` are **not** used; rembg runs entirely on the ONNX runtime.

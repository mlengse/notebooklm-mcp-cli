# ChatGPT connector setup

This guide covers using `notebooklm-mcp-cli` from ChatGPT with normal ChatGPT file uploads and downloadable NotebookLM outputs.

## Capabilities

- Add a normal ChatGPT-uploaded file to NotebookLM with `source_add_chatgpt_file`.
- Retrieve source text with `source_get_content` and get a downloadable `/artifacts/...` URL.
- Download already-generated NotebookLM artifacts with `download_artifact`, including audio, video, slide decks, reports, mind maps, infographics, quizzes, flashcards, and data tables.
- Serve downloaded files from the MCP HTTP server at `/artifacts/{filename}`.

## Recommended flow for existing artifacts

1. Call `studio_status` with a `notebook_id`.
2. Pick an artifact whose `status` is `completed` and note its `artifact_id` and `type`.
3. Call `download_artifact` with:
   - `notebook_id`
   - `artifact_type`
   - `artifact_id`
   - `output_path`
4. Use the returned `download_url` from ChatGPT.

Example result:

```json
{
  "status": "success",
  "artifact_type": "audio",
  "path": "C:\\Users\\you\\Downloads\\overview.m4a",
  "download_url": "https://your-public-host.example/artifacts/overview.m4a"
}
```

## Local authentication

Before exposing the MCP server to ChatGPT, verify NotebookLM auth locally:

```powershell
uv run nlm login --profile default --check
```

If expired, run:

```powershell
uv run nlm login --profile default --clear
```

## Option A: OpenAI/ChatGPT secure tunnel

Use this when ChatGPT is configured in tunnel mode.

Start the local MCP server:

```powershell
cd X:\Code\notebooklm-mcp-cli
pwsh -File .\tools\run-chatgpt-tunnel.ps1 -Profile default -NoOpenAITunnel
```

Then create/connect the ChatGPT secure MCP tunnel using target URL:

```text
http://127.0.0.1:8811/mcp
```

The same server exposes files at:

```text
http://127.0.0.1:8811/artifacts/<filename>
```

When ChatGPT connects through a secure tunnel, the server captures forwarded host headers and returns public artifact links using the tunnel host.

## Option B: Cloudflare tunnel

Use this when you want a stable public hostname.

Start the MCP server:

```powershell
cd X:\Code\notebooklm-mcp-cli
uv run notebooklm-mcp --transport http --host 127.0.0.1 --port 8811 --query-timeout 600
```

Create a Cloudflare tunnel ingress rule pointing your hostname to the local origin:

```yaml
hostname: notebooklm-mcp.example.com
service: http://127.0.0.1:8811
```

Then configure ChatGPT to use:

```text
https://notebooklm-mcp.example.com/mcp
```

Downloads will be returned as:

```text
https://notebooklm-mcp.example.com/artifacts/<filename>
```

For a quick local reverse-proxy test:

```powershell
pwsh -File .\tools\run-cloudflare-chatgpt.ps1 -PublicHostname notebooklm-mcp.example.com -Profile default
```

## Tool usage examples

### Download an existing audio artifact

```json
{
  "notebook_id": "<notebook-id>",
  "artifact_type": "audio",
  "artifact_id": "<completed-audio-artifact-id>",
  "output_path": "C:\\Users\\you\\Downloads\\notebooklm-audio.m4a",
  "wait": true,
  "wait_timeout": 180,
  "poll_interval": 5
}
```

### Download an existing slide deck

```json
{
  "notebook_id": "<notebook-id>",
  "artifact_type": "slide_deck",
  "artifact_id": "<completed-slide-deck-artifact-id>",
  "output_path": "C:\\Users\\you\\Downloads\\notebooklm-slides.pdf",
  "slide_deck_format": "pdf"
}
```

### Retrieve source text as a downloadable file

```json
{
  "source_id": "<source-id>",
  "wait": true,
  "wait_timeout": 120,
  "poll_interval": 3
}
```

## Delay handling

NotebookLM may need time to index new sources or propagate generated media URLs. The download tools now poll transient states by default:

- `source_get_content`: waits for source text indexing.
- `download_artifact`: waits for existing generated artifacts whose media URL is still propagating.

For already-generated artifacts, this usually returns immediately.

## Security notes

- `/artifacts/{filename}` serves files from the configured `public_artifacts` directory only.
- Path traversal is neutralized by reducing requests to the basename.
- Do not expose the MCP server publicly without the same access controls you use for other private NotebookLM tooling.
- Cloudflare Access or ChatGPT's secure tunnel is preferred over an unauthenticated public hostname.

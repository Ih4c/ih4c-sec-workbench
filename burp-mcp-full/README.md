# BurpSuite MCP Full Control Extension

Full control of every core BurpSuite capability over the MCP protocol. Cross-platform: Windows / Linux (Kali) / macOS.

## Quick Start

### 1. Build the Extension

**Windows**:
```cmd
cd burp-mcp-full
build.bat
```

**Linux / Kali / macOS**:
```bash
cd burp-mcp-full
chmod +x build.sh
./build.sh
```

The build script automatically: detects JDK 21+, downloads dependencies (montoya-api 2025.5 / gson / nanohttpd), compiles, embeds the extension descriptor (`META-INF/extensions/burp-extension.properties`) into the jar, and packages a fat jar. No Gradle needed.

Output: `build/libs/burp-mcp-full.jar`.

### 2. Load into Burp

```
Burp Suite → Extensions → Add → Java → select build/libs/burp-mcp-full.jar
```

After loading, you will see in Output:
```
[MCP] Server started on http://127.0.0.1:9876
```

### 3. Authentication (Enabled by Default Since v2)

On startup the extension auto-generates a random token and writes it to `~/.burp-mcp-token`. `mcp-bridge.js` reads that file automatically and attaches an `Authorization: Bearer <token>` header to every request — no manual configuration needed.

When a fixed token is required (e.g., shared by multiple clients), you can use:
- JVM argument: `-Dburp.mcp.token=<token>`
- Environment variable: `BURP_MCP_TOKEN=<token>` (also used on the bridge side)

All `/health`, `/tools`, and `/` (POST) requests require this header, otherwise 403 is returned. CORS has been tightened to allow only the `http://127.0.0.1` origin.

### 4. Configure the MCP Client

Add this to any MCP client (Claude Code / Kiro / Cursor / Cline / Windsurf) (stdio mode):

```json
{
  "mcpServers": {
    "burpsuite": {
      "command": "node",
      "args": ["<path to this directory>/mcp-bridge.js"]
    }
  }
}
```

### 5. Start Using It

Tell the AI: "analyze the requests in the Burp proxy history and find security vulnerabilities"

## Feature List

The extension exposes 78 tools. Common categories are below (the full list is in `getToolList()` in `src/main/java/com/burpmcp/McpHttpServer.java`, or see `GET http://127.0.0.1:9876/tools`, which requires the Authorization header):

| Category | Tools |
|------|------|
| Proxy history | `proxy_history`, `proxy_detail`, `proxy_history_filtered`, `proxy_websocket`, `proxy_clear`, `search_history`, `highlight`, `annotate`, `compare` |
| Sending requests | `send_request`, `send_to_repeater`, `repeater_send`, `repeater_modify_send`, `send_to_intruder` |
| Intruder attacks | `intruder_attack`, `intruder_attack_async`, `intruder_attack_wordlist`, `intruder_pitchfork`, `intruder_cluster_bomb`, `intruder_battering_ram`, `intruder_with_options`, `payload_process` |
| Scan / crawl | `scan`(active/passive), `scan_active`, `scan_results`, `scan_issue_detail`, `crawl`, `sequencer` |
| Scope / Sitemap | `sitemap`, `target_info`, `get_scope`, `add_to_scope`, `remove_from_scope`, `add_issue` |
| Intercept / rules | `intercept_toggle`, `register_http_handler`, `remove_http_handler`, `register_proxy_rule`, `remove_proxy_rule` |
| Encode/decode | `encode`, `decode`, `convert_request`, `export_request`, `generate_csrf_poc`, `extract_from_response`, `token_analysis` |
| Collaborator | `collaborator_generate`, `collaborator_poll` |
| Configuration | `export_config`, `import_config`, `set_upstream_proxy`, `set_dns_override`, `set_http2`, `cookie_jar`, `save_project`, `burp_version`, `extensions_list`, `log` |

> Scanning/crawling (`scan`, `scan_active`, `crawl`) requires **Burp Professional**. The Community edition returns an explicit license error. Issues added manually (`add_issue`) are written to the Site map.

## Key Tool Parameters

### `intruder_attack` — Automated Enumeration Attacks

| Parameter | Description |
|------|------|
| `url_template` | URL template; the placeholder defaults to `@@` |
| `placeholder` | The placeholder string (default `@@`) |
| `from` / `to` | Enumeration start/end values |
| `pad_digits` | Number of zero-padding digits (0 = no padding) |
| `method` | HTTP method (default GET) |
| `body_template` | Request body template (may contain the placeholder) |
| `headers` | Request headers object |
| `success_length_not` | Hit condition: response length ≠ this value |
| `success_contains` | Hit condition: response body contains this string |

### `scan` — Start an Audit

| Parameter | Description |
|------|------|
| `url` | Target URL (required; added to scope automatically) |
| `mode` | `active` (default) or `passive` |

After starting, poll `scan_results` for issues and active-audit status (request count, error count, insertion-point count).

### `register_proxy_rule` — Proxy Request Interception Rules

| Parameter | Description |
|------|------|
| `url_contains` | Hit condition: URL contains this string |
| `intercept` | `true` intercept / `false` allow without intercepting (default true) |

Deregister rules via `remove_proxy_rule` (based on `Registration.deregister()`, truly unloading them from Burp).

## Usage Examples

### View proxy history
```json
POST http://127.0.0.1:9876
{"tool": "proxy_history", "params": {"limit": 10, "url_filter": "personalblog"}}
```

### Send a request
```json
POST http://127.0.0.1:9876
{"tool": "send_request", "params": {"method": "GET", "url": "https://example.com/api/test"}}
```

### Automated enumeration attack (core feature)
```json
POST http://127.0.0.1:9876
{
  "tool": "intruder_attack",
  "params": {
    "url_template": "https://target.com/api/verify?code=@@",
    "method": "POST",
    "from": 0,
    "to": 999999,
    "pad_digits": 6,
    "success_length_not": 176,
    "headers": {"User-Agent": "Mozilla/5.0"}
  }
}
```

### Toggle interception
```json
POST http://127.0.0.1:9876
{"tool": "intercept_toggle", "params": {"enable": false}}
```

## Port Configuration

The default listener is `127.0.0.1:9876`. To change it (for example, on a port conflict with the official PortSwigger MCP extension):

1. **Burp side**: pass the JVM argument `-Dburp.mcp.port=9877` when starting Burp, or set the environment variable `BURP_MCP_PORT=9877`.
2. **Bridge side**: set the environment variables `BURP_MCP_PORT=9877` and `BURP_MCP_HOST=127.0.0.1` in the MCP client configuration.

Both sides must use the same port. If Burp is not running or the port is unreachable, the bridge returns clear connection-error guidance on `tools/list` and `tools/call`.

## Troubleshooting

| Symptom | Diagnosis |
|------|------|
| No "[MCP] Server started" in Burp Output | The port is occupied or the extension failed to load; check the Burp Errors panel |
| MCP client reports "Burp MCP not connected" | Confirm Burp is running with the extension loaded; confirm both sides use the same port |
| Scanning returns "requires Burp Professional" | Expected; the Community edition does not support the Scanner API |
| `remove_http_handler` / `remove_proxy_rule` has no effect | Confirm the earlier `register_*` returned success=true |

## Building from Source (Gradle Optional)

```bash
cd burp-mcp-full
gradle jar      # requires Gradle 8.7+ installed locally
# Output: build/libs/burp-mcp-full.jar
```

> The `build.bat` / `build.sh` scripts are recommended (zero dependencies, jars downloaded automatically). The Gradle path is only a fallback.

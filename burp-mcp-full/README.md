# BurpSuite MCP Full Control Extension

Control all core BurpSuite functionality through the MCP protocol. Supports Windows / Linux (Kali) / macOS.

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

The build script automatically detects JDK 21+, downloads dependencies (montoya-api 2026.7 / gson 2.15.0 / nanohttpd 2.3.1), compiles the extension, embeds the extension descriptor (`META-INF/extensions/burp-extension.properties`) in the JAR, and packages a fat JAR. Gradle is not required.

Output: `build/libs/burp-mcp-full.jar`.

### 2. Load into Burp

```
Burp Suite â†’ Extensions â†’ Add â†’ Java â†’ select build/libs/burp-mcp-full.jar
```

After loading, Output should show:
```
[MCP] Server started on http://127.0.0.1:9876
```

### 3. Authentication (enabled by default since v2)

On startup, the extension generates a random token and writes it to `~/.burp-mcp-token`. `mcp-bridge.js` reads this file automatically and sends the `Authorization: ******` header with every request; no manual configuration is required.

To use a fixed token (for example, when sharing it across multiple clients), use:
- JVM argument: `-Dburp.mcp.token=<token>`
- Environment variable: `BURP_MCP_TOKEN=<token>` (also used by the bridge)

All `/health`, `/tools`, and `/` (POST) requests require this header; otherwise they return 403. CORS is restricted to the `http://127.0.0.1` origin only.

### 4. Configure the MCP Client

Add the following to any MCP client (Claude Code / Kiro / Cursor / Cline / Windsurf) in stdio mode:

```json
{
  "mcpServers": {
    "burpsuite": {
      "command": "node",
      "args": ["<æœ¬ç›®å½•è·¯å¾„>/mcp-bridge.js"]
    }
  }
}
```

### 5. Start Using It

Tell the AI: "Analyze requests in Burp proxy history and find security vulnerabilities."

## Feature List

The extension exposes 78 tools. Common categories are listed below (see `getToolList()` in `src/main/java/com/burpmcp/McpHttpServer.java` for the full list, or request `GET http://127.0.0.1:9876/tools` with the Authorization header):

| Category | Tools |
|----------|-------|
| Proxy History | `proxy_history`, `proxy_detail`, `proxy_history_filtered`, `proxy_websocket`, `proxy_clear`, `search_history`, `highlight`, `annotate`, `compare` |
| Send Requests | `send_request`, `send_to_repeater`, `repeater_send`, `repeater_modify_send`, `send_to_intruder` |
| Intruder Attacks | `intruder_attack`, `intruder_attack_async`, `intruder_attack_wordlist`, `intruder_pitchfork`, `intruder_cluster_bomb`, `intruder_battering_ram`, `intruder_with_options`, `payload_process` |
| Scanning / Crawling | `scan` (active/passive), `scan_active`, `scan_results`, `scan_issue_detail`, `crawl`, `sequencer` |
| Scope / Sitemap | `sitemap`, `target_info`, `get_scope`, `add_to_scope`, `remove_from_scope`, `add_issue` |
| Interception / Rules | `intercept_toggle`, `register_http_handler`, `remove_http_handler`, `register_proxy_rule`, `remove_proxy_rule` |
| Encoding / Decoding | `encode`, `decode`, `convert_request`, `export_request`, `generate_csrf_poc`, `extract_from_response`, `token_analysis` |
| Collaborator | `collaborator_generate`, `collaborator_poll` |
| Configuration | `export_config`, `import_config`, `set_upstream_proxy`, `set_dns_override`, `set_http2`, `cookie_jar`, `save_project`, `burp_version`, `extensions_list`, `log` |

> Scanning/crawling (`scan`, `scan_active`, `crawl`) requires **Burp Professional**. The Community edition returns a clear license error. Manually added issues (`add_issue`) are written to the Site map.

## Key Tool Parameters

### `intruder_attack` â€” Automated Enumeration Attack

| Parameter | Description |
|-----------|-------------|
| `url_template` | URL template; the default placeholder is `@@` |
| `placeholder` | Placeholder string (default `@@`) |
| `from` / `to` | Enumeration start and end values |
| `pad_digits` | Number of zero-padding digits (0 means no padding) |
| `method` | HTTP method (default GET) |
| `body_template` | Request-body template (including the placeholder) |
| `headers` | Headers object |
| `success_length_not` | Match condition: response length â‰  this value |
| `success_contains` | Match condition: response body contains this string |

### `scan` â€” Start an Audit

| Parameter | Description |
|-----------|-------------|
| `url` | Target URL (required; automatically added to scope) |
| `mode` | `active` (default) or `passive` |

After starting, poll `scan_results` for issues and active-audit status (request count, error count, and insertion-point count).

### `register_proxy_rule` â€” Proxy Request Interception Rule

| Parameter | Description |
|-----------|-------------|
| `url_contains` | Match condition: URL contains this string |
| `intercept` | `true` intercepts / `false` allows without interception (default true) |

Use `remove_proxy_rule` to deregister the rule (based on `Registration.deregister()`, which actually unloads it from Burp).

## Usage Examples

### View Proxy History
```json
POST http://127.0.0.1:9876
{"tool": "proxy_history", "params": {"limit": 10, "url_filter": "personalblog"}}
```

### Send a Request
```json
POST http://127.0.0.1:9876
{"tool": "send_request", "params": {"method": "GET", "url": "https://example.com/api/test"}}
```

### Automated Enumeration Attack (Core Feature)
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

### Toggle Interception
```json
POST http://127.0.0.1:9876
{"tool": "intercept_toggle", "params": {"enable": false}}
```

## Port Configuration

The default listener is `127.0.0.1:9876`. To change it (for example, to avoid a port conflict with PortSwigger's official MCP extension):

1. **Burp side**: start Burp with the JVM argument `-Dburp.mcp.port=9877`, or set the `BURP_MCP_PORT=9877` environment variable.
2. **Bridge side**: set the `BURP_MCP_PORT=9877` and `BURP_MCP_HOST=127.0.0.1` environment variables in the MCP client configuration.

The ports on both sides must match. If Burp is not running or the port is unreachable, the bridge returns clear connection-error guidance for `tools/list` and `tools/call`.

## Troubleshooting

| Symptom | Troubleshooting |
|------|------|
| Burp Output does not show "[MCP] Server started" | The port is in use or the extension failed to load; check the Burp Errors panel |
| MCP client reports "Burp MCP not connected" | Confirm that Burp is running and the extension is loaded; confirm that both ports match |
| Scanning returns "requires Burp Professional" | Expected: the Community edition does not support the Scanner API |
| `remove_http_handler` / `remove_proxy_rule` has no effect | Confirm that the earlier `register_*` call returned success=true |

## Build from Source (Gradle Optional)

```bash
cd burp-mcp-full
gradle jar      # éœ€æœ¬æœºå·²è£… Gradle 9.9.1+
# è¾“å‡º: build/libs/burp-mcp-full.jar
```

> `build.bat` / `build.sh` is recommended (zero dependencies; downloads the JAR automatically). The Gradle path is provided only as an alternative.

# JEB MCP — Analyzing APKs with Claude Code

A tool that connects the JEB Pro decompiler via MCP (Model Context Protocol) to analyze APKs from Claude Code.

## Requirements

- **JEB Pro** 5.x
- **Python 3.8+** (for MCP server, no external packages required)
- **Claude Code** CLI

## Installation

### 1. Place Files

Place the following files in your JEB installation directory:

```
<JEB_DIR>/
├── bin/jeb-engines.cfg                        # Modified in Step 2
├── coreplugins/scripts/McpBridgeAutoStart.py  # JEB Bridge (auto-start)
└── mcp_server/jeb_mcp_server.py               # MCP Server
```

### 2. Enable Python Plugin

Add the following to `bin/jeb-engines.cfg`:

```
.LoadPythonPlugins = true
```

This setting causes `coreplugins/scripts/McpBridgeAutoStart.py` to be loaded automatically on JEB startup, which starts the MCP Bridge HTTP server.

### 3. Configure Claude Code MCP

Register the MCP server in your Claude Code config file (`~/.claude/settings.json` or your project's `.mcp.json`).

**.mcp.json** (create in project root):
```json
{
  "mcpServers": {
    "jeb": {
      "command": "python",
      "args": ["mcp_server/jeb_mcp_server.py"],
      "cwd": "<JEB_DIR>"
    }
  }
}
```

Or add directly from Claude Code:
```bash
claude mcp add jeb -- python <JEB_DIR>/mcp_server/jeb_mcp_server.py
```

### 4. Change Port (Optional)

The default port is `18700`. To change it, set the `JEB_MCP_PORT` environment variable.

```json
{
  "mcpServers": {
    "jeb": {
      "command": "python",
      "args": ["mcp_server/jeb_mcp_server.py"],
      "cwd": "<JEB_DIR>",
      "env": {
        "JEB_MCP_PORT": "19000"
      }
    }
  }
}
```

Since the JEB side also reads the same environment variable, set it as a system environment variable before launching JEB, or add it to your JEB launch script.

## Usage

### Basic Flow

```
1. Launch JEB → Open APK (Bridge starts automatically)
2. Launch Claude Code (MCP server connects automatically)
3. Begin analysis
```

### Analysis Examples

```
# Basic analysis
> Analyze this APK's manifest
> Show me the list of exported components
> Decompile MainActivity

# Vulnerability analysis (built-in skills)
> /find-intent-redirect
> /find-deeplink-vuln

# Code search
> Find code that calls loadUrl
> Search for startActivity patterns in the com.example package

# Reversing
> Analyze obfuscated class a.b.c and rename it with meaningful names
```

## MCP Tools (24 total)

| Category | Tool | Description |
|----------|------|-------------|
| Navigation | `get_jeb_status` | Check connection status |
| | `execute_script` | Execute arbitrary Jython code |
| | `list_units` | List analysis units |
| | `list_classes` | List classes (filterable) |
| | `list_methods` | List methods |
| | `list_fields` | List fields |
| Decompilation | `decompile_class` | Decompile entire class |
| | `decompile_method` | Decompile a method |
| Analysis | `get_xrefs` | Cross-references |
| | `search_strings` | String search |
| | `search_code` | Source code pattern search |
| | `get_callgraph` | Call graph |
| | `get_class_hierarchy` | Class inheritance hierarchy |
| Renaming | `rename_class` | Rename a class |
| | `rename_method` | Rename a method |
| | `rename_field` | Rename a field |
| | `rename_variable` | Rename a variable |
| | `bulk_rename` | Bulk rename |
| | `move_class_to_package` | Move to a package |
| | `add_comment` | Add a comment |
| Android | `get_apk_manifest` | AndroidManifest.xml |
| | `get_apk_certificate` | Signing certificate info |
| | `get_apk_resource` | Look up resource files |
| | `list_components` | List components |

## Architecture

```
Claude Code ──stdio──> MCP Server (Python 3) ──HTTP──> JEB Bridge (Jython) ──> JEB API
```

1. When Claude Code calls a tool, the MCP Server generates a Jython code template
2. The template is sent via HTTP POST to the JEB Bridge
3. The JEB Bridge executes the Jython code inside JEB
4. Results are returned as JSON

## Troubleshooting

### Bridge won't start
- Verify `.LoadPythonPlugins = true` is present in `bin/jeb-engines.cfg`
- Check the JEB console for `[MCP Bridge]` log messages
- If there's a port conflict, change it with the `JEB_MCP_PORT` environment variable

### MCP server connection fails
- Confirm JEB is running and an APK is open
- Check Bridge status with `curl http://localhost:18700/status`
- Verify the port matches on both the MCP server and Bridge sides

### search_code is slow
- Use the `class_filter` parameter to limit the search scope to a package name

### Decompilation only returns stubs
- `decompile_class` internally decompiles each method individually, so this is expected behavior
- Use `decompile_method` to inspect individual methods

# Template Import Guide

> How to import `.penpot` template files into your self-hosted Penpot instance using the REST API. This is useful for bootstrapping projects with pre-built UI kits, design systems, and dashboard layouts.

## Prerequisites

- Self-hosted Penpot instance with `enable-access-tokens` flag
- Valid Access Token (Settings > Access Tokens in Penpot UI)
- A `.penpot` file to import

## Import via REST API

File import is **not available as an MCP tool**. Use the Penpot REST API directly:

```bash
# Set your variables
PENPOT_API_URL="http://localhost:9001"
PENPOT_ACCESS_TOKEN="your-token-here"
PROJECT_ID="your-project-id"

# Import the file
curl -X POST "$PENPOT_API_URL/api/rpc/command/import-binfile" \
  -H "Authorization: Token $PENPOT_ACCESS_TOKEN" \
  -F "file=@template.penpot" \
  -F "project-id=$PROJECT_ID" \
  -F "name=My Template"
```

### Response Format

The import API returns **Server-Sent Events (SSE)** format, not JSON:

```
data: {"type":"progress","data":{"progress":0.5}}
data: {"type":"progress","data":{"progress":1.0}}
data: {"type":"done","data":{...}}
```

Parse lines starting with `data:`. The final `data:` line contains the import result.

### Post-Import Verification

After import, verify using MCP tools:

```
1. list_files(projectId) → confirm new file appears
2. list_pages(fileId) → check all pages imported correctly
3. query_shapes(fileId, pageId) → verify shapes are accessible
```

---

## Finding Templates

### Official Sources

| Source | URL | Format |
|--------|-----|--------|
| Penpot Hub | https://penpot.app/penpothub/libraries-templates | Browse & download `.penpot` files |
| GitHub penpot-files | https://github.com/penpot/penpot-files | 130+ `.penpot` files, direct download |
| Penpot Community | https://community.penpot.app/c/libraries-templates | Community contributions |

### Downloading from GitHub

```bash
# Example: Download Dashboard UI Starter Kit
curl -L -o dashboard.penpot \
  "https://github.com/penpot/penpot-files/raw/main/Dashboard%20UI%20Starter%20Kit.penpot"

# Verify it's a valid ZIP file
file dashboard.penpot
# Should show: "Zip archive data, ..."
```

**Important:** If `file` shows `"data"` instead of `"Zip archive"`, the file may be corrupted. Try downloading from Penpot Hub instead.

---

## Template Quality Checklist

Before importing a template, evaluate it:

### Pre-Import Checks

| Check | How | Red Flag |
|-------|-----|----------|
| License | Read GitHub README or template description | "DEMO", "PRO", "freemium" labels |
| Design style | View screenshots/previews | Bootstrap-style (dated appearance) |
| Completeness | Check page count and descriptions | "PRO version required for full access" |
| File integrity | `file template.penpot` | Shows "data" instead of "Zip archive" |

### Post-Import Checks

```
1. list_pages(fileId) → get page count
2. For each page: navigate to it in Penpot UI
3. Look for "PRO-locked" or "not included in FREE version" messages
4. Report: X of Y pages fully accessible
```

### Known Template Evaluations

These templates from `penpot/penpot-files` have been tested:

| Template | Status | Notes |
|----------|--------|-------|
| Dashboard UI Starter Kit | FREE, Excellent | Modern dark/light dashboard with charts, calendar, chat, stats. 6 pages |
| Sales Dashboard Example | FREE, Excellent | Sales dashboard with revenue, leaderboard, cost breakdown. Dark/light. 5 pages |
| Material Design 3 | FREE, Good | Google MD3 component library. 25+ component types. 6 pages |
| SaaS Dashboard UI Kit | FREE, Good | Modern SaaS layout, full access |
| Labyrinth UI | FREE, Good | Material Design kit for SaaS. 49 pages |
| Ant Design UI Kit Lite | Broken | GitHub file not valid ZIP. Download from Penpot Hub instead |
| CoreUI DesignSystem DEMO | Avoid | Bootstrap-based (dated style), ~60% pages PRO-locked |

---

## Batch Import Script

For importing multiple templates:

```bash
#!/bin/bash
# batch-import.sh - Import multiple .penpot files

PENPOT_API_URL="${PENPOT_API_URL:-http://localhost:9001}"
PROJECT_ID="$1"

if [ -z "$PROJECT_ID" ] || [ -z "$PENPOT_ACCESS_TOKEN" ]; then
  echo "Usage: PENPOT_ACCESS_TOKEN=xxx ./batch-import.sh <project-id>"
  exit 1
fi

for file in *.penpot; do
  name="${file%.penpot}"
  echo "Importing: $name"

  # Verify file integrity
  if ! file "$file" | grep -q "Zip archive"; then
    echo "  SKIP: $file is not a valid .penpot file"
    continue
  fi

  curl -s -X POST "$PENPOT_API_URL/api/rpc/command/import-binfile" \
    -H "Authorization: Token $PENPOT_ACCESS_TOKEN" \
    -F "file=@$file" \
    -F "project-id=$PROJECT_ID" \
    -F "name=$name" > /dev/null

  echo "  Done: $name"
done

echo "Import complete. Verify with: list_files(projectId)"
```

---

## Troubleshooting

See [Troubleshooting Guide](TROUBLESHOOTING.md) for:
- 401 Unauthorized errors
- "zip END header not found" errors
- Duplicate file creation
- SSE response parsing

---

*This guide is maintained by the community. Contributions welcome!*

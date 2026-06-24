# Agent Integration Guide

> How AI coding agents (Claude Code, Cursor, Windsurf, etc.) can effectively use penpot-mcp-server to create and manage designs programmatically.

## Quick Start for Agents

### 1. Navigation Pattern (Always Start Here)

```
list_teams() → pick teamId
  → list_projects(teamId) → pick projectId
    → list_files(projectId) → pick fileId
      → list_pages(fileId) → pick pageId
        → query_shapes(fileId, pageId) → work with shapes
```

This hierarchy is mandatory. You cannot skip levels — every shape operation requires `fileId` and `pageId`.

### 2. MCP Configuration

```json
{
  "penpot-headless": {
    "type": "stdio",
    "command": "npx",
    "args": ["-y", "@zcubekr/penpot-mcp-server"],
    "env": {
      "PENPOT_ACCESS_TOKEN": "<your-access-token>",
      "PENPOT_API_URL": "http://localhost:9001"
    }
  }
}
```

**Prerequisites:**
- Self-hosted Penpot instance with `enable-access-tokens` flag
- Access Token generated from Settings > Access Tokens

### 3. Verification

After configuring, verify the connection works:

```
list_teams() → should return at least one team
```

If this fails, see [Troubleshooting](TROUBLESHOOTING.md).

---

## Tool Decision Tree

Use this decision tree to pick the right tool for each task:

```
What do you need to do?
│
├── Navigate/discover
│   ├── Find teams/projects/files → list_teams, list_projects, list_files
│   ├── Find pages in a file → list_pages
│   └── Find shapes on a page → query_shapes (filtered) or get_page_shapes (all)
│
├── Create shapes
│   ├── Rectangle/square → create_rectangle
│   ├── Circle/ellipse → create_circle
│   ├── Text label → create_text
│   ├── Container/layout → create_frame
│   ├── Vector graphic → create_svg
│   └── Custom path → create_path
│
├── Modify shapes
│   ├── Change position/size/color/text → update_shape
│   ├── Move to different parent → move_shapes
│   ├── Align multiple shapes → align_shapes
│   ├── Space shapes evenly → distribute_shapes
│   └── Remove shape → delete_shape
│
├── Work with components
│   ├── Create reusable component → create_component
│   ├── List available components → list_components
│   └── Update component metadata → update_component
│
├── Manage files
│   ├── Create new file → create_file
│   ├── Import .penpot template → see TEMPLATE_IMPORT.md
│   ├── Rename file → rename_file
│   └── Delete file → delete_file
│
└── Query/inspect
    ├── Find shapes by type/name/color → query_shapes
    ├── Get detailed shape properties → get_shape_properties
    └── Search shapes by name → search_shapes
```

---

## Common Agent Workflows

### Workflow 1: Create a Dashboard Layout

```
1. create_file(projectId, "My Dashboard")
2. list_pages(fileId) → get default pageId
3. create_frame(fileId, pageId, {name: "Sidebar", x: 0, y: 0, width: 250, height: 800})
4. create_frame(fileId, pageId, {name: "Header", x: 250, y: 0, width: 1150, height: 64})
5. create_frame(fileId, pageId, {name: "Content", x: 250, y: 64, width: 1150, height: 736})
6. create_text(fileId, pageId, {content: "Dashboard", ...}) inside Header frame
7. Create stat cards, charts, tables inside Content frame
```

### Workflow 2: Bulk Update Shapes

```
1. query_shapes(fileId, pageId, {types: ["text"], fields: ["all"]})
2. Filter results for target shapes
3. Loop: update_shape(fileId, pageId, shapeId, {newProperties})
4. Verify: query_shapes again to confirm changes
```

### Workflow 3: Inspect Existing Design

```
1. list_pages(fileId) → enumerate all pages
2. For each page: get_page_shapes(fileId, pageId) → shape overview
3. For interesting shapes: get_shape_properties(fileId, shapeId) → full details
4. Extract colors, fonts, sizes, layout patterns
```

---

## Shape Property Reference

### Common Properties for `update_shape`

| Property | Type | Example | Notes |
|----------|------|---------|-------|
| `x`, `y` | number | `100` | Position (absolute page coordinates) |
| `width`, `height` | number | `200` | Dimensions |
| `name` | string | `"Header"` | Shape name in layers panel |
| `hidden` | boolean | `true` | Visibility toggle |
| `opacity` | number | `0.5` | 0.0 to 1.0 |
| `rotation` | number | `45` | Degrees |
| `fillColor` | string | `"#FF5733"` | Hex color |
| `fillOpacity` | number | `1.0` | Fill transparency |
| `strokeColor` | string | `"#000000"` | Border color |
| `strokeWidth` | number | `2` | Border width |
| `strokeAlignment` | string | `"center"` | `"inner"`, `"center"`, `"outer"` |
| `borderRadius` | number | `8` | Corner radius (rectangles) |
| `shadow` | object | `{color, offsetX, offsetY, blur, spread}` | Drop shadow |
| `blur` | object | `{type, value}` | Blur effect |

### Text-Specific Properties

| Property | Type | Example |
|----------|------|---------|
| `content` | string | `"Hello World"` |
| `fontFamily` | string | `"Inter"` |
| `fontSize` | number | `16` |
| `fontWeight` | string | `"700"` |
| `textAlign` | string | `"center"` |
| `lineHeight` | number | `1.5` |
| `letterSpacing` | number | `0` |
| `textDecoration` | string | `"underline"` |

---

## Query Patterns

### Filter by Shape Type

```
query_shapes(fileId, pageId, {types: ["rect"]})      → rectangles only
query_shapes(fileId, pageId, {types: ["text"]})      → text elements only
query_shapes(fileId, pageId, {types: ["frame"]})     → frames/boards only
query_shapes(fileId, pageId, {types: ["circle"]})    → circles only
```

### Filter by Name

```
query_shapes(fileId, pageId, {namePattern: "Button"})  → shapes with "Button" in name
query_shapes(fileId, pageId, {namePattern: "^Header"}) → shapes starting with "Header"
```

### Select Specific Fields

```
query_shapes(fileId, pageId, {fields: ["id", "name", "type", "x", "y", "width", "height"]})
```

Use `fields: ["all"]` for complete property dump (can be very large).

---

## Important Gotchas for Agents

### 1. Root Frame

`query_shapes` may return a Root Frame with ID `00000000-0000-0000-0000-000000000000`. This is an invisible container — **always filter it out** in your results.

### 2. Coordinate System

All shape coordinates (`x`, `y`) are **absolute page coordinates**, not relative to parent frames. When placing shapes inside a frame, calculate positions relative to the page origin.

### 3. Large Responses

`query_shapes` on complex pages can return 200K+ characters. Use:
- `types` filter to narrow down shape types
- `fields` filter to request only needed properties
- `namePattern` to search by name

### 4. Session State

The MCP server maintains session state. If you encounter stale data after external changes to the Penpot file, re-query to get fresh data.

### 5. Import is Not a Tool

File import (`.penpot` files) is **not available as an MCP tool**. Use the Penpot REST API directly:

```bash
curl -X POST "$PENPOT_API_URL/api/rpc/command/import-binfile" \
  -H "Authorization: Token $PENPOT_ACCESS_TOKEN" \
  -F "file=@template.penpot" \
  -F "project-id=$PROJECT_ID" \
  -F "name=Template Name"
```

See [Template Import Guide](TEMPLATE_IMPORT.md) for details.

---

## Best Practices

1. **Navigate before creating** — Always `list_pages` to find the right page before creating shapes
2. **Name everything** — Set meaningful `name` properties so shapes are findable via `query_shapes`
3. **Use frames for layout** — Group related shapes in frames for better organization
4. **Verify after changes** — After `update_shape`, use `get_shape_properties` to confirm the update
5. **Batch queries** — Use `query_shapes` with filters instead of multiple `get_shape_properties` calls
6. **Check for duplicates** — After file import, `list_files` to ensure no accidental duplicates

---

*This guide is maintained by the community. Contributions welcome!*

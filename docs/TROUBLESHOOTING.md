# Troubleshooting Guide

> Common issues when using penpot-mcp-server and their solutions. Organized by category for quick lookup.

## Connection Issues

### `list_teams` returns timeout or connection refused

**Cause:** Penpot Docker containers are not running.

**Fix:**
```bash
docker compose -p penpot up -d
# Wait 15 seconds for backend startup
curl -s -o /dev/null -w "%{http_code}" http://localhost:9001/api/rpc/command/get-profile
# Should return 200 when ready
```

### `list_teams` returns 401/403 Unauthorized

**Cause:** Access Token is invalid or expired.

**Fix:**
1. Go to Penpot UI > Settings > Access Tokens
2. Generate a new token
3. Update the `PENPOT_ACCESS_TOKEN` environment variable in your MCP configuration
4. Restart your AI agent/IDE session (MCP servers initialize at startup)

### "Bad Gateway" (502) from Penpot

**Cause:** Backend container hasn't finished starting up yet.

**Fix:** Wait 10-15 seconds. Penpot backend needs time to initialize. Use the health check:
```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:9001/api/rpc/command/get-profile
```
Retry when it returns `200`.

---

## Tool Errors

### `create_rectangle` / `create_text` returns "invalid page"

**Cause:** The `pageId` is wrong or the page has been deleted.

**Fix:**
```
list_pages(fileId) → get current valid page IDs
```

### `query_shapes` returns Root Frame with all zeros ID

**Cause:** Normal behavior. Every page has an invisible root frame (`00000000-0000-0000-0000-000000000000`).

**Fix:** Filter it out in your results. This is not a real shape.

### `query_shapes` returns extremely large response (200K+ characters)

**Cause:** Complex pages with many shapes. Requesting `fields: ["all"]` amplifies this.

**Fix:**
- Use `types` filter: `{types: ["text"]}` to narrow down
- Use `fields` filter: `{fields: ["id", "name", "type"]}` for minimal data
- Use `namePattern` to search specific shapes

---

## Configuration Issues

### MCP server not found / tools not available

**Cause:** AI agent hasn't loaded the MCP server configuration, or needs a session restart.

**Fix:**
1. Verify MCP config in your AI agent settings
2. Restart the AI agent session (most agents only load MCP servers at startup)
3. Test with `list_teams()` after restart

### Environment variable changes not taking effect

**Cause:** MCP servers are initialized at agent startup. Runtime env changes are not reflected.

**Fix:** Restart the AI agent session after any configuration changes.

---

## Docker Issues

### Penpot containers keep restarting

**Cause:** Missing or invalid `PENPOT_SECRET_KEY` in docker-compose.yaml.

**Fix:**
```bash
# Generate a proper secret key
python3 -c "import secrets; print(secrets.token_urlsafe(64))"
# Update PENPOT_SECRET_KEY in docker-compose.yaml
docker compose -p penpot down && docker compose -p penpot up -d
```

### Access Token feature not available in UI

**Cause:** The `enable-access-tokens` flag is not set in Penpot configuration.

**Fix:** Add `enable-access-tokens` to the `PENPOT_FLAGS` environment variable in your docker-compose.yaml:
```yaml
- PENPOT_FLAGS=enable-registration enable-login-with-password enable-access-tokens enable-backend-api-doc
```
Then restart:
```bash
docker compose -p penpot down && docker compose -p penpot up -d
```

---

## File Import Issues

### `.penpot` file import returns "zip END header not found"

**Cause:** The `.penpot` file is corrupted or uses a non-standard format.

**Fix:**
- Verify the file is a valid ZIP: `file template.penpot` should show "Zip archive data"
- If it shows "data" instead of "Zip archive" → the file is corrupted
- Try downloading from Penpot Hub (SaaS) instead of GitHub raw download
- Some files in the `penpot/penpot-files` repo may have encoding issues

### Import API returns 401 Unauthorized

**Cause:** Empty or invalid token in the Authorization header.

**Fix:**
- Verify token is not empty: `echo $PENPOT_ACCESS_TOKEN`
- If using fish shell, note that `source file` doesn't work with `VAR=VALUE` format
- Extract token correctly:
  ```bash
  # Bash
  source ~/penpot/.penpot-credentials

  # Fish shell (bash-style source doesn't work)
  set TOKEN (grep PENPOT_ACCESS_TOKEN ~/penpot/.penpot-credentials | cut -d= -f2-)
  ```

### Import creates duplicate files

**Cause:** API request was retried after a timeout, but the first request actually succeeded.

**Fix:**
- After import, always verify with `list_files(projectId)`
- If duplicates found, use `delete_file(fileId)` to remove extras

### Import API returns SSE stream instead of JSON

**Cause:** This is normal behavior. The import endpoint returns Server-Sent Events (SSE) format.

**Fix:** Parse the response as SSE — look for lines starting with `data:`. The final `data:` line contains the import result.

---

## Performance Tips

1. **Install globally** to avoid npx startup delay:
   ```bash
   npm install -g @zcubekr/penpot-mcp-server
   ```

2. **Use field filters** in `query_shapes` to reduce response size:
   ```
   query_shapes(fileId, pageId, {fields: ["id", "name", "type", "x", "y"]})
   ```

3. **Batch your operations** — plan your changes before executing, rather than querying after each individual change.

---

## Getting Help

If your issue isn't covered here:
1. Check the [GitHub Issues](https://github.com/zcube/penpot-mcp-server/issues)
2. Open a new issue with:
   - MCP server version (`npm list -g @zcubekr/penpot-mcp-server`)
   - Penpot version (Settings > About in Penpot UI)
   - Error message and reproduction steps

---

*This guide is maintained by the community. Found a new issue and fix? Please contribute!*

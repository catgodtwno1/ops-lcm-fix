---
name: ops-lcm-fix
description: Diagnose and fix LCM (Lossless-Claw Memory) compaction failures. Covers the false-positive auth error bug where summary content containing auth-related keywords triggers incorrect provider_auth failure classification, and the MiniMax apiKey env-marker issue. Use when LCM compaction fails silently, gateway logs show auth errors for summary model, or context grows unbounded.
---

# LCM Compaction Fix

Diagnose and repair LCM (lossless-claw) compaction failures in OpenClaw.

## Known Bugs

### Bug 1: False-positive auth error from summary content (v0.5.1)

**Upstream issue:** [Martian-Engineering/lossless-claw#171](https://github.com/Martian-Engineering/lossless-claw/issues/171)

**Symptom:** LCM logs `provider auth error` but the summary model (e.g., Haiku) actually succeeded. The error `Detail:` field contains readable summary text instead of a JSON error response.

**Root cause:** `pickAuthInspectionValue()` in `src/summarize.ts` returns the full API response when no error-related fields are found. `collectAuthFailureText()` then walks `content[].text` (the actual summary), and if it contains words like "401", "unauthorized", or "invalid api key" (because the conversation being summarized discussed auth errors), `AUTH_ERROR_TEXT_PATTERN` matches → false positive.

**Fix:** In `src/summarize.ts`, function `pickAuthInspectionValue()` (~line 391):

```diff
-  return Object.keys(subset).length > 0 ? subset : value;
+  // Return empty object when no error-related fields found, so that
+  // collectAuthFailureText does NOT walk assistant content which could
+  // contain auth-related keywords from the conversation being summarized.
+  return Object.keys(subset).length > 0 ? subset : {};
```

**How to tell it's this bug (not a real auth error):**
- Real auth error: `Detail: 401 {"error":{"type":"..."}}`
- False positive: `Detail: assistant text Summary of conversation about...`

### Bug 2: MiniMax apiKey env-marker literal (OpenClaw core)

**Symptom:** LCM compaction returns real HTTP 401 with "invalid api key" when using MiniMax models.

**Root cause:** `models.json` stores the apiKey as the literal string `"MINIMAX_API_KEY"` instead of the actual key value. Every gateway restart regenerates this file, overwriting manual fixes.

**Workaround:** Don't use MiniMax for `summaryModel`. Use Anthropic models instead:

```json
{
  "plugins": {
    "entries": {
      "lossless-claw": {
        "config": {
          "summaryModel": "anthropic/claude-haiku-4-5"
        }
      }
    }
  }
}
```

## Diagnostic Steps

### 1. Check LCM status

```bash
# Total summaries and recent activity
sqlite3 ~/.openclaw/lcm.db "
  SELECT count(*) as total FROM summaries;
  SELECT model, count(*) as cnt, max(created_at) as last_at 
  FROM summaries GROUP BY model ORDER BY last_at DESC;
"
```

### 2. Check for compaction errors in gateway log

```bash
# Look for auth errors vs successful compressions
sqlite3 ~/.openclaw/lcm.db "
  SELECT summary_id, model, token_count, created_at 
  FROM summaries ORDER BY created_at DESC LIMIT 10;
"
```

### 3. Verify the plugin is enabled

```bash
python3 -c "
import json
with open('$HOME/.openclaw/openclaw.json') as f: cfg = json.load(f)
lcm = cfg.get('plugins',{}).get('entries',{}).get('lossless-claw',{})
print('enabled:', lcm.get('enabled', 'not set'))
print('summaryModel:', lcm.get('config',{}).get('summaryModel', 'default'))
print('threshold:', lcm.get('config',{}).get('threshold', 'default'))
"
```

### 4. Apply the patch

```bash
# Find the line to patch
grep -n "Object.keys(subset).length > 0 ? subset : value" \
  ~/.openclaw/extensions/lossless-claw/src/summarize.ts

# Apply fix (replace 'value' with '{}')
sed -i '' 's/Object.keys(subset).length > 0 ? subset : value/Object.keys(subset).length > 0 ? subset : {}/' \
  ~/.openclaw/extensions/lossless-claw/src/summarize.ts
```

### 5. Restart gateway

```bash
launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway
```

### 6. Verify fix

Wait for compaction to trigger (context reaches threshold), then:

```bash
sqlite3 ~/.openclaw/lcm.db "
  SELECT summary_id, model, created_at 
  FROM summaries WHERE created_at > datetime('now', '-1 hour')
  ORDER BY created_at DESC;
"
```

## Recommended Config

- **summaryModel:** `anthropic/claude-haiku-4-5` (cheapest Anthropic, ~$0.001/compression)
- **threshold:** `0.75` (triggers at 75% context usage)
- **Avoid MiniMax** for summaryModel due to Bug 2

## Version Info

- lossless-claw: v0.5.1
- Bug 1 status: patched locally, upstream issue filed (#171)
- Bug 2 status: workaround only (use Anthropic models)

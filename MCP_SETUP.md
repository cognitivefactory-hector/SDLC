# MCP Setup — Observability Servers

Wiring the starter-path observability tools into Claude Code as MCP servers, so you
can query traces and errors by prompt instead of leaving the editor.

## What's an MCP server here vs. what isn't

| Starter-path tool | MCP server? | How you actually use it |
|---|---|---|
| **Sentry** | ✅ Yes (hosted) | Added below — query/debug prod errors from the editor |
| **Langfuse** | ✅ Yes (self-hosted) | Add *after* you deploy Langfuse (its MCP lives on your instance) |
| **OpenTelemetry** | ❌ No | Instrumentation you add **in code**; it *emits* traces to Langfuse |
| **DeepEval** | ❌ No | A **test library** (`pip install deepeval`); runs in CI, not as MCP |

---

## 1. Sentry — added now ✅

Configured in [`.mcp.json`](.mcp.json) (project scope):

```json
{ "mcpServers": { "sentry": { "type": "http", "url": "https://mcp.sentry.dev/mcp" } } }
```

**To activate:**
1. Reopen this folder in Claude Code (it picks up `.mcp.json` and asks you to approve the new server).
2. On first use you'll be sent through **OAuth** — log in with your Sentry account. No API key to store.
3. Verify: `claude mcp list` should show `sentry … ✔ Connected`.

**Want it in *every* project, not just this folder?** Skip `.mcp.json` and run:
```bash
claude mcp add -s user --transport http sentry https://mcp.sentry.dev/mcp
```

---

## 2. Langfuse — add once it's deployed ⏳

Langfuse's MCP server runs *on your Langfuse instance* at `/api/public/mcp`, so it only
exists after starter-path **step 1** (stand up self-hosted Langfuse). Then:

```bash
# Token is base64 of "PUBLIC_KEY:SECRET_KEY" from your Langfuse project settings
export LANGFUSE_MCP_TOKEN=$(printf '%s' "pk-lf-xxxx:sk-lf-xxxx" | base64)

claude mcp add --transport http langfuse \
  https://YOUR-LANGFUSE-HOST/api/public/mcp \
  --header "Authorization: Basic ${LANGFUSE_MCP_TOKEN}"
```

For local dev the host is `http://localhost:3000`. Once added, you can prompt things like
*"show me the slowest Langfuse trace today"* or *"what did the last run cost in tokens?"*.

---

## 3. OpenTelemetry & DeepEval — not MCP, for reference

- **OpenTelemetry**: add the OTel SDK to your app; point its exporter at Langfuse. This is
  what *produces* the traces the Langfuse MCP later reads. Code, not config.
- **DeepEval**: `pip install deepeval`; write eval cases (see the HTML guide §2–3) and run
  `deepeval test run` in CI. A test runner, not a server.

---

## Quick checklist

- [ ] Approve `sentry` server on reopen, complete OAuth
- [ ] `claude mcp list` shows sentry connected
- [ ] (later) Deploy Langfuse → add its MCP with the command above
- [ ] Instrument one app with OpenTelemetry → Langfuse
- [ ] Add DeepEval to that app's CI

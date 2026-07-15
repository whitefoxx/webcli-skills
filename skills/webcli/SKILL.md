---
name: webcli
description: Use to drive the user's real, logged-in Chrome through the WebCLI extension — a headless, agent-free browser bridge. Open pages, read/extract content (get_page_text / get_html / get_dom_outline), see and operate the page (get_interactives → click / type_into / select_option / press_key / scroll_page), take screenshots, and manage tabs. GENERIC browser tools only (no site adapters, no in-browser LLM). Talk to it over plain HTTP (curl) — no MCP setup. Reach for this on any "use my browser", "read this page while I'm logged in", "scrape/automate this page", or "click/fill this form for me" request.
---

# WebCLI

**The user's logged-in browser, as your tools — generic and headless.** WebCLI is a
Chrome extension with NO in-browser agent and NO site adapters: just the generic
browser primitives, exposed to you over a local bridge you talk to with `curl`.
Everything runs on the user's machine, in their real Chrome session (no re-auth).

## 1. Start the bridge

The WebCLI extension dials OUT to a local daemon (default port **8788**). Start it:

```bash
npx -y github:whitefoxx/webcli-skills            # runs the bridge on 8788
# or a specific port:  BRIDGE_PORT=8788 npx -y github:whitefoxx/webcli-skills
```

Then check it's connected (the extension auto-connects within ~1 min; the user may
need to load/enable the **WebCLI** extension once):

```bash
curl -s http://127.0.0.1:8788/status
# → {"ok":true,"connected":true,"port":8788,"client":"webcli","tools":23}
```

If `connected:false`, ask the user to open `chrome://extensions`, confirm **WebCLI**
is loaded/enabled, and (if it just started) reload it once. It reconnects on its own
after that.

## 2. Drive it (all over HTTP)

- **List the exact tools + their args:** `curl -s http://127.0.0.1:8788/tools`
  (this is the source of truth — tool names are `generic__<name>`).
- **Call a tool:**

  ```bash
  curl -s http://127.0.0.1:8788/command \
    -d '{"tool":"generic__open_url","args":{"url":"https://example.com"}}'
  # → {"ok":true,"result":{"tabId":123,"url":"https://example.com","active":false}}
  ```

## 3. The core loop

Read a page:

```bash
# One-shot open + read + close:
curl -s .../command -d '{"tool":"generic__get_page_text","args":{"url":"https://…","format":"markdown"}}'
# Or read an already-open tab: generic__get_text_from_tab {tab_id}
```

Operate a page (perceive → act → verify):

```bash
open_url {url}              → {tabId}
get_interactives {tab_id}   → {links,buttons,inputs,…} each with a `ref`
click {tab_id, ref}         # or type_into {tab_id, ref, text} / select_option / press_key
get_text_from_tab {tab_id}  # verify the result; screenshot {tab_id} for a visual check
close_tab {tab_id}          # clean up background tabs you opened
```

## 4. The tool surface (generic only)

Navigation/content: `open_url`, `get_page_text`, `get_text_from_tab`, `get_html`,
`get_dom_outline`, `screenshot`, `scroll_page`, `close_tab`. Tabs: `list_tabs`,
`get_active_tab`, `manage_tabs`. Perceive+interact: `get_interactives`, `click`,
`click_by_text`, `type_into`, `select_option`, `press_key`, `hover`. Search/scan:
`find_in_page`, `query_dom`, `find_in_dom`, `wait_for_selector`, `read_more`.

That's the whole surface — there are **no** site-specific adapters and **no**
`web_task`/agent tool. If a call returns "tool not found", it's not part of WebCLI
(you may be thinking of the full Web Agent extension). Always trust `GET /tools`.

## Notes

- Writes/interactions (click/type/submit) act on the user's real session — be
  deliberate; don't batch destructive actions.
- Tabs you open in the background pile up; close them when done.
- The extension ID (for `chrome.runtime.connect` from a web app like browser-md)
  is shown in the WebCLI toolbar popup, with a copy button.

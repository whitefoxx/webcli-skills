# webcli-skills

Bridge daemon + agent skill for **WebCLI** — a headless, agent-free Chrome
extension that exposes your logged-in browser's **generic browser tools** (open
pages, read/extract, click/type/scroll, screenshot, manage tabs) to external AI
agents (Claude Code, Codex, localmd). No site adapters, no in-browser LLM —
just the primitives, driven over plain HTTP.

It's the "pure provider" sibling of the full [Web Agent](https://github.com/whitefoxx/web-agent-skills)
extension: same transport, generic tools only.

## Install the skill (one command)

```bash
npx skills add whitefoxx/webcli-skills -g
```

(auto-detects Claude Code / Cursor / Codex …; reads `skills/webcli/SKILL.md`.)
Or just hand this repo URL to your AI and let it read the skill.

## Run the bridge

```bash
npx -y github:whitefoxx/webcli-skills        # daemon on 127.0.0.1:9376
# custom port:  BRIDGE_PORT=8790 npx -y github:whitefoxx/webcli-skills
```

The WebCLI extension dials the daemon automatically (default port **9376**, a
distinct port from the full Web Agent bridge's 8787 so both can run at once).

## Drive it

```bash
curl -s http://127.0.0.1:9376/status                       # is the extension connected?
curl -s http://127.0.0.1:9376/tools                        # the generic tool catalog (source of truth)
curl -s http://127.0.0.1:9376/command \
  -d '{"tool":"generic__open_url","args":{"url":"https://example.com"}}'
```

Full driving guide: [`skills/webcli/SKILL.md`](./skills/webcli/SKILL.md).

## HTTP API (binds 127.0.0.1 only)

| Method + path   | Result                                                      |
| --------------- | ----------------------------------------------------------- |
| `GET /ping`     | `{ok:true}`                                                 |
| `GET /status`   | `{ok, connected, port, client, tools}`                      |
| `GET /tools`    | `{ok, tools:[…]}` — generic tools in OpenAI-tool shape      |
| `POST /command` | body `{tool, args}` → `{ok, result}` or `{ok:false, error}` |

## Web app integration (no bridge, no extension id)

A web app can call the same tools directly from its own pages — **if, and only
if, the user added its origin** under **Web app access** in the WebCLI toolbar
popup (the default list is empty; there are no built-in origins). WebCLI then
injects a tiny relay into that origin's pages, and the page talks to it over
`window.postMessage`:

```js
// 1. Detect the relay via the DOM marker — synchronous, race-free:
//    document.documentElement.dataset.webcliRelay  // the extension id, set at document_start
//    (a `ready` frame {webcli:'mcp',dir:'to-page',ext,ready:true} is ALSO posted,
//    but in practice it can dispatch before your listener exists — treat it as a
//    bonus, not the detection mechanism)
// 2. Send JSON-RPC (MCP): initialize / tools/list / tools/call —
window.postMessage(
  { webcli: 'mcp', dir: 'to-ext', ext, // echo `ext` from the ready frame
    msg: { jsonrpc: '2.0', id: 1, method: 'tools/call',
           params: { name: 'generic__list_tabs', arguments: {} } } },
  window.location.origin,
);
// 3. Responses arrive the same way:
//    { webcli:'mcp', dir:'to-page', ext, msg:{ jsonrpc:'2.0', id:1, result:{…} } }
window.addEventListener('message', (e) => {
  const d = e.data;
  if (d?.webcli === 'mcp' && d.dir === 'to-page' && d.msg?.id === 1) console.log(d.msg);
});
```

Always echo `ext` (from the DOM marker or any received frame): a user can have
two WebCLI installs (store + dev), and an untargeted frame would be executed by
both. Your first frame may omit `ext` and learn it from the reply envelope.

## Layout

- `server.mjs` — the bridge daemon (`npx`/`node` entry, `webcli-bridge` bin).
- `skills/webcli/SKILL.md` — the drop-in agent skill.

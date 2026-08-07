# webcli-skills

Bridge daemon + agent skill for **WebCLI** — a headless, agent-free Chrome
extension that exposes your logged-in browser's **generic browser tools** (open
pages, read/extract, click/type/scroll, screenshot, manage tabs) to external AI
agents (Claude Code, Codex, …). No site adapters, no in-browser LLM — just the
primitives, driven over plain HTTP.

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

That daemon is the whole surface **from 0.3.0 on**. 0.2.0 also shipped a second
path — a postMessage relay letting a **web page** call the tools from an origin
the user allowlisted — and 0.3.0 removes it: WebCLI is the bridge for CLI
agents, and browser-page access moved to a dedicated companion extension built
for the one web app that used it.

If you are on 0.2.0 and relying on that path, it stops working when you update.
Your configured origin list is cleared on update, so rolling back does not
restore it.

## Layout

- `server.mjs` — the bridge daemon (`npx`/`node` entry, `webcli-bridge` bin).
- `skills/webcli/SKILL.md` — the drop-in agent skill.

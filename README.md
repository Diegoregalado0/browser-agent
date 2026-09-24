# Browser Agent

Version 1.0.0

An agent that carries out tasks in Chrome from plain-language instructions. It reads pages,
clicks, types, and navigates through the Chrome DevTools Protocol, using a model from
Anthropic, OpenAI (or any OpenAI-compatible endpoint), Google Gemini, Mistral, or Ollama.

It ships in two editions that share the same agent loop, tools, providers, and UI:

| | Local | Extension |
|---|---|---|
| Runs as | Node server + dedicated Chrome profile | Chrome side panel, no server |
| Page control | DevTools port | `chrome.debugger` |
| Desktop control | Yes (macOS) | No |
| Platforms | macOS | Any desktop Chrome 120+ |
| API keys | `config.json` or environment | `chrome.storage.local` |

## Requirements

- Node 22 or newer
- Google Chrome
- macOS and the Xcode command line tools, for the local edition's desktop helper

## Local edition

```sh
npm install
./scripts/install.sh
```

This installs the `browser-agent` command and `~/Applications/Browser Agent.app`.

```sh
browser-agent            # start the server if needed and open the agent browser
browser-agent window     # open the chat as a separate app window
browser-agent serve      # run in the foreground (--port=7788, --no-open)
browser-agent status
browser-agent stop
browser-agent logs
browser-agent activity   # recent tasks, actions, and safety-check decisions
```

The agent browser is Chrome with its own profile in `~/.browser-agent/chrome-profile`. Sign in
to sites and install extensions there once; they persist. The chat is in its side panel
(Cmd+Shift+Y).

API keys are set in Settings or read from `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`,
`GEMINI_API_KEY`, and `MISTRAL_API_KEY`, including from a `.env` file in the project folder.

Desktop control needs Accessibility and Screen Recording permission for the app that
launches the server (Browser Agent.app or your terminal). Request them in Settings, grant
them in System Settings, then restart with `browser-agent stop && browser-agent`.

`BROWSER_AGENT_HOME` moves all data to another folder, which is useful for testing.

## Extension edition

```sh
npm install
npm run build:extension
```

Output:

- `dist/extension/`: load it from `chrome://extensions` with "Load unpacked"
- `dist/browser-agent-extension.zip`: the Chrome Web Store package

Open the panel from the toolbar icon or Cmd+Shift+Y (Ctrl+Shift+Y on Windows and Linux).
Each window's panel runs its own agent, which only works in that window's tabs. Keys stay in
the browser profile, are never synced, and are sent only to their provider. Closing the
panel stops the task. For Ollama, start it with `OLLAMA_ORIGINS=chrome-extension://*`.

## Providers

| Provider | Default model | Notes |
|---|---|---|
| OpenAI | `gpt-6-sol` | Responses API. A custom base URL switches to Chat Completions. |
| Anthropic | `claude-opus-5` | Adaptive thinking and effort where supported. |
| Gemini | none | Pick a model in Settings. |
| Mistral | `mistral-medium-latest` | Sends only the newest 4 screenshots per request. |
| Ollama | none | Needs a tool-capable model; a vision model to read screenshots. |

The history is provider-neutral, so the provider can change mid-conversation.

## Tools

`browser` (screenshot, click, type, keys, scroll, drag), `navigate`, `read_page`, `find`,
`form_input`, `get_page_text`, `tabs`, `network_requests`, and, when developer tools are
on, `javascript_exec` and `edit_html`. The local edition adds `desktop`.

A task opens its own tabs and never takes over tabs it did not open. The model decides
between reusing its current tab and opening a new one (`navigate` with `new_tab`). The tab
it is working in sits in a tab group labeled "Agent" while the task runs.

## Safety and limits

- **Modes**: Guarded (default), Guarded with a prompt for each new site, and Auto. Guarded
  runs a small model that checks each state-changing action against the request and scans
  page content for prompt injection. Checks fail closed.
- **Always asks**: clicks, typing, and form changes on sensitive sites (banks, payments,
  password managers, account security, crypto exchanges, plus your own list), and typing
  into password fields. This applies in every mode.
- **Limits**, all adjustable, 0 to disable:

  | Limit | Default | When reached |
  |---|---|---|
  | Model requests per minute | 20 | pauses |
  | Browser actions per minute | 60 | pauses |
  | Tokens per task | 2,000,000 | stops the task |
  | Tokens per day | 10,000,000 | stops and refuses new tasks |

  Safety-check calls count toward the token limits.
- **Extension only**: opens http(s) pages only, and `javascript_exec` / `edit_html` start off.

## Data

| Data | Local | Extension |
|---|---|---|
| Settings and keys | `~/.browser-agent/config.json` (mode 600) | `chrome.storage.local` |
| Conversations | `~/.browser-agent/sessions/` | IndexedDB |
| Activity log | `~/.browser-agent/activity.log` | none |
| Daily token count | `~/.browser-agent/usage.json` | `chrome.storage.local` |

Screenshots are never saved. Ghost mode saves no conversation and writes no log entries;
it is always on in incognito windows.

## Layout

```
bin/browser-agent.js      CLI and launcher
src/controller.js         conversation state, sessions, Ghost mode, UI messages
src/agent.js              agent loop
src/browser-tools.js      page tools, on top of a transport
src/transport-cdp.js      transport: DevTools port (local)
extension/                transport: chrome.debugger, storage, entry point (extension)
src/providers/            model providers
src/guard.js              safety checks
src/limits.js             rate and token limits, sensitive sites
src/server.js             HTTP and WebSocket server (local)
src/chrome-keeper.js      launches the agent Chrome and holds its DevTools pipe
native/oshelper.swift     mouse, keyboard, and permissions helper (macOS)
ui/                       chat and settings UI
scripts/                  installer and extension build
```

## Checks

```sh
npm run check    # syntax check of every source file
```

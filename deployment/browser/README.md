# Browser

Serves a page with a call button. Useful for iterating on an agent before putting it on a phone number.

## 1. Publish an agent

```sh
AGENT=http-tools python publish.py
```

## 2. Run it

```sh
python deployment/browser/server.py
```

Open http://localhost:3000 and start the call.

## 3. Iterate

Edit the file in [agents/](../../agents/), run `python publish.py`, and start another call. When the agent behaves the way you want, see [deployment/telephony](../telephony/).

---

## What it does

Publishes `agents/<AGENT>.jsonc` on startup if it has no id yet, so a fresh clone works with only an API key.

`GET /token` proxies AssemblyAI's token endpoint using your key and returns a 60 second session token. The key is never sent to the page.

The page streams the microphone as 24 kHz PCM16 over `wss://agents.assemblyai.com/v1/ws`, plays the reply back, and discards queued audio when you interrupt. Capture and playback each run in their own AudioContext with a resampling worklet, so a browser that refuses to open a context at 24 kHz still sounds right.

The side pane has two tabs. Events lists every websocket frame in both directions, with audio runs collapsed into counts. Agent shows the published agent as the API stored it, read only, served by `GET /agent`. Tool header values and LLM keys are stripped from that response, but the system prompt is in it, so on a public deployment anyone opening the page can read it.

The session message contains only `{ agent_id }`. Prompt, voice, tools and turn detection are read from the stored agent, which is why the browser and the phone behave the same.

## Tool calls

`POST /tool/send_summary` is where the `send_summary` http tool lands. AssemblyAI posts the agent's arguments there as JSON, the handler prints them, and it returns `{"sent": true}`. That print is the stub; the real SMS send replaces it and nothing above it has to change.

Reach it from a test call, without a voice session:

```sh
curl -X POST http://localhost:3000/tool/send_summary \
  -H 'Content-Type: application/json' \
  -d '{"caller_name":"Dana Whitfield","callback_number":"+1 415 555 2671","problem_description":"burst pipe under the kitchen sink","urgency":"high"}'
```

### It needs a public https URL locally

The tool URL in `agents/calldesk.jsonc` is a stored-agent setting, and **AssemblyAI makes the request from its own servers**, not from the browser tab. That has two consequences:

- `http://` is rejected by the API outright: `webhook URL must use https://`.
- `127.0.0.1` and `localhost` resolve on AssemblyAI's machine, so they never reach this process.

For local development, run a tunnel and point `CALLDESK_TOOL_URL` at it:

```sh
ngrok http 3000
```

Then set `CALLDESK_TOOL_URL` to the `https://` URL ngrok prints and re-run `AGENT=calldesk python publish.py`. The server prints a reminder at startup if it sees a loopback address. Without this the agent still collects everything correctly, it just never reaches `send_summary`, and the failure is silent from the caller's side.

Once deployed, the URL is your own HTTPS endpoint and the tunnel is no longer involved.

## Environment

| | |
| --- | --- |
| `ASSEMBLYAI_API_KEY` | Required. Stays in this process. |
| `AGENT` | Which file in `agents/` to serve. Defaults to `minimal`. |
| `AGENT_ID_<NAME>` | The id `python publish.py` saved for that file. Connected to as it is. |
| `AGENT_ID` | Overrides the per-file keys, for serving one specific agent. |
| `PORT` | Defaults to 3000, moves to the next free port if taken. |
| `CALLDESK_TOOL_URL` | Where the `send_summary` tool posts. Must be a public https URL; see above. |

## Editing the page

The server is [server.py](server.py), the page is [index.html](index.html) and the client is [app.js](app.js), all served as they are. Save and refresh.

## Hosting

`render.yaml` is configured for one-click deploys. Render prompts for `ASSEMBLYAI_API_KEY` during Blueprint creation, since that is the only variable marked `sync: false`, and sets `PORT` itself. `AGENT` and `AGENT_ID` arrive with defaults and are editable under Environment on the service.

With no id set the service publishes `AGENT` on boot and updates the agent of that name on later restarts, so restarts do not pile up duplicate agents. Setting `AGENT_ID` to the id from your `.env` is still better: the deployment then serves the same agent you tested locally.

Anyone with the URL can start sessions billed to your key.

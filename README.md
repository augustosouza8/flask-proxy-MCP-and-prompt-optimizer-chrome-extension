# Prompt Optimizer — Flask Proxy (MCP + Agno + Chrome Extension)

A minimal Flask proxy that sits between the **Prompt Optimizer MCP server** (over SSE via Agno) and the **Chrome extension**. It exposes a single, CORS‑enabled endpoint — `/optimize` — so the extension (or any client) can request an optimized prompt without embedding API keys in the browser.

---

## What this repo does

* **Single endpoint:** `POST /optimize` → returns `{ "optimized_prompt": "..." }`.
* **Bridges to MCP:** Uses **Agno** + **MCPTools** to call your MCP server via **Server‑Sent Events (SSE)**.
* **Model provider:** Runs Groq’s `qwen/qwen3-32b` model through Agno.
* **CORS enabled:** Accepts cross‑origin requests (intended for the Chrome extension).
* **Debug friendly:** Prints tool call traces when `debug_mode=True` (default in code for local dev).

## Quick start

### 1) Install

```bash
pip install -r requirements.txt
```

### 2) Configure environment

Create a `.env` file (or export as env vars):

```bash
GROQ_API_KEY=your_groq_api_key          # required
PORT=8000                               # optional; defaults to 8000
```

If you host your own MCP server, update `_SSE_URL` in `app.py`.

### 3) Run locally

```bash
python app.py
# Server listens on 0.0.0.0:${PORT:-8000}
```

### 4) Test the endpoint

```bash
curl -X POST http://127.0.0.1:8000/optimize \
  -H "Content-Type: application/json" \
  -d '{"prompt":"Write a cheerful out-of-office message."}'
```

**Response**

```json
{"optimized_prompt":"..."}
```

## Endpoint contract

**POST** `/optimize`

* **Request body**: `{ "prompt": "<original user prompt>" }`
* **Response**: `{ "optimized_prompt": "<rewritten prompt>" }`
* **Errors**: `400` if `prompt` missing; `500` on backend/MCP issues (JSON `{ "error": "..." }`).

## How it works (under the hood)

1. Flask receives the prompt and calls `query_agent()`.
2. `query_agent()` runs the async agent (Agno) that:

   * opens an SSE connection to the MCP server (`MCPTools(server_params=get_sse_params(), transport="sse")`),
   * calls the **Prompt Optimizer** MCP tool,
   * returns the tool output **verbatim**.
3. Flask wraps the result into `{ "optimized_prompt": ... }`.

**Timeouts:** SSE connect/read timeouts are set to **300s** by default in `get_sse_params()`.

## Deploy (example with Gunicorn)

```bash
gunicorn app:app --bind 0.0.0.0:${PORT:-8000}
```

### Render/Heroku tips

* Set `GROQ_API_KEY` in dashboard.
* Ensure your MCP server (e.g., on Hugging Face Spaces) is reachable from the deployment.
* If your MCP host sleeps, first request may be slow while it wakes.

## Security & reliability notes

* **CORS** is currently permissive for development. In production, restrict origins to your extension/site.
* Consider adding a **shared secret** header or simple **rate‑limit** (e.g., behind a reverse proxy) if the endpoint is public.
* The service is **stateless** and does not store prompts or results.

## Project structure

```
flask-proxy-mcp-and-prompt-optimizer-chrome-extension/
├── app.py             # Flask app + Agno agent + MCP SSE client
├── requirements.txt   # Flask, flask-cors, agno, groq, mcp, dotenv, gunicorn
├── LICENSE            # MIT
└── README.md          # You are here
```

## Troubleshooting

* **500: "Environment variable 'GROQ\_API\_KEY' is not set."** → Create `.env` or set the variable.
* **CORS error in browser** → Verify `flask_cors.CORS(app)` is active; restrict/allow origins as needed.
* **Long/idle responses** → Your MCP host may be sleeping; keep‑alive or upgrade the hosting plan.
* **Tool mismatch** → Ensure your MCP server exposes the Prompt Optimizer tool the agent expects.

## License

MIT — see `LICENSE`.

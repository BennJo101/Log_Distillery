# 🥃 Log Distillery

> *Hierarchical AI Summarization & Reduction — Small Batch*

Log Distillery takes a raw log file of any size, chunks it into numbered barrels, runs each chunk through an AI agent for a summary, then recursively distills those summaries — round after round — until a single refined report remains. After the final pour, it automatically generates four categories of actionable aftercare recommendations based on what was found in the logs.

---

## Files

| File | Description |
|------|-------------|
| `log_distillery.html` | Full version — choose between **Claude Direct** (Anthropic API) or **N8N Webhook** at runtime via the toggle |
| `log_distillery_n8n.html` | N8N-only version — streamlined, no API key UI, all AI calls route through your N8N webhook |

Both are single self-contained HTML files. No build step, no dependencies, no server required. Open in any modern browser.

---

## How It Works

```
Raw Log
   │
   ▼
Chunk into N-line barrels (default: 30 lines each)
   │
   ▼
Round 1 ── AI summarizes each barrel individually
   │        → [Summary 1] [Summary 2] [Summary 3] ... [Summary N]
   ▼
Round 2 ── Summaries batched (default: 6 per group) and re-distilled
   │        → [Summary A] [Summary B] [Summary C]
   ▼
Round 3+ ── Repeats until one summary remains (up to 10 rounds)
   │
   ▼
Final Pour ── Single distilled report streams to screen
   │
   ▼
Aftercare ── Four AI-generated recommendation panels fire automatically
             Log Hygiene · Incident Response · Monitoring · N8N Automation
```

---

## Quick Start

### `log_distillery.html` (Dual Mode)

1. Open `log_distillery.html` in your browser
2. Select your still using the **Claude Direct / OR / N8N Webhook** toggle:
   - **Claude Direct** — enter your Anthropic API key and choose a model
   - **N8N Webhook** — enter your N8N webhook URL and optional auth header
3. Drop a log file onto the drop zone, browse for one, or paste directly
4. Adjust **Mash Bill** (lines per chunk) and **Barrel Batch Size** if needed
5. Click **Fire the Still**

### `log_distillery_n8n.html` (N8N Only)

1. Open `log_distillery_n8n.html` in your browser
2. Enter your N8N Webhook URL and optional auth header
3. Drop, browse, or paste your log
4. Click **Fire the Still**

---

## Configuration

| Field | Default | Description |
|-------|---------|-------------|
| Mash Bill (lines/chunk) | 30 | Lines per barrel. Larger = fewer API calls, less granularity |
| Barrel Batch Size | 6 | How many summaries are grouped per distillation round |

**Recommended Mash Bill by log size:**

| Log Size | Mash Bill |
|----------|-----------|
| < 200 lines | 15–25 |
| 200–1,000 lines | 30 (default) |
| 1,000–5,000 lines | 75–100 |
| > 5,000 lines | 150–200 |

---

## Claude Direct Mode

Calls the Anthropic API directly from your browser. No proxy or server required.

**Supported models:**

| Model | Speed | Quality | Best For |
|-------|-------|---------|----------|
| `claude-sonnet-4` | Fast | High | General use — recommended |
| `claude-haiku-4-5` | Fastest | Good | Large logs, cost-sensitive |
| `claude-opus-4-6` | Slow | Highest | Complex logs needing deep analysis |

Your API key is never stored — it lives only in the input field for the duration of the session.

---

## N8N Webhook Mode

Each chunk and each distillation call is POSTed to your N8N webhook individually. N8N handles the AI call — Bedrock, Azure OpenAI, Ollama, or any LLM node — and returns the summary. This keeps all log content inside your own infrastructure.

### Payload sent to N8N (per call)

```json
{
  "system_prompt": "You are a log analysis AI...",
  "user_message":  "Summarize the following log section (lines 1–30):\n\n  1: 2024-01-15 08:00:01 INFO ...",
  "chunk_id":      1,
  "round":         1,
  "lines":         "1-30"
}
```

Aftercare calls use `"round": "aftercare"` and `"chunk_id"` set to one of:
`"hygiene"` · `"incident"` · `"monitoring"` · `"automation"`

This lets you route aftercare calls to a different model or workflow in N8N if desired.

### Expected N8N response

Return any of the following — the Distillery will find it automatically:

```json
{ "summary": "..." }
{ "text": "..." }
{ "output": "..." }
{ "message": "..." }
```

Or a plain string response body.

### Optional Auth Header

If your N8N webhook requires authentication, enter the full header value in the **Auth Header** field. It is sent as the `Authorization` header on every request.

```
Bearer your-token-here
```

---

## Setting Up N8N

### Option A — N8N Cloud (Easiest)

1. Sign up at [app.n8n.cloud](https://app.n8n.cloud)
2. Create a free account (includes a trial)
3. Skip to **Building the Workflow** below — your instance is ready immediately

### Option B — Self-Hosted with Docker (Recommended for Production / CMMC)

Self-hosting keeps all log data inside your own perimeter. Recommended for GovCloud, CMMC, FedRAMP, or any environment where data cannot leave your boundary.

**Prerequisites:**
- A Linux server or VM (Ubuntu 22.04+ recommended) with at least 2 GB RAM
- Docker v20.10+ installed (`docker -v` to verify)
- Port 5678 open in your firewall

**Step 1 — Create the N8N directory and set permissions**

```bash
mkdir ~/n8n && cd ~/n8n
mkdir n8n_data
sudo chown -R 1000:1000 n8n_data
```

**Step 2 — Create a `docker-compose.yml`**

```yaml
services:
  n8n:
    image: n8nio/n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=changeme
      - N8N_HOST=your-server-ip-or-domain
      - WEBHOOK_URL=http://your-server-ip-or-domain:5678/
      - GENERIC_TIMEZONE=America/Detroit
    volumes:
      - ./n8n_data:/home/node/.n8n
```

Replace `your-server-ip-or-domain`, `admin`, and `changeme` with your actual values.

**Step 3 — Start N8N**

```bash
docker compose up -d
```

N8N will be available at `http://your-server:5678`. The first time you open it you will be prompted to create an owner account.

**Step 4 — (Optional) Keep N8N updated**

```bash
docker compose pull && docker compose up -d
```

> **Tip for RheinMan / GovCloud:** Deploy inside your existing AWS GovCloud or Azure Government VNet. Set `WEBHOOK_URL` to your internal hostname so the Log Distillery can reach it from within the boundary without any traffic leaving the perimeter.

---

### Building the Log Distillery Workflow in N8N

Once your N8N instance is running:

**Step 1 — Create a new workflow**

Click **+ New Workflow** in the N8N editor.

**Step 2 — Add a Webhook trigger node**

- Click **+** to add a node, search for **Webhook**
- Set **HTTP Method** to `POST`
- Set **Path** to something descriptive, e.g. `log-distillery`
- Set **Respond** to `Using 'Respond to Webhook' Node`
- Copy the **Test URL** shown — you'll paste this into the Log Distillery's Webhook URL field

**Step 3 — Add an AI Agent node**

Connect an AI node after the Webhook trigger. Options:

- **Anthropic node** — set the system prompt to `{{ $json.system_prompt }}` and user message to `{{ $json.user_message }}`
- **AWS Bedrock node** — same expressions, routes through your GovCloud Bedrock endpoint
- **OpenAI / Azure OpenAI node** — same expressions, use your GPT-4.1 deployment at 300K TPM
- **HTTP Request node** — call any LLM API endpoint manually

**Step 4 — Add a Respond to Webhook node**

Connect it after the AI node. Set:
- **Respond With** → `JSON`
- **Response Body** → `{ "summary": "{{ $json.text }}" }` (adjust the field name to match your AI node's output field)

**Step 5 — Test it**

- Click **Listen for Test Event** on the Webhook node
- In the Log Distillery, paste the Test URL and fire a small log
- Watch the execution light up in N8N — confirm the response is received

**Step 6 — Activate for production**

- Toggle the workflow to **Active** (top right)
- Switch from the Test URL to the **Production URL** in the Log Distillery
- Production URL format: `http://your-server:5678/webhook/log-distillery`

> **Note:** N8N generates two URLs for every webhook — a Test URL (only active while the editor is open and listening, times out after 120 seconds) and a Production URL (always active while the workflow is published). Use the Test URL during development and the Production URL once everything is working.

---

## Aftercare Recommendations

After the final distilled summary is produced, four recommendation panels generate automatically using the same AI backend:

| Panel | Covers |
|-------|--------|
| **📦 Log Hygiene** | Archival strategy, rotation policy, retention periods, compression |
| **🚨 Incident Response** | Triage steps, error remediation, notification targets, post-incident review |
| **📊 Monitoring** | Alert thresholds, dashboard suggestions, anomaly detection rules, health check cadence |
| **⚙️ N8N Automation** | Workflow triggers, auto-ticketing, Slack/email alerts, escalation logic |

All four are driven by the actual distilled summary — specific to the events, errors, and patterns found in your logs, not generic boilerplate. Each card pulses copper while generating and turns gold when complete.

In N8N mode, aftercare calls arrive at your webhook with `"round": "aftercare"` so you can branch them to a different model or workflow if you want specialized handling per category.

---

## UI Overview

| Element | Function |
|---------|----------|
| Still selector | Toggle between Claude Direct and N8N Webhook (dual-mode version only) |
| Drop zone | Drag & drop a file, browse, or paste log text directly |
| Stats bar | Live estimate of line count, barrel count, and distillation rounds |
| Round cards | Each distillation round rendered with collapsible barrel tiles |
| Barrel tiles | Click any tile to open the full summary text in the detail drawer |
| Progress bar | Copper fill tracks pipeline completion |
| Final Pour | Distilled summary streams to screen character by character |
| Aftercare grid | Four cards generating independently with pulsing copper status dots |

---

## Compliance Notes

| Mode | Data Routing | Suitable For |
|------|-------------|--------------|
| Claude Direct | Log content → `api.anthropic.com` from browser | General use, non-sensitive logs |
| N8N Webhook | Log content → your N8N instance only | CMMC, FedRAMP, NIST, GCC High, GovCloud |

For CMMC/FedRAMP environments: run N8N inside your AWS GovCloud or Azure Government boundary, point it at Bedrock Claude or Azure OpenAI GPT-4.1, and no log content ever crosses your perimeter.

---

## Browser Compatibility

Chrome 90+, Edge 90+, Firefox 88+, Safari 15+. Requires `fetch`, `async/await`, and `navigator.clipboard`.

---

*Log Distillery — Est. by Artificial Intelligence · Small Batch*

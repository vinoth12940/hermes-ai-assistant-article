# I Built an Always-On AI Assistant for $6/Month — Here's the Full Setup Guide

## From Telegram Bot to a Personal AI That Never Sleeps

Imagine having an AI assistant that lives in your Telegram, responds to voice messages, sends you daily stock market briefings, tracks your favorite cricket scores, and can execute code, browse the web, and manage your server — all running autonomously on a $6/month VPS.

That's exactly what I built with **Hermes Agent**.

No OpenAI API bills burning a hole in my pocket. No complex cloud deployments. Just a single server and a Telegram bot that became my always-on AI companion.

Here's how I did it, step by step.

---

## Why I Wanted This

I was tired of opening ChatGPT in a browser every time I had a question. I wanted something that:

- Lives in my phone (Telegram)
- Understands voice messages
- Can actually do things (run code, search the web, manage files)
- Sends me proactive updates (market news, AI news)
- Costs almost nothing to run

After trying various solutions — ChatGPT API wrappers, custom LangChain setups, even running local LLMs — I found that none gave me the full package. Then I discovered Hermes Agent.

---

## What is Hermes Agent?

Hermes Agent is an open-source AI agent framework by Nous Research. Think of it as the "brain" that connects an LLM to real-world tools — terminal access, file operations, web browsing, cron scheduling, and messaging platforms like Telegram, Discord, Slack, and WhatsApp.

It's not just a chatbot. It's an **agent** — meaning it can think, plan, and execute multi-step tasks autonomously.

Key features that sold me:
- **Multi-platform messaging** — Telegram, Discord, Slack, WhatsApp, Signal
- **Voice messages** — Send voice, get voice back
- **Tool execution** — Run shell commands, browse the web, manage files
- **Scheduled tasks** — Cron jobs for daily briefings
- **Persistent memory** — Remembers things across sessions
- **Skills system** — Extensible with community skills
- **Smart model routing** — Use cheap models for simple tasks, powerful ones for complex ones

---

## The Hardware: A $6/month VPS

Here's the beauty — you don't need expensive hardware.

**My Setup:**
- **Provider:** Hetzner Cloud (Helsinki)
- **Plan:** CX22 (4 vCPU, 8 GB RAM, 150 GB NVMe)
- **OS:** Ubuntu 24.04 LTS
- **Cost:** ~$6/month (€5.47)
- **Hostname:** ubuntu-8gb-hel1-2

That's it. No GPU needed. No fancy hardware. Just a basic VPS.

> 💡 **Why no GPU?** Hermes uses cloud LLM APIs for inference, so the VPS only needs to run the agent software itself, which is lightweight. The heavy lifting (model inference) happens on the cloud provider's servers.

---

## Step 1: Server Setup

First, SSH into your fresh VPS and update the system:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3 python3-pip python3-venv git curl
```

Create a user for Hermes (don't run as root):

```bash
sudo useradd -m -s /bin/bash hermes
sudo usermod -aG sudo hermes
sudo su - hermes
```

---

## Step 2: Install Hermes Agent

```bash
git clone https://github.com/nousresearch/hermes-agent.git ~/.hermes/hermes-agent
cd ~/.hermes/hermes-agent
python3 -m venv venv
source venv/bin/activate
pip install -e .
```

That's the core installation. Hermes Agent is now on your server.

---

## Step 3: Choose Your LLM Provider

This is where you decide which AI "brain" powers your assistant. Hermes supports dozens of providers:

- OpenAI (GPT-4o, GPT-4, etc.)
- Anthropic (Claude)
- Google (Gemini)
- OpenRouter (access to 200+ models)
- Nous Research
- DeepSeek
- Z.AI / GLM (what I use)
- And many more...

**My choice: Z.AI Coding Plan**

I went with Z.AI's coding plan because it offers excellent value:
- **Model:** GLM-5 Turbo
- **Endpoint:** `https://api.z.ai/api/coding/paas/v4`
- **Cost:** Very affordable subscription (not pay-per-token)

Run the setup wizard:
```bash
cd ~/.hermes/hermes-agent
source venv/bin/activate
hermes setup
```

This walks you through provider selection, API key entry, and model configuration. It stores everything securely in `~/.hermes/.env`.

---

## Step 4: Connect Telegram

This is where it gets fun — your AI now lives in your phone.

1. **Create a Telegram Bot:**
   - Open Telegram, search for @BotFather
   - Send `/newbot`
   - Pick a name and username
   - Copy the bot token

2. **Configure Hermes:**
   ```bash
   hermes config set telegram.bot_token "YOUR_BOT_TOKEN"
   hermes config set telegram.allowed_users "YOUR_TELEGRAM_ID"
   hermes config set telegram.home_channel "YOUR_TELEGRAM_ID"
   ```

3. **Start the Gateway:**
   ```bash
   hermes gateway run
   ```

Now message your bot on Telegram. It works!

4. **Install as a Service (so it survives reboots):**
   ```bash
   hermes gateway install
   ```

This creates a systemd service that auto-starts Hermes on boot and restarts it if it crashes.

---

## Step 5: Enable Voice Messages

This was a game-changer. I can send voice notes while driving and get voice responses back.

**Speech-to-Text (STT) — understanding your voice:**

```bash
cd ~/.hermes/hermes-agent
source venv/bin/activate
pip install faster-whisper
```

In `~/.hermes/config.yaml`:
```yaml
stt:
  enabled: true
  provider: local
  local:
    model: base
```

The `base` model downloads automatically (~150 MB) and runs on CPU. It's fast enough for real-time transcription.

**Text-to-Speech (TTS) — the AI talks back:**

I used Microsoft Edge TTS (free, no API key needed):

```yaml
tts:
  provider: edge
  edge:
    voice: en-US-AriaNeural
```

Enable auto voice replies:
```yaml
voice:
  auto_tts: true
```

Now when I send a voice message, Hermes transcribes it, processes it, and replies with a voice note. The full pipeline:

```
Your Voice → Whisper (local STT) → GLM-5 Turbo (AI brain) → Edge TTS → Voice Reply
```

---

## Step 6: Enable Tools

Hermes has a rich set of tools. Enable them for your platform:

```bash
hermes tools
```

This opens an interactive selector. For Telegram, I enabled:

| Tool | What It Does |
|------|-------------|
| terminal | Run shell commands on the VPS |
| file | Read, write, search files |
| web | Web search and page extraction |
| browser | Browser automation |
| cronjob | Schedule recurring tasks |
| memory | Persistent cross-session memory |
| vision | Analyze images |
| code_execution | Run Python scripts |
| delegation | Spawn sub-agents for parallel tasks |
| tts | Voice message replies |
| skills | Load extensible skill sets |

After enabling tools, send `/reset` in Telegram to activate them.

---

## Step 7: Set Up Automated Daily Briefings

This is where Hermes went from "cool toy" to "actually useful." I set up three automated daily briefings that arrive in my Telegram every morning:

**1. US Market Daily Briefing**
```bash
hermes cron create \
  --name "US Market Daily Briefing" \
  --schedule "every 1440m" \
  --deliver telegram \
  --prompt "Generate a comprehensive US market briefing covering S&P 500, Dow, Nasdaq, sector performance, commodities, and bond yields. Include key movers and data points."
```

**2. Daily GenAI Briefing**
```bash
hermes cron create \
  --name "Daily GenAI Briefing" \
  --schedule "0 9 * * *" \
  --deliver telegram \
  --prompt "Search HackerNews, Reddit, GitHub trending, and Twitter for the latest AI/ML news. Summarize the top 10 stories with links."
```

**3. Indian Market Daily Briefing**
```bash
hermes cron create \
  --name "Indian Market Daily Briefing" \
  --schedule "every 1440m" \
  --deliver telegram \
  --prompt "Generate a briefing on Sensex, Nifty 50, sector performance, USD/INR, and FII/DII activity."
```

These run autonomously — Hermes wakes up, does the research, writes the briefing, and delivers it to my Telegram. I wake up to fresh market news every morning.

---

## Step 8: Persistent Memory

One feature that makes Hermes feel like a real assistant is its persistent memory system.

It remembers:
- My name, preferences, and communication style
- Facts about my environment (server specs, installed tools)
- Corrections I've made ("don't include Indian markets unless I ask")
- Project conventions and pitfalls we've discovered together

This memory carries across sessions. I don't have to repeat myself. It's like having an assistant who actually pays attention.

Memory is stored in two places:
- **User profile** — who I am, my preferences
- **Agent notes** — environment facts, lessons learned, tool quirks

Memory usage is automatically managed — old entries get flushed when limits are reached.

---

## Step 9: Skills System

Hermes has a skills system that extends its capabilities. Think of skills as reusable recipes.

Some of my favorites:
- **claude-code / codex** — Delegate coding tasks to Claude or Codex
- **jupyter-live-kernel** — Interactive data analysis
- **github-pr-workflow** — Full pull request lifecycle
- **youtube-content** — Fetch and summarize YouTube videos
- **polymarket** — Query prediction market data
- **plan** — Structured planning for complex tasks

Install skills from the hub:
```bash
hermes skills search "docker"
hermes skills install docker-management
```

---

## Step 10: Maintenance & Monitoring

To keep everything running smoothly, I set up automated maintenance:

```bash
# Maintenance script runs every 5 minutes via system crontab
crontab -e
# Add: */5 * * * * /home/hermes/.hermes/scripts/maintenance.sh
```

The maintenance script handles:
- Gateway health checks
- Memory and disk monitoring
- Log rotation (keeps logs under 1 MB)
- Telegram polling verification
- Cron job status checks
- Auto-update every 3 days (with passwordless sudo)

**Monitoring commands:**
```bash
hermes status    # Full system status
hermes doctor    # Diagnostic checks
hermes cron list # View scheduled jobs
hermes update    # Update to latest version
```

---

## The Config That Makes It All Work

Here's a simplified version of my `~/.hermes/config.yaml`:

```yaml
model:
  base_url: https://api.z.ai/api/coding/paas/v4
  default: glm-5-turbo
  provider: zai

agent:
  max_turns: 60
  reasoning_effort: medium

stt:
  enabled: true
  provider: local
  local:
    model: base

tts:
  provider: edge
  edge:
    voice: en-US-AriaNeural

voice:
  auto_tts: true

memory:
  memory_enabled: true
  flush_min_turns: 6

compression:
  enabled: true
  target_ratio: 0.2

session_reset:
  at_hour: 4
  idle_minutes: 1440
  mode: both
```

This is surprisingly minimal for what it accomplishes.

---

## What My Daily Workflow Looks Like

**Morning:** Wake up to three briefing messages — US markets, Indian markets, and GenAI news — already waiting in Telegram.

**Throughout the day:**
- "What happened in the RCB match?" → Get a full scorecard
- Voice message: "Can you check the latest on Qwen 3.5?" → Get a researched answer
- "Write a Python script to parse this CSV" → Get working code
- "Deploy this to GitHub" → Done automatically

**Evening:** No need to check anything. If something important happened, Hermes already told me.

---

## Cost Breakdown

| Component | Monthly Cost |
|-----------|-------------|
| Hetzner VPS (CX22) | ~$6 |
| Z.AI Coding Plan | Varies (very affordable) |
| Telegram Bot | Free |
| Whisper STT (local) | Free |
| Edge TTS | Free |
| Tavily Web Search | Free tier |
| **Total** | **~$6-15/month** |

Compare that to:
- ChatGPT Plus: $20/month (browser only, no automation)
- Claude Pro: $20/month (browser only, no automation)
- Custom agent cloud: $50-200+/month

---

## Lessons Learned

**1. Match your endpoint to your plan.** Z.AI has two different API URLs — standard vs coding plan. Using the wrong one gives cryptic "Unknown Model" errors.

**2. Never set `api_key: ''` in config.yaml.** An empty string overrides environment variables and breaks everything silently.

**3. Gateway restarts need proper permissions.** If running as a systemd service, you need passwordless sudoers for the restart command.

**4. Local Whisper model names matter.** The config should use `base`, `tiny`, `small`, etc. — not OpenAI's `whisper-1` which is an API model name.

**5. Compression is your friend.** Long conversations eat up context windows. Hermes auto-compresses old messages while protecting recent ones.

**6. Start simple, then automate.** Get the basic chatbot working first. Add voice, then cron jobs, then memory. Don't try to set up everything at once.

---

## What's Next

I'm exploring:
- **Local LLM fallback** — Running Qwen 3.5 on the VPS as a backup when cloud APIs are down
- **Browser automation** — Using Browserbase for more complex web interactions
- **Multi-platform** — Adding Discord and Slack integrations
- **Automated blogging** — Having Hermes research, write, and publish articles

The beauty of Hermes is that it's extensible. Whatever you can imagine your AI assistant doing, there's probably a way to make it happen.

---

## Final Thoughts

Building an always-on AI assistant used to require a team of engineers, expensive cloud infrastructure, and months of development. Today, with tools like Hermes Agent, a single developer can set up a powerful, autonomous AI assistant in a few hours on a $6/month VPS.

The result isn't just a chatbot. It's a genuine assistant that understands context, remembers preferences, executes tasks, and proactively delivers information. It's the closest thing to having JARVIS in your pocket.

If you're interested in building something similar, the Hermes Agent GitHub repo has excellent documentation and an active community. The setup I've described here took me from zero to a fully functional AI assistant in one afternoon.

The future of personal AI isn't in browser tabs. It's in your messaging apps, running autonomously, always ready.

---

**Tech Stack Summary:**
- VPS: Hetzner CX22 (4 vCPU, 8 GB RAM)
- OS: Ubuntu 24.04 LTS
- AI Framework: Hermes Agent v0.5.0 by Nous Research
- LLM: GLM-5 Turbo via Z.AI Coding Plan
- Messaging: Telegram Bot API
- STT: faster-whisper (local, CPU)
- TTS: Microsoft Edge TTS (free)
- Search: Tavily API
- Scheduling: Hermes cron jobs
- Version Control: Git + GitHub

*If you found this useful, clap and follow for more articles on AI, automation, and building with open-source tools.*

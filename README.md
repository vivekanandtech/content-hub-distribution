# 🤖 AI Content Hub & Distribution — Make.com Automation

> A fully automated content pipeline that ingests RSS feeds, uses Mistral AI to analyze and rewrite content for 5 different platform audiences, then distributes simultaneously to Telegram, Discord, LinkedIn, Twitter/X, and Slack — with zero manual intervention.

---

## 📊 At a Glance

| Metric | Value |
|---|---|
| Platforms | 5 |
| AI Calls per Article | 6 |
| Make.com Modules | 22+ |
| Manual Steps | 0 |
| Cost per Article | ~$0.01 |

---

## 🏗️ Architecture Overview

The scenario follows a **sequential pipeline with intelligent branching**. Each article is processed once through a shared preprocessing stage, then simultaneously distributed to all 5 platforms with unique AI-rewritten content tailored per audience.

```
RSS Feed → Set Variables → Bitly URL → Mistral AI → JSON Parse → Store AI Data → Router
                                                                                      │
                    ┌─────────────────────────────────────────────────────────────────┤
                    │                │               │              │                 │
               Telegram          Discord         LinkedIn        Twitter           Slack
```

---

## 🔄 Pipeline Stages

### Stage 1 — Ingestion
- **RSS Feed module** polls configured feed(s) on a schedule
- Automatically deduplicates — never processes the same article twice
- Extracts: title, URL, description, publish date, category

### Stage 2 — Preprocessing
- **Set Variables** module normalises fields and prepares data for downstream modules
- **Bitly API** shortens the article URL for all platforms (enables click tracking)

### Stage 3 — AI Analysis (Master Call)
- **Mistral AI** (`mistral-small-latest`) performs a single master analysis call
- Outputs a structured JSON object with:
  - `summary` — concise article summary
  - `category` — content classification (e.g. Tech, Business, Marketing)
  - `sentiment` — Positive / Neutral / Negative
  - `topics` — key topic tags
  - `hashtags` — ready-to-use hashtag list

### Stage 4 — JSON Parsing & Storage
- **JSON Parse module** extracts fields from Mistral's nested response array
- `choices[1].message.content` indexing used to target the correct response block
- **Store AI Data** (Set Variables) persists parsed fields for all downstream routes

### Stage 5 — Router & Platform Distribution
The Router fans out to 5 parallel routes. Each route:
1. Makes a platform-specific Mistral AI rewrite call
2. Formats the content for that platform's conventions
3. Publishes via the platform's API or webhook
4. Logs the result to Google Sheets

---

## 📡 Platform Routes

### ✈️ Route 1 — Telegram
- **Filter:** All content (no restriction)
- **AI Tone:** Technical, informative
- **Format:** Summary + topics + hashtags
- **Delivery:** Telegram Bot API

### 🎮 Route 2 — Discord
- **Filter:** All content (no restriction)
- **AI Tone:** Casual, conversational with emojis
- **Format:** Summary + discussion hook
- **Delivery:** Discord Webhook

### 💼 Route 3 — LinkedIn
- **Filter:** `category == "Business"` OR `category == "Marketing"`
- **AI Tone:** Professional, insight-driven
- **Format:** Long-form post with business takeaways
- **Delivery:** Zapier Webhook → LinkedIn API

### 🐦 Route 4 — Twitter / X
- **Filter:** `sentiment == "Positive"` AND `category == "Tech"` (or Business)
- **AI Tone:** Punchy, engaging
- **Format:** 4-tweet numbered thread, split by custom delimiter
- **Delivery:** Zapier Webhook → Twitter/X API

### 💬 Route 5 — Slack
- **Filter:** `category == "Tech"` OR `category == "Business"`
- **AI Tone:** Structured briefing
- **Format:** Action items + key insights for team review
- **Delivery:** Slack API → `#content-review` channel

---

## 🧰 Tech Stack

| Tool | Role |
|---|---|
| **Make.com** | Core automation orchestrator (22+ modules) |
| **Mistral AI** (`mistral-small-latest`) | AI engine — master analysis + 5 platform rewrites |
| **Bitly API** | URL shortening & click analytics |
| **Telegram Bot API** | Direct messaging to Telegram channel |
| **Discord Webhooks** | Posting to Discord server |
| **LinkedIn API** | Professional post publishing (via Zapier) |
| **Twitter/X API** | Thread creation and posting (via Zapier) |
| **Slack API** | Team briefing in `#content-review` |
| **Zapier** | Bridge layer for LinkedIn + Twitter (webhook → API) |
| **Google Sheets** | Master logging dashboard (status, timestamps, AI data) |

---

## ⚙️ Configuration

### Required API Keys / Credentials

- `MISTRAL_API_KEY` — [console.mistral.ai](https://console.mistral.ai)
- `BITLY_API_TOKEN` — [app.bitly.com](https://app.bitly.com)
- `TELEGRAM_BOT_TOKEN` + `TELEGRAM_CHAT_ID`
- `DISCORD_WEBHOOK_URL`
- `LINKEDIN_ACCESS_TOKEN` (via Zapier OAuth)
- `TWITTER_API_KEY` + `TWITTER_API_SECRET` + `ACCESS_TOKEN` + `ACCESS_SECRET` (via Zapier)
- `SLACK_BOT_TOKEN` + `SLACK_CHANNEL_ID`
- Google Sheets: connected via Make.com Google module (OAuth)

### Router Filter Logic

```
Route 3 (LinkedIn):  category == "Business" OR category == "Marketing"
Route 4 (Twitter):   sentiment == "Positive" AND (category == "Tech" OR category == "Business")
Route 5 (Slack):     category == "Tech" OR category == "Business"
Routes 1 & 2:        No filter — all content passes
```

### Mistral AI Prompt Structure

**Master Analysis prompt outputs (JSON):**
```json
{
  "summary": "...",
  "category": "Tech | Business | Marketing | Other",
  "sentiment": "Positive | Neutral | Negative",
  "topics": ["topic1", "topic2"],
  "hashtags": ["#tag1", "#tag2"]
}
```

**Twitter Thread format (4 tweets separated by `|||`):**
```
1/ Hook tweet text here ||| 2/ Second tweet ||| 3/ Third tweet ||| 4/ CTA + link
```

---

## 🔧 Engineering Challenges & Solutions

### 1. Router Merge Problem
**Problem:** Make.com's Router cannot merge multiple routes back into a single downstream module.  
**Solution:** Redesigned the architecture so each route publishes independently to its own platform. No merge needed.

### 2. JSON Parsing from Mistral Response
**Problem:** Mistral returns content in a nested array with newline characters that break JSON parsing.  
**Solution:** Used Make.com's `replace()` function and correct array indexing: `choices[1].message.content`.

### 3. Special Characters Breaking JSON
**Problem:** AI-generated content contains quotes, newlines, and special characters that break JSON when sent as an HTTP body.  
**Solution:** Switched from raw JSON body to **form URL-encoded** format — escaping is handled automatically.

### 4. Zapier Data Parsing
**Problem:** Zapier receives form-encoded data as a single `Type` field string rather than parsed individual fields.  
**Solution:** Used Python's `urllib.parse.parse_qs()` inside a Zapier Code step to decode URL-encoded strings into accessible fields.

### 5. Twitter Thread Creation
**Problem:** Twitter/X native module was removed from Make.com.  
**Solution:** Zapier webhook approach — Mistral generates 4 numbered tweets separated by a `|||` delimiter, which Zapier splits and posts as a threaded conversation.

### 6. Hugging Face API Deprecation
**Problem:** The original `api-inference` Hugging Face endpoint was deprecated mid-build.  
**Solution:** Pivoted to Mistral AI — superior JSON formatting, better instruction-following, and a generous **1M tokens/month free tier**.

---

## 📈 Results

- ✅ **100% automated** — RSS to 5 platforms with zero manual steps
- ✅ **6 unique AI outputs** per article (1 master analysis + 5 platform rewrites)
- ✅ **~$0.01 cost** per article across all platforms
- ✅ **5 simultaneous posts** per article run
- ✅ **Infinitely scalable** — add new RSS feeds by cloning the source module
- ✅ **Full audit trail** — every run logged to Google Sheets with timestamps and status

---

## 📁 Repository Structure

```
/
├── README.md                  # This file
├── scenario/
│   └── make_scenario.json     # Exported Make.com scenario blueprint
├── zapier/
│   ├── linkedin_zap.md        # Zapier Zap setup for LinkedIn
│   └── twitter_zap.md         # Zapier Zap setup for Twitter/X thread
├── prompts/
│   ├── master_analysis.md     # Mistral master analysis prompt
│   ├── telegram_rewrite.md    # Platform-specific prompt: Telegram
│   ├── discord_rewrite.md     # Platform-specific prompt: Discord
│   ├── linkedin_rewrite.md    # Platform-specific prompt: LinkedIn
│   ├── twitter_thread.md      # Platform-specific prompt: Twitter thread
│   └── slack_briefing.md      # Platform-specific prompt: Slack
└── docs/
    └── architecture.png       # System architecture diagram
```

---

## 🚀 Getting Started

1. **Import the scenario** — go to Make.com → Scenarios → Import, upload `scenario/make_scenario.json`
2. **Connect credentials** — add your API keys for each module (see Configuration above)
3. **Set up Zapier Zaps** — follow the guides in `/zapier/` for LinkedIn and Twitter
4. **Configure your RSS feed URL** in the RSS module
5. **Set your Google Sheet ID** in the logging modules
6. **Run once manually** to verify the pipeline end-to-end
7. **Activate scheduling** — set your desired run interval (recommended: every 1–4 hours)

---

## 📬 Contact

Built as part of an automation portfolio project.  
Available for similar projects on **Upwork**.

---

*Built with Make.com · Mistral AI · Telegram · Discord · LinkedIn · Twitter/X · Slack · Zapier · Google Sheets*

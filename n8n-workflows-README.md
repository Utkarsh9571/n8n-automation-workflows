# n8n Automation Workflows — GKWebTech

Production automation workflows built and maintained for GKWebTech digital agency and associated businesses. All workflows are self-hosted on a Hostinger VPS at `n8n.gkwebtech.cloud`.

> **Note:** Workflow JSON files have credentials and API keys removed. Replace all `YOUR_*` placeholders with your own credentials before importing.

---

## Workflow Overview

| # | Workflow | Trigger | Status |
|---|----------|---------|--------|
| 1 | Multilingual SEO Blog Pipeline | Schedule (daily) | ✅ Active |
| 2 | Blog Approval & Publisher | Schedule (hourly check) | ✅ Active |
| 3 | LinkedIn Content Poster — GKWebTech | Telegram message | ✅ Active |
| 4 | LinkedIn Content Poster — Ayurvedastro | Telegram message | ✅ Active |
| 5 | Website AI Chatbot — GKWebTech | Webhook (website) | ✅ Active |
| 6 | Website AI Chatbot — Institute | Webhook (website) | ✅ Active |
| 7 | YouTube Video Pipeline | Telegram message | ⚠️ Paused (Gemini video limits) |
| 8 | WordPress Blog Poster — Ayurvedastro | Schedule | ✅ Active |

---

## Workflow Details

---

### 1. Multilingual SEO Blog Pipeline

**File:** `/blog-automation/seo-blog-pipeline.json`

**Trigger:** Daily schedule (10:00 AM)

**What it does:**
Fully automated SEO content generation pipeline that produces blogs in 3 languages without any manual input.

**Architecture:**
```
Schedule Trigger
    ↓
Get Topic from Google Sheets (Sheet 1 — Blog Queue)
    ↓
Get Keyword Pool from Google Sheets (Sheet 2 — Keywords)
    ↓
Convert Keywords Array (Code Node)
    ↓
AI Keyword Selection → picks 1 primary + 4 secondary keywords
    ↓
Fetch Existing Blogs from API (for internal linking)
    ↓
Generate English Blog (Gemini AI)
    ↓
Clean JSON Output (Code Node — strips markdown, extracts valid JSON)
    ↓
Translate to Dutch (Gemini AI)
    ↓
Clean JSON Output
    ↓
Translate to German (Gemini AI)
    ↓
Clean JSON Output
    ↓
Update Keyword Sheet (mark keywords as used)
    ↓
Fetch Image from Unsplash API
    ↓
Upload Image to Cloudinary
    ↓
Save Full Blog Draft to Google Sheets (status: waiting_for_approval)
```

**Blog Output Schema:**
```json
{
  "title": "...",
  "slug": "...",
  "excerpt": "...",
  "content": "...(HTML with H2/H3/lists/FAQ)...",
  "metaTitle": "...",
  "metaDescription": "...",
  "primaryKeyword": "...",
  "secondaryKeywords": ["...", "..."],
  "tags": ["...", "..."],
  "imagePrompt": "...",
  "language": "en"
}
```

**Key Technical Challenge — JSON Cleaning:**
AI models don't always output clean JSON. Added a Code node after every AI node:

```javascript
let text = $json.output;
text = text.replace(/```json/g, "").replace(/```/g, "");
const firstBrace = text.indexOf("{");
if (firstBrace !== -1) text = text.slice(firstBrace);
const lastBrace = text.lastIndexOf("}");
if (lastBrace !== -1) text = text.slice(0, lastBrace + 1);
let parsed = JSON.parse(text);
return [{ json: parsed }];
```

**APIs Used:** Gemini AI, Unsplash, Cloudinary, Google Sheets API, MongoDB (via REST API)

---

### 2. Blog Approval & Publisher

**File:** `/blog-automation/blog-publisher.json`

**Trigger:** Schedule (every hour)

**What it does:**
Checks Google Sheets for blog rows with status `approved`. When found, sends all 3 language versions (EN/NL/DE) to the MongoDB-backed website API. Updates row status to `published`.

**Architecture:**
```
Schedule Trigger
    ↓
Read Google Sheet — find rows where status = "approved"
    ↓
For each approved blog:
    POST English blog → https://api.gkwebtech.cloud/api/blogs
    POST Dutch blog → https://api.gkwebtech.cloud/api/blogs
    POST German blog → https://api.gkwebtech.cloud/api/blogs
    ↓
Update row status → "published"
```

**Why human-in-the-loop:**
Fully autonomous publishing was considered but rejected. Having a human approval step ensures content quality before it appears on the live website.

---

### 3. LinkedIn Content Poster — GKWebTech

**File:** `/social-media/linkedin-gkwebtech.json`

**Trigger:** Telegram message to bot

**What it does:**
Send a topic or keyword via Telegram. n8n generates an AI post using Gemini, selects a relevant image, and posts to the GKWebTech LinkedIn company page automatically.

**Architecture:**
```
Telegram Webhook Trigger
    ↓
Extract topic from message
    ↓
Generate LinkedIn post (Gemini AI)
    ↓
Select relevant image (Unsplash)
    ↓
POST to LinkedIn Pages API
    ↓
Send confirmation message back via Telegram
```

**Running since:** January 2026 — zero failures in 2+ months

---

### 4. LinkedIn Content Poster — Ayurvedastro

**File:** `/social-media/linkedin-ayurvedastro.json`

**Trigger:** Telegram message to bot

Same architecture as Workflow 3, adapted for the Ayurvedastro brand. Different Telegram bot, different LinkedIn company page, different content tone prompt.

---

### 5. Website AI Chatbot — GKWebTech

**File:** `/chatbots/chatbot-gkwebtech.json`

**Trigger:** Webhook (embedded on gkwebtech.cloud)

**What it does:**
AI chatbot embedded on the agency website. Answers visitor questions about GKWebTech services, pricing, and contact information. Uses website content as context. Falls back gracefully for questions outside scope.

**Architecture:**
```
Webhook Trigger (visitor sends message)
    ↓
AI Agent (Gemini) with system prompt containing website context
    ↓
Generate response
    ↓
Return JSON response to website
```

---

### 6. Website AI Chatbot — Institute

**File:** `/chatbots/chatbot-institute.json`

**Trigger:** Webhook (embedded on institute.gkwebtech.cloud)

Same architecture as Workflow 5, with institute-specific context — course information, fees, enrollment process, contact details.

---

### 7. YouTube Video Pipeline

**File:** `/video/youtube-pipeline.json`

**Trigger:** Telegram message

**Status:** ⚠️ Paused

**What it does:**
Intended to generate short educational videos using Gemini video generation, determine content category (quick tip / promotional / educational), and post to YouTube with title, description, and hashtags.

**Why paused:**
Gemini's video generation API only produces 8-second clips, which is insufficient for useful content. Other video generation APIs with longer output were outside budget. Will revisit when better free/affordable options are available.

---

### 8. WordPress Blog Poster — Ayurvedastro

**File:** `/blog-automation/wordpress-ayurvedastro.json`

**Trigger:** Schedule (daily)

**What it does:**
Generates SEO blog content for the Ayurvedastro WordPress website. Similar to the main blog pipeline but adapted for WordPress — posts directly to the WordPress REST API instead of MongoDB. Content is ayurveda and astrology focused.

---

## Repository Structure

```
/blog-automation/
    seo-blog-pipeline.json          # Main multilingual blog generation workflow
    blog-publisher.json             # Approval check and publish workflow
    wordpress-ayurvedastro.json     # WordPress blog poster

/social-media/
    linkedin-gkwebtech.json         # LinkedIn poster for GKWebTech
    linkedin-ayurvedastro.json      # LinkedIn poster for Ayurvedastro

/chatbots/
    chatbot-gkwebtech.json          # Agency website chatbot
    chatbot-institute.json          # Institute website chatbot

/video/
    youtube-pipeline.json           # YouTube video pipeline (paused)

/docs/
    setup-guide.md                  # How to import and configure these workflows
    credentials-required.md         # List of all API credentials needed
    json-cleaning-pattern.md        # The JSON cleaning code node explained
```

---

## How to Import a Workflow

1. Open your n8n instance
2. Go to Workflows → Import from File
3. Select the `.json` file
4. Update all credential placeholders — search for `YOUR_` in the workflow nodes
5. Activate the workflow

---

## Credentials Required

To run these workflows you will need:

| Service | Used In |
|---------|---------|
| Gemini AI API Key | Blog pipeline, chatbots, LinkedIn poster |
| Google Sheets OAuth | Blog pipeline, publisher |
| Cloudinary API Key + Secret + Cloud Name | Blog pipeline |
| Unsplash Access Key | Blog pipeline, LinkedIn poster |
| Telegram Bot Token | LinkedIn posters, YouTube pipeline |
| LinkedIn OAuth (Pages API) | LinkedIn posters |
| MongoDB REST API endpoint | Blog publisher |
| WordPress REST API credentials | Ayurvedastro blog poster |

---

## Key Lessons from Building These

**LLMs are unreliable for strict JSON output.** Always add a Code node after every AI node to clean the output before parsing. Without this, workflows break randomly on malformed responses.

**Don't depend on free API rate limits for production.** Unsplash has a 50 requests/hour limit on free tier. Gemini has generous limits via Jio offer but still has daily quotas. Always have a fallback plan.

**Human-in-the-loop is more trustworthy than full automation** for content going on production websites. The Google Sheets approval gate adds one manual step but prevents publishing bad AI output.

**Automation must be idempotent.** If a workflow runs twice for the same topic, it should not create duplicate content. The blog publisher checks for existing slug + language before inserting.

---

*Built and maintained by [Utkarsh Sharma](https://github.com/Utkarsh9571)*  
*Self-hosted n8n at GKWebTech — Alwar, India*

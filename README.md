# B2B Outreach Engine (n8n)

Four n8n workflows that run a PR / media outreach campaign end to end: find targets, enrich them
with contact details, draft a short personalised pitch with an LLM, send in throttled batches, and
follow up. A Notion database is the CRM and holds the state of every lead, so no contact is
pitched twice. Built for a client's podcast and media outreach.

```
1  Sourcing          SerpAPI → clean names and domains → contact lookup → Notion
2  Drafting          Notion status "Approved" → Firecrawl + SerpAPI news → LLM agent → draft back to Notion
3A Send queue        daily 09:00 → approved leads → one at a time with a delay → Gmail → Notion ("Sent" + thread ID)
3B Reply/follow-up   daily 08:00 → Gmail thread check → replied: Notion + Telegram alert / no reply after 4 days: follow-up
```

| File | What it does |
|---|---|
| `workflow-1-sourcing-enrichment.json` | Manual trigger → SerpAPI publication data → split into rows → Code node normalises names and domains (including author names taken from URL slugs) → contact lookup → map to the Notion schema → log leads. |
| `workflow-2-pitch-drafting.json` | Notion trigger on status "Approved" → Firecrawl page scrape and SerpAPI news in parallel → merge → an AI agent (Gemini) writes a pitch under hard rules: under 120 words, first person, no fluff, a fixed four-part structure (hook, bridge, value, soft call to action) → writes the draft back to Notion. |
| `workflow-3a-send-queue.json` | Daily cron → approved leads from Notion → queue check → loop with a Wait node between emails (anti-blocking) → **sandbox switch**: if a lead has `sandbox_mode` on, the email goes to a QC address instead of the target → Gmail → writes the Gmail thread ID and "Sent" back to Notion. |
| `workflow-3b-reply-followup.json` | Daily 08:00 → sent leads from Notion → fetch the Gmail thread → more than one message means a reply: mark "Replied" and send a Telegram alert; otherwise, if the last message is older than 4 days, reply in the thread with a short follow-up and mark "Followed-up". |

## Things to know before you run it

- **The contact lookup in workflow 1 points at `http://localhost:3005/v1/people/match`.** In this
  export that is a local stand-in. For real use, point the node at the Apollo people-match API
  and add your key.
- **Client details removed.** The pitch prompt in workflow 2 originally named the client and
  described his background. They are now `[CLIENT NAME]` and `[CLIENT BACKGROUND]` placeholders.
  Fill in the sender you are writing for.
- **Notion property names** (for example `property_nama_jurnalis`) follow the original database.
  Rebuild a database with the same properties or edit the nodes.
- Credentials (Notion, SerpAPI, Firecrawl, Gemini, Gmail, Telegram) are `REPLACE_ME` in the export.
  The Telegram chat ID is `YOUR_CHAT_ID`.
- Send in sandbox mode first. Cold email carries legal and deliverability duties (consent rules,
  unsubscribe handling, sending limits); check those for your market.

This is a workflow export, not a measured benchmark: there is no results file.

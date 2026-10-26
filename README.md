# AI Lead Prospect Finder

Finds local trade businesses, audits their websites, and uses AI to score each one as a sales lead, then logs everything to Google Sheets and emails a top-5 summary. Built in n8n for Synclo, an automation agency for Australian trade businesses.

![Workflow](workflow.png)

## How it works

**1. Find new prospects**
- Pulls the next `pending` trade + suburb (e.g. "plumber in Cranbourne") from a Search Queue sheet.
- Searches the **official Google Places API** (Text Search, no scraping).
- Marks the search `done` straight away, so the queue always moves forward.
- Removes duplicates by checking each `place_id` against the Leads sheet.

**2. Audit their websites**
- Branches on whether the business has a website.
- Fetches each site and checks for online booking, live chat, contact forms, mobile-ready layout and 24/7 / emergency claims.
- Flags businesses with no website, which are often the strongest prospects.

**3. Score, save and report**
- An LLM scores each lead 1–10 with a **structured output schema**: score, 3 specific gaps, recommended offer, a personalised SMS opener and a confidence level.
- Appends every lead to the Leads sheet with status `needs_review`.
- Emails a top-5 summary ranked by score.

## Design decisions

- **Human in the loop:** nothing is sent to prospects automatically. Every lead lands as `needs_review`, so a person approves outreach. That keeps it compliant with Australia's Spam Act.
- **Structured AI output:** a JSON schema parser forces consistent, machine-readable scores instead of free text.
- **Confidence scoring:** the model flags low-data leads (unreachable site, no reviews) instead of guessing.
- **Fault tolerance:** website fetches time out after 10s and continue on error, so one bad site never stops a run.

## Tools

n8n · Google Places API (New) · REST APIs · JavaScript (Code nodes) · OpenAI (LLM chain + structured output parser) · Google Sheets · Gmail · Cron scheduling

## What I debugged

- **Stuck search queue:** when a search returned no new leads, the workflow stopped before marking it done, so the 7am schedule repeated the same search every day. Fixed by marking the search done straight after the Places call and letting the split step output empty results.
- **Timezone:** the schedule ran on UTC (6pm Melbourne) instead of 7am local. Fixed in the workflow settings.
- **Silent failures:** added a separate error-alert workflow that emails the failed node and error message on any scheduled run.

## Setup

1. Import `workflow.json` into n8n.
2. Create credentials: Google Places API key, Google Sheets, Gmail and an OpenAI key.
3. Create two sheets:
   - **Search Queue:** `trade`, `suburb`, `status`
   - **Leads:** `place_id`, `business`, `trade`, `suburb`, `address`, `phone`, `website`, `rating`, `reviews`, `google_maps_url`, `no_website`, `has_booking`, `has_chat`, `has_form`, `mobile_ready`, `mentions_24_7`, `score`, `gaps`, `recommended_offer`, `opener`, `confidence`, `status`, `date_added`
4. Replace `YOUR_SEARCH_QUEUE_SHEET_ID`, `YOUR_LEADS_SHEET_ID` and `YOUR_EMAIL@example.com`.
5. Optionally point the workflow's error workflow at your own alert workflow.

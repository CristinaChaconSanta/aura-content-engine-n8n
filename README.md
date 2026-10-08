# Aura Content Engine (n8n)

Exported n8n workflows that turn a Sunday research pass into a week of social posts for Aura Studio. Credentials stay in n8n. They are not in these JSON files.

## Problem

A content week was a scramble: scan Reddit, influencers, and news by hand, then repeat last month's angles because nothing remembered what had already shipped.

## What it does

The current research workflow (`workflow-content-engine.json`, 23 nodes) runs Sunday at 8pm:

1. Loads brand config (name, industry, queries, voice, platforms).
2. Pulls Reddit and influencer pages through Apify, and industry news through NewsAPI.
3. Merges those sources.
4. Loads the context graph first (what already ran, what worked, editorial rules).
5. Loads the catalog second (RAG or a local catalog).
6. Asks a model (GPT-4o in the setup guide) for the week's plan.
7. Runs guardrails. If nothing passes, it alerts and does not generate images.
8. Generates carousel images, assembles the package, writes the context graph, and notifies.

Later exports are the production line that grew out of that research pass:

| File | Workflow | Nodes |
|---|---|---|
| `workflow-content-engine.json` | Trend research and production | 23 |
| `aura-studio-v2.json` | Aura Studio content engine | 47 |
| `aura-studio-v3.json` | Template rendering pipeline | 50 |
| `aura-studio-v4.json` | Autonomous writing | 47 |
| `aura-studio-complete.json` | Full engine | 59 |
| `test-conexiones.json` | Connection smoke test | 3 |
| `context-graph-template.json` | Empty memory document for the graph | — |

Show `aura-studio-v4.json` and `workflow-content-engine.json`. The earlier files are the iteration history.

## Stack

- n8n
- Apify, NewsAPI, OpenAI (text and `gpt-image-1`)
- Optional Gemini image node, left as an alternate path
- Context graph in Supabase, Google Sheets, or a local JSON file
- Optional vector store (Supabase, Pinecone, or Qdrant) in place of the hardcoded catalog

## Architecture

1. Import the JSON in n8n (Workflows → Import from File).
2. Create credentials in n8n and attach them to the named nodes. The export points at credential names such as "Apify API Token". It does not embed the token.
3. Edit the brand-config node.
4. Point "Load context graph" and "Update context graph" at Supabase, Sheets, or the template file.
5. Leave image generation on OpenAI, or rewire the alternate Gemini node.
6. LinkedIn publish is documented as an optional HTTP node. The setup guide says not to connect Meta: accounts using AI automation were getting blocked.

## Error handling

Guardrails in the setup guide, applied before images:

1. Minimum confidence **40/100**.
2. Linked sources must be present.
3. The package structure is required.
4. Carousels must have **3–10** slides.
5. Caption length is capped.
6. A banned-word filter runs.
7. If every candidate fails, the workflow alerts and skips image generation.

Context graph is read before RAG on purpose. Reversing that order returns raw catalog chunks with no memory of last week, and the model repeats itself.

## Results

This repo does not store post-level performance. The setup guide's image cost estimate is:

- about **USD 0.04–0.08 per image** on `gpt-image-1`
- about **USD 1.40–2.80 per week** for 7 posts with about 5 slides each

Gemini is the zero-cost alternate, with lower visual control. No published impression or lead numbers live in these files.

## Run it

1. Run n8n (Cloud or `npx n8n`).
2. Import `workflow-content-engine.json` or `aura-studio-v4.json`.
3. Add Apify, NewsAPI, and OpenAI credentials in n8n. Do not paste them into the JSON.
4. Execute the workflow manually once. Then enable the Sunday 8pm trigger.
5. Use `test-conexiones.json` (3 nodes) when a credential fails and you need to see which connection broke.

Spanish setup notes with the same steps: `GUIA-SETUP.md`, `SETUP-AURA-STUDIO.md`, `SETUP-v3.md`.

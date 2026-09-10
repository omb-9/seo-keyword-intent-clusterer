# SEO Keyword Intent Clusterer (Telegram + OpenRouter)

An n8n workflow that turns a raw keyword export into a content-plan-ready spreadsheet. Upload a CSV to a Telegram bot, and get back an `.xlsx` where every keyword is classified by search intent, funnel stage, topic cluster, recommended page format, and priority, grouped so the highest value clusters sit at the top of the file.

## Why this exists

Keyword exports from Ahrefs, Semrush, or Search Console give you volume and difficulty but no answer to the question that actually drives a content plan: what does this person want, and what page should we build for them? This workflow answers that at scale, and it solves the problem that makes most batched LLM classification unusable, which is that each batch invents its own cluster names.

The fix is a two-pass design. Pass one classifies keywords in parallel batches. Pass two takes only the distinct cluster labels that came back, merges near duplicates into a canonical set, and assigns each one a pillar page title. The result is a coherent cluster map instead of forty variations on the same topic.

## Features

- Accepts CSV or TXT, with comma, semicolon, tab, or pipe delimiters, sniffed automatically
- Detects the header row and finds the keyword column by name, falling back to the first column
- Carries any `volume`, `difficulty`, or `cpc` columns straight through into the output
- Deduplicates case-insensitively and drops blank or numeric-only rows
- Classifies intent into the four standard categories, plus funnel stage, content format, priority, and branded/local flags
- Two-pass cluster consolidation with pillar page titles
- Output sheet is pre-sorted by cluster value, so it is readable top to bottom with no sorting in Excel
- Returns both the spreadsheet and a Telegram summary of the intent mix and where to start
- Partial failure tolerance: a failed batch marks those keywords `unclassified` rather than breaking the run

## Output columns

`Keyword` · `Cluster` · `Pillar Page` · `Intent` · `Funnel Stage` · `Content Format` · `Priority` · `Branded` · `Local` · `Search Volume` · `Difficulty` · `CPC` · `Notes`

Intent uses the four standard categories: informational, navigational, commercial, and transactional. Content format is constrained to a fixed enum of fifteen real page types (comparison page, pricing page, location page, glossary, and so on) so the output stays consistent across runs.

## Setup

1. Import `seo-keyword-intent-clusterer.n8n.json` into n8n.
2. Create a Telegram credential from your BotFather token and attach it to every Telegram node.
3. Create an HTTP Header Auth credential named `OpenRouter API Key`:
   - Header name: `Authorization`
   - Header value: `Bearer sk-or-your-key-here`
4. Attach that credential to both OpenRouter nodes.
5. Open the Config node and set your models and limits.
6. Activate the workflow, then send the bot `/help` to confirm it responds.

### Config values

| Field | Default | Notes |
|---|---|---|
| `model` | `anthropic/claude-sonnet-4.5` | Primary model |
| `fallbackModel` | `openai/gpt-4.1-mini` | Used automatically if the primary is unavailable |
| `temperature` | `0.1` | Low on purpose. Classification should be repeatable, not creative |
| `batchSize` | `40` | Keywords per API call. Lower it if you hit token or rate limits |
| `maxKeywords` | `1000` | Hard cap per upload, to protect against a 50k row export |

## How it works

The Telegram Trigger receives the message and Route Input splits it three ways: `/help` and `/start` get the format guide, a message carrying a document goes to the processing path, and anything else gets a short prompt to send a file.

Download Keyword File pulls the binary from Telegram, and Parse Keyword File does the real work: it sniffs the delimiter, runs an RFC4180-style parser that handles quoted fields and embedded newlines, detects the header, locates the keyword column and any metric columns, dedupes, applies the cap, and splits the list into batches with a complete OpenRouter payload attached to each one. Everything the model needs, including the JSON schema, is built here rather than spread across node parameters.

A progress message fires once so the sender is not left waiting blind. Classify Intent then runs one call per batch, rate limited to one request at a time with a short interval between them, retrying up to three times.

Collect Classifications merges the responses back onto the original keyword list **by matching the keyword string, not the item index**. This matters: if a batch fails and drops out of the success branch, index-based merging would silently shift every subsequent row and produce a plausible-looking but completely wrong spreadsheet. String matching means a failed batch produces exactly three or four visibly `unclassified` rows instead.

That node also builds the pass two payload, sending only the distinct cluster labels with a few example keywords each. Consolidate Clusters merges them and returns pillar page titles. This node is configured to continue on error, so if pass two fails the workflow still delivers using the raw labels.

Build Rows applies the mapping, sorts by cluster value, and emits one flat row per keyword. Build Spreadsheet converts those items to `.xlsx`, Send Spreadsheet delivers it with a caption, and Build Summary recomputes the headline stats from the finished rows for the follow-up Telegram message.

## Failure behaviour

| Failure | What happens |
|---|---|
| Unreadable or empty file | Specific error message naming the reason |
| No usable keyword column | Error message pointing at the format guide |
| One batch fails after retries | User is notified once, those keywords come back `unclassified`, spreadsheet still delivered |
| All batches fail | Failure notice sent, no partial file |
| Pass two consolidation fails | Raw cluster labels are kept, pillar titles are blank, run completes normally |
| Malformed JSON from the model | Parser tolerates fenced or partial JSON and falls back cleanly |

## Customization ideas

- Swap the Telegram Trigger for a Google Drive trigger so dropping a file in a folder kicks off the run.
- Add a Google Sheets node after Build Rows to write results into a shared planning sheet instead of, or alongside, the `.xlsx`.
- Extend the pass one schema with a `serp_feature` field if you want featured snippet and People Also Ask opportunities flagged per keyword.
- Add a third pass that drafts outlines for the top N pillar pages.

## Limitations

- Intent classification is a judgment call, and edge cases (ambiguous head terms especially) will need human review. Treat the output as a strong first pass, not a final plan.
- The model does not see live SERPs, so it infers intent from the query alone. A keyword whose SERP has shifted commercially may still be classified informational.
- Clustering quality degrades on very large lists, since pass two only sees labels rather than every keyword.
- Telegram caps uploads at 20MB for bots, which is far above any realistic keyword CSV, but worth knowing.

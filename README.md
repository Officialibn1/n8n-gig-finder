# n8n Gig Finder

An n8n workflow that finds freelance gigs and jobs related to **n8n and workflow automation** from several sources, filters and scores them, and sends the best ones to you as an alert with a link.

An LLM **extracts facts** from each posting (budget, urgency, red flags). A Code node **computes the score** from weights you control in the Config node. The LLM never produces the score.

> **Status**
> - Built and tested on n8n **2.40.5**, self-hosted (Docker). Both workflows were also imported and run in a clean instance.
> - **Two workflow files ship.** `gig-finder.json` is the slim version (OpenAI-compatible LLM, Telegram alerts): import this one. `gig-finder-full.json` adds Slack alerts and an Anthropic branch, with the provider and channel chosen in Config. Sections marked *(full only)* apply to the full file.
> - Tested: the n8n forum, Hacker News and Remotive sources; OpenAI (`gpt-4o`, default) and Upstage Solar models through the OpenAI-compatible branch; Telegram and Slack delivery; Discord error notices.
> - Built but **not tested**: the Anthropic branch (full only; no API credit was available).
> - **Not tested on n8n Cloud.** The workflow uses only core nodes and no external modules in Code nodes, so it should be portable, but Data Table availability and Code node limits on Cloud plans are unverified.
> - **Not built**: WhatsApp delivery (see [WhatsApp](#whatsapp-documented-not-built)) and Ollama.

## Files

| File | What it is |
|---|---|
| `gig-finder.json` | Slim version: OpenAI-compatible LLM, Telegram alerts. Import this one. |
| `gig-finder-full.json` | Adds Slack alerts and an Anthropic branch; provider and channel are chosen in Config. |
| `gig-finder-error.json` | The error workflow. Import it with either version. |

## How it works

1. **Triggers.** `Trigger: Schedule` (default every 6 hours). To test on demand, hover the trigger and click Execute Workflow, or press Ctrl+Enter.
2. **Config.** One node holds the settings except the schedule. In the slim version the LLM model and the Telegram destination are set on their own nodes; in the full version they are in Config too.
3. **Sources.** n8n community forum (Jobs category), Hacker News ("Ask HN: Freelancer? Seeking freelancer?" threads via Algolia), and Remotive. Each is normalized to one schema: `title`, `url`, `source`, `posted_at`, `description`, `budget_text`, `external_id`, `meta`.
4. **Pre-filter.** A cheap keyword filter runs before any LLM call. Supply-side titles ("seeking work", "for hire") are dropped. The n8n forum bypasses the include check because its category is already n8n context.
5. **Age filter and dedupe.** Gigs older than `filters.max_age_days` are dropped. The dedupe key is SHA-256 of `source:external_id`, checked against a Data Table.
6. **Forum enrichment.** Forum topics arrive without a body, so the first post is fetched for new forum gigs only.
7. **LLM extraction.** The posting is wrapped in delimiters and sent to the configured model. The output must match a JSON schema and is validated again in code.
8. **Scoring.** A Code node turns the extracted facts into a 0 to 100 score (below).
9. **Alert.** Gigs at or above `scoring.min_score` are sent to Telegram (or Slack in the full version), best first. Every evaluated gig is recorded in the Data Table with its outcome.
10. **Errors.** Failures notify a Discord channel (below).

## Setup

1. **Create a Data Table** named `seen_gigs` with text columns `dedupe_key`, `url`, `source`, `title`, `outcome` and a number column `score`. Select it in all three Data Table nodes: `Gigs: Keep Unseen Only`, `Seen: Record Alerted Gig`, `Seen: Record Non-Alerted Gig`.
2. **Create credentials.** Credentials are not part of an import, so after importing, every node that needs one shows a prompt to connect it. Slim version: an OpenAI (or OpenAI-compatible) API credential for `LLM (OpenAI-compatible): Chat Model`, a Telegram Bot credential for `Telegram: Send Alert`, and a Discord webhook credential for `Discord: Send Failure Notice` and the error workflow's Discord node. *(Full only)* additionally a Slack credential for `Slack: Send Alert` and, only if you use that provider, an Anthropic credential for `LLM (Anthropic): Chat Model`.
3. **Set your values.** Slim version: choose your model in the `LLM (OpenAI-compatible): Chat Model` dropdown, replace `REPLACE_ME` with your Telegram chat ID in `Telegram: Send Alert`, and review `Config` for keywords and scoring. *(Full only)* Edit `Config` instead: every `REPLACE_ME`, plus the `llm` and `notify` sections.
4. **Import and publish the error workflow** (it cannot be selected until it is published), then open the main workflow's Settings and select it under *Error Workflow*. Importing does **not** link it, whatever order you import in (tested on 2.40.5), so this step is required. Without it, unhandled failures will not notify you.
5. **First run:** set `filters.max_age_days` to `3` so an empty table doesn't send a burst of old gigs, then raise it.
6. **Telegram node:** operation must be *Send Message* with Parse Mode *HTML*.
7. **Publish.** In the slim version, publish once every credential is connected. *(Full only)* n8n will not let you publish a workflow while any node is missing a credential, so delete the branches you don't use first. Delete the whole unused branch rather than disabling it: a disabled node is expected to pass its input through unchanged (not tested here), which could record gigs as alerted or extracted when nothing was sent or extracted.
   - Telegram only: delete `Slack: Format Message`, `Slack: Send Alert` and the `slack` rule in `Alert: Route by Channel`. For Slack only, do the reverse and set `notify.channel` to `slack`.
   - OpenAI only: delete `LLM (Anthropic): Extract Gig Facts`, `LLM (Anthropic): Chat Model`, `LLM (Anthropic): Output Parser` and the `anthropic` rule in `LLM: Route by Provider`.
   - After deleting a rule, check that the fallback output is still connected to the "Unknown" Stop and Error node. A mismatched `provider` or `channel` value should stop the run with an error, not drop items.

The schedule is not in Config because a trigger runs before Config exists. Remotive's API asks for at most about 4 calls a day, so keep the schedule at 6 hours or slower.

## Config reference

The `Config` node is the source of truth. Defaults (the `llm` and `notify` rows exist in the full version only; the slim version sets those on the nodes):

| Section | Key | Default | Meaning |
|---|---|---|---|
| `forum` | `base_url`, `category_path` | community.n8n.io, `/c/jobs/13.json` | Jobs category endpoint |
| `hn` | `thread_title_prefix` | `Ask HN: Freelancer? Seeking freelancer?` | Which threads to accept |
| `hn` | `max_thread_age_days` | 45 | Fail loudly if the newest thread is older |
| `remotive` | `search_term`, `limit` | `n8n`, 50 | Server-side search (saves API quota) |
| `filters` | `include_keywords` | `n8n`, `workflow automation` | Relevance gate |
| `filters` | `exclude_title_patterns` | seeking work, for hire, [available], open to work | Drop supply-side posts |
| `filters` | `include_bypass_sources` | `n8n-forum` | Skip the include check |
| `filters` | `max_age_days` | 14 | Ignore older gigs |
| `llm` *(full only)* | `provider`, `model` | `openai_compatible`, `gpt-4o` | `openai_compatible` or `anthropic` |
| `llm` *(full only)* | `temperature` | `0` | Set to `null` for models that reject it |
| `notify` *(full only)* | `channel` | `telegram` | `telegram` or `slack` |
| `notify` *(full only)* | `telegram.chat_id`, `slack.channel_id` | `REPLACE_ME` | Your destination |
| `scoring` | `weights`, `min_score`, tables | see below | All scoring behavior |

*(Full only)* An unknown `llm.provider` or `notify.channel` stops the run with an error instead of passing data through.

## Scoring

```
base    = 100 x sum(part x weight) / sum(weights)
score   = clamp(base - penalties, 0, 100)
alerts  = score >= min_score
```

| Component | Default weight | How the part (0 to 1) is computed |
|---|---|---|
| relevance | 30 | `core` 1, `secondary` 0.6, `mention_only` 0.25, `none` 0, plus 0.2 if the keyword is in the title |
| budget | 25 | See below |
| recency | 15 | Linear decay over 168 hours |
| urgency | 10 | Sum of signal values (asap, immediate start, deadline, urgent wording) |
| source trust | 10 | forum 1, Hacker News 0.7, Remotive 0.6 |
| engagement | 10 | one-off project 1, ongoing contract 0.9, full-time 0.4, unknown 0.5 |

Weights are relative, so you can edit one without keeping the total at 100. `min_score` defaults to 40.

**Gates.** If `poster_type` is not in `allowed_poster_types` (default `hirer`), the score is 0. A red flag listed in `red_flag_block` (default `upfront_fee`, `off_platform_payment`) also forces 0. Blocked gigs are still recorded with their outcome for auditing.

**Penalties.** `upfront_fee` 40, `off_platform_payment` 30, `equity_or_exposure_only` 25, `vague_scope` 15, `unpaid_trial_work` 10; suspected injection text 20; implausible budget 15.

**Budget.** An unknown budget scores 0.15. A stated budget scores `0.3 + 0.7 x min(1, amount / 3000)`, so stating a budget is never worse than hiding it, and a bigger one is never worse than a smaller one. A config guard throws if the floor is set below the unknown score. Budgets are converted to USD per comparable period; any currency other than USD is treated as unknown (no exchange-rate data is shipped, because hard-coded rates go stale).

## LLM extraction

Extracted fields: `poster_type` (hirer, worker_seeking_work, partnership_or_promo, other), `engagement`, `n8n_relevance` (core, secondary, mention_only, none), `budget_min`, `budget_max`, `currency`, `budget_period`, `urgency_signals`, `red_flags`, `summary`. Allowed values are fixed lists so the score is computable and the output can be validated.

- **Providers:** the OpenAI-compatible branch works with OpenAI, and with Upstage by setting the credential's base URL. The slim version has only this branch; pick the model in the chat model node. The full version also has an Anthropic branch, which is a copy and is untested. n8n has no node to pick a chat model by expression, so each provider is its own branch. When you edit the prompt, system message or schema, apply the same edit to every branch.
- **Temperature:** the slim version does not set a temperature on the chat model, because some models reject any temperature value, and a note on the chain explains how to fix that error. The `gpt-4o` benchmark runs used temperature 0, so the shipped sampling was not benchmarked. Set 0 in the node's options if your model accepts it. *(Full only)* `llm.temperature`, set to `null` for models that reject it.
- **Validation:** a Code node re-checks every enum and type, because a schema that is missing a `required` entry lets fields be silently skipped (this happened during development).
- **Missing data is `null`, never a guess.**
- **Injection handling:** job text is untrusted. It is delimited, and the model is told to treat it as data. This is only a partial defense; see limitations.

## Reliability and error handling

| Failure | Behavior |
|---|---|
| A source is down or returns nothing unexpected | Discord notice naming the source; the other sources continue |
| HN thread not found or stale | Fails loudly with a clear message |
| LLM call fails | One grouped Discord notice; the gig is **not** recorded, so it retries next run |
| LLM output fails validation | Notice; recorded as `extraction_invalid` so it does not re-run (delete the row to retry) |
| Alert delivery fails | Notice; the gig is **not** recorded as seen, so it retries next run |
| Anything unhandled | The separate error workflow posts to Discord |

Handled failures end the execution as a success, so the error workflow fires only for what the workflow did not anticipate.

## Testing and sample data

`Gigs: Keep Unseen Only` is pinned with five synthetic `[SAMPLE]` gigs (all URLs on `example.com`): a good gig, a mention-only role, a freelancer advertising themselves, a scam-looking post, and a post containing injected instructions. Manual runs still call the live sources, then process the samples, and each run re-sends sample alerts and writes sample rows to `seen_gigs`. Delete those rows afterwards. Unpin the node to test with live data. Pinned data is used only in manual executions (per n8n's documentation), so a published workflow always processes live data.

Demo results with the shipped default model (`gpt-4o`), run in a clean instance with `min_score` 40:

| Sample | Expected | Observed |
|---|---|---|
| 1. n8n lead routing, $800 | alerted | 71, alerted |
| 2. Operations analyst, n8n nice to have | alerted or below threshold | 54, alerted |
| 3. Freelancer for hire | filtered_poster_type, score 0 | 0, filtered_poster_type |
| 4. Deposit and crypto payment | blocked_red_flag, score 0 | 0, blocked_red_flag |
| 5. Injected instructions | alerted with a warning | 41, alerted |

Scores vary by model and drift down over time because the sample dates are fixed.

## Data, privacy and terms

- The Data Table stores `dedupe_key`, url, source, title, outcome and score. It never stores posting text, because postings can contain personal details.
- **Remotive** asks for attribution and a link back, allows about 4 requests a day, delays jobs by 24 hours, and asks that its jobs not be redistributed. Alerts for Remotive jobs say "via Remotive". This repository ships no Remotive data.
- The n8n forum is polled lightly: one list request per run, plus one request per new forum gig, spaced out. The forum's own terms were not reviewed in detail; check them before running this at a high frequency.
- Reddit was evaluated and rejected: its API now requires per-app manual approval, which users of a shared template cannot be assumed to have, and the alternatives violate its terms.

## Known limitations

- **Low yield.** Few real n8n gigs are posted each month. Most Hacker News postings are people seeking work, and Remotive's search returns jobs that mention n8n only in passing.
- **Cross-posted gigs** appear under two sources and can alert twice.
- **Overlapping runs** (a scheduled run during a manual one) can alert the same gig twice, because dedupe checks at the start and writes at the end.
- **Models disagree on subtle labels.** `solar-mini4` rated a role that mentions n8n as optional as `secondary`; `gpt-4o` rated it `mention_only`. Prompt and schema were tuned on a small benchmark of about ten items, so accuracy on new data is unmeasured.
- **Prompt injection is only partly handled.** In testing, `gpt-4o` resisted a budget manipulation but still returned the injected relevance value. The code limits the damage with a budget cap, a plausibility check and a penalty, and the alert says the posting contains instructions aimed at AI readers. Detection is a short regex list, so it misses paraphrases and flags legitimate screening text. Flagged gigs are penalized, not blocked.
- **Allowed values live in two places** (the output parser schema and the validator) and can drift.
- **Known n8n 2.40.5 quirk:** setting On Error to "continue using error output" on the Basic LLM Chain node loses paired-item data even for successful items. The workflow uses regular continue plus an IF node on the `error` field instead.
- **Currency and period handling are crude.** Only USD is converted, and per-project and monthly budgets share a factor of 1.
- **Slim version settings.** The LLM model and Telegram destination are set on their nodes, so "all settings in one Config node" holds only for the full version.
- **Temperature is unset in the slim version**, while the benchmark ran at 0.
- **No notification throttle.** A source that stays down sends a notice on every run.
- **Discord is a single point of failure** for notices.
- **Postings are sent to the LLM in full.** There is no length cap, so very long postings cost more and give injected text more room.
- **Dedupe key** is not a content hash, so a re-posted gig with a new ID alerts again.
- **Telegram rate limiting** is handled only by retries.
- The Data Table is created by hand, and its ID is instance-specific.

## WhatsApp (documented, not built)

WhatsApp Business Cloud API is not included. From vendor documentation (verify against Meta's current docs before relying on it):

- Only an approved **message template** can start a conversation. Free-form text is allowed only inside the 24-hour window that opens when the recipient messages the business first.
- A gig alert is business-initiated, so it would need an approved utility template (fixed text with numbered placeholders), which means the multi-line alert layout would need adapting.
- The Meta test phone number can message only a small list of registered recipients (reported as up to five).
- Templates are billed per delivered message, by recipient country, and new accounts start with a daily cap on business-initiated conversations.
- Setup needs a Meta developer account, an app and a verified business for production use.

To add it, write a `WhatsApp: Format Message` node that builds the template parameters from `alert_fields`, a send node, and a rule in a channel Switch.

## Extending

- **Add a source:** add a section to Config, a branch that ends in the common schema, an error output wired to a `Problem: Source Failed` node, and merge it into `Gigs: Merge Sources`.
- **Add an alert channel:** add a formatter that escapes for that channel, a send node with its error output handled, and a Switch rule. The slim version has no Switch, so add one after `Alert: Prepare Fields` with a fallback to Stop and Error, or start from the full file.
- **Add an LLM provider:** copy a branch, keep the prompt, system message and schema identical, and add a Switch rule. The same applies: the slim version has no provider Switch.

## Credits and license

Data from the n8n community forum (Discourse), Hacker News via the Algolia search API, and Remotive. Released under the MIT License (see `LICENSE`).

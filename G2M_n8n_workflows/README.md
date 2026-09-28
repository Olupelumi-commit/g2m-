# G2M n8n Workflow Pack

A modular set of eight [n8n](https://n8n.io/) workflow exports for building a B2B go-to-market (GTM) process: define an ideal customer profile (ICP), discover and enrich leads, qualify and research them, generate outreach, manage follow-ups, and triage replies.

The workflows are starter architecture, not a turnkey production deployment. They deliberately contain no credentials and use provider-neutral HTTP nodes where possible.

## What is included

| # | Workflow | Entry point | Purpose |
| --- | --- | --- | --- |
| 01 | `01_icp_builder.json` | `POST /webhook/g2m/icp-builder` | Turns campaign criteria into a structured ICP and search queries. |
| 02 | `02_lead_discovery.json` | Manual | Runs the ICP search queries, normalizes search results, and removes duplicate URLs. |
| 03 | `03_lead_enrichment.json` | Manual | Enriches a company/contact and verifies an email address. |
| 04 | `04_lead_qualification.json` | Manual | Scores leads against the ICP and routes qualified versus rejected records. |
| 05 | `05_lead_research.json` | Manual | Searches for news, hiring, funding, and other observable signals, then summarizes them. |
| 06 | `06_outreach_generation.json` | Manual | Drafts a fact-based cold email and either holds it for review or sends it. |
| 07 | `07_follow_up_sequence.json` | `POST /webhook/g2m/follow-up` | Waits two days, checks whether a contact replied, and sends a follow-up only when they have not. |
| 08 | `08_response_intelligence.json` | `POST /webhook/g2m/inbound-response` | Classifies inbound replies, flags unsubscribes, and exposes a hand-off point for a CRM or human. |

Conceptually, the workflows fit together as follows:

```text
ICP Builder → Lead Discovery → Enrichment → Qualification → Research → Outreach
                                                                    │
                                      ┌─────────────────────────────┘
                                      ▼
                              Follow-up Sequence ← Reply Intelligence
```

They are intentionally exported as separate, independent workflows. Importing them does **not** connect them, pass records between them, or provide persistence. Add your own database/CRM and `Execute Workflow` or webhook hand-offs to make this a fully automated pipeline.

## Quick start

1. Run a current n8n instance with access to the services you choose.
2. In n8n, select **Workflows → Import from File** and import each JSON file.
3. Set the environment variables below in the environment where n8n runs, then restart n8n if necessary.
4. Open every imported workflow and replace the example/default endpoints, authentication, and field mappings with those required by your chosen providers.
5. Test with non-production data and keep all workflows inactive until their behavior is verified.
6. Activate webhook workflows only after n8n is reachable at a stable public URL. Use their production webhook URLs in calling systems.

Manual-trigger workflows expect an incoming lead/ICP record. During development, add a temporary **Set** node or call them from another workflow with **Execute Workflow**. Before activation, wire each workflow to your preferred queue, database, spreadsheet, CRM, or the preceding workflow.

## Configuration

Set these as n8n environment variables. The values below are names used directly in the JSON exports.

| Variable | Required by | Description |
| --- | --- | --- |
| `LLM_API_URL` | 01, 05, 06, 08 | OpenAI-compatible chat-completions endpoint. Defaults to `https://api.openai.com/v1/chat/completions`. |
| `LLM_API_KEY` | 01, 05, 06, 08 | Bearer token for the LLM endpoint. |
| `LLM_MODEL` | 01, 05, 06, 08 | Model identifier. Defaults to `gpt-4o-mini`. |
| `SEARCH_API_URL` | 02, 05 | Search endpoint. Defaults to Tavily’s search URL. |
| `SEARCH_API_KEY` | 02, 05 | Search-provider API key. |
| `ENRICHMENT_API_URL` | 03 | Company/contact enrichment endpoint. Defaults to Apollo’s organization-enrichment URL. |
| `EMAIL_VERIFY_URL` | 03 | Email-verification endpoint. Defaults to Hunter’s verifier URL. |
| `EMAIL_VERIFY_API_KEY` | 03 | Email-verification API key. |
| `SENDER_EMAIL` | 06, 07 | Sender address used by n8n’s Email Send node. Configure the node’s mail credentials separately. |
| `AUTO_SEND` | 06 | Set to `true` only to allow generated outreach to reach the Email Send node. Any other value keeps drafts on the approval path. |
| `REPLY_STATUS_URL` | 07 | Endpoint that returns a reply-status record for a `contact_id`. |

The workflows do not uniformly implement every provider’s authentication scheme. For example, the default enrichment request sends a domain but no Apollo API key. Provider schemas and authentication vary, so update the HTTP Request nodes and their field mappings after choosing a provider.

## Inputs and outputs

### ICP Builder

Send a JSON request to the webhook, for example:

```json
{
  "campaign_name": "US SaaS outbound",
  "offer": "Revenue operations consulting",
  "target_market": "B2B SaaS in the United States",
  "criteria": "50–500 employees; VP Sales, CRO, and RevOps leaders; hiring sales reps",
  "limit": 100
}
```

`criteria` may also be supplied as `icp`. The response includes the normalized campaign fields and an LLM-generated `icp` object containing industries, locations, company-size bounds, target titles, technology/pain/intent signals, exclusions, and `search_queries`.

### Lead record shape

Pass a consistent record through the middle workflows. At minimum, use `email` and either `domain`, `company_domain`, or `source_url`. The following fields enable more useful qualification and research:

```json
{
  "company": "Example Inc.",
  "domain": "example.com",
  "email": "person@example.com",
  "title": "VP of Sales",
  "industry": "SaaS",
  "employees": 175,
  "intent_signals": ["Hiring account executives"],
  "icp": { "target_titles": ["VP Sales"], "industries": ["SaaS"] }
}
```

The qualification workflow awards up to 100 points: company-size fit (25), title fit (25), industry fit (20), intent signals (20), and having an email (10). A score of 75 or more is marked qualified.

### Follow-up and reply webhooks

`POST /webhook/g2m/follow-up` needs `contact_id` and `email`; optional `followup_subject` and `followup_body` override the default message. Its reply-status service must return a record with a boolean `replied` field.

`POST /webhook/g2m/inbound-response` should include the sender’s `email` and the inbound message data your LLM needs to classify. The classification categories are `interested`, `meeting_request`, `question`, `not_interested`, `unsubscribe`, `out_of_office`, and `other`.

## Operational and compliance notes

- Verify API response shapes before relying on an HTTP node. The included normalization handles common Tavily/Serper-style result fields, but not every provider response.
- Store leads, search evidence, email status, replies, and suppressions in a durable system of record. The “Add Suppression” node currently creates an output record; it does not write to a suppression list on its own.
- Connect “Route To CRM / Human” to your CRM, help desk, or alerting channel. Connect “Await Approval” to a review/approval process if sending is not automated.
- Honor unsubscribe requests immediately, check suppression lists before every send, and apply the privacy, anti-spam, and email-delivery requirements that apply to your recipients and jurisdictions.
- Start with `AUTO_SEND` unset (or any value other than the literal string `true`), a test mailbox, rate limits, and a small internal test list. Review generated text for accuracy and tone before enabling delivery.
- Keep API keys in n8n credentials or deployment secrets—never in workflow JSON, Git, or incoming webhook payloads.

## Suggested production extensions

For a robust deployment, add a data store/CRM, idempotency keys, retries with backoff, rate limiting, provider error handling, observability, a review queue, event logging, consent/suppression checks, and clear ownership for reply handling. Also replace the placeholder reply-status service with a real integration to your email or CRM provider.

## License

No license file is included. Add one before distributing or reusing this project outside its intended team.

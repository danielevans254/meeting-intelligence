# Meeting intelligence workflow

`workflows/meeting-intelligence.json` accepts Fireflies, Microsoft Teams, Google Meet, Zoom, or a canonical transcript payload; generates a structured meeting brief; creates one destination task per action item; and posts a single digest to Slack or Teams.

## Flow

1. A POST webhook accepts an event. `Normalize Meeting` maps provider-specific fields to one contract and rejects missing IDs or tiny transcripts.
2. `Test Mode?` returns the fixture's `mock_summary`; production routes to OpenAI, Anthropic, or Gemini.
3. `Parse and Validate Summary` strips markdown fences, parses JSON, validates arrays, and normalizes action items.
4. `Build Task Requests` emits one item per action. Empty action lists emit a skip marker so the notification still runs.
5. Jira or Airtable is called only in production. Test mode returns deterministic IDs such as `MOCK-TASK-001`.
6. Tasks are aggregated and exactly one Slack or Teams digest is sent. The last node returns a compact execution receipt.

## Canonical request

```json
{
  "test_mode": true,
  "provider": "fireflies",
  "event_id": "provider-event-id",
  "title": "Weekly launch review",
  "participants": [{ "name": "Maya", "email": "maya@example.com" }],
  "transcript": { "sentences": [{ "speaker_name": "Maya", "text": "..." }] },
  "config": {
    "llm_provider": "openai",
    "task_system": "jira",
    "notification_system": "slack"
  },
  "mock_summary": {
    "summary": "Only used in test mode.",
    "key_points": [],
    "decisions": [],
    "risks": [],
    "action_items": []
  }
}
```

Supported provider aliases are `fireflies`, `teams`, `meet`, `google_meet`, `zoom`, and `canonical`. When a provider's native webhook does not include transcript text, first use its API/download step and send the completed transcript to this workflow.

## Test cases

After importing and activating, copy the Test URL shown by `Meeting Transcript Webhook` and run:

```powershell
$body = Get-Content -Raw .\1-meeting-intelligence\fixtures\fireflies-airtable-slack.json
Invoke-RestMethod -Method Post -Uri '<N8N_TEST_WEBHOOK_URL>' -ContentType 'application/json' -Body $body
```

Expected: two mock Airtable tasks, one mock Slack notification, status `completed`.

Repeat with `teams-jira-teams.json`. Expected: one mock Jira issue and one mock Teams notification. No credentials are required for either test.

## Production configuration

- Set one LLM provider and model in the environment or replace the HTTP authentication with n8n credentials.
- Configure Jira project key or Airtable base/table.
- Configure Slack or Teams incoming webhook URL.
- Put the public webhook behind signature verification. Each transcript provider signs events differently, so signature validation belongs in the provider-specific ingress or API gateway.
- Add a durable idempotency lookup keyed by `event_id` before enabling retries. Destination task keys should include the meeting ID and action index.
- Store only the transcript retention period your privacy policy permits.

## Output contract

The response contains `event_id`, provider, summary, normalized action items, created task references, notification destination, status, and timestamps. It never returns API keys or raw provider responses.


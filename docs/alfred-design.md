# Alfred — Email Triage and Draft Workflow

A serverless, event-driven email assistant on AWS. Inbound mail is classified by Claude (via Bedrock), forwarded to a human operator with a summary, and — when the operator replies with instructions — a Reply All draft is prepared in the operator's Drafts folder for manual review and send.

**Alfred never sends email autonomously.** Drafts only. Human in the loop on every outbound message.

---

## Goal

Configure an OpenClaw installation to operate as an email-facing assistant under the identity **Alfred** for a Google Workspace mailbox. Alfred:

1. Receives inbound email via Gmail API push notifications
2. Classifies and summarizes the message using Bedrock Haiku
3. Forwards the message to the operator's personal Gmail with a plain-English summary and recommendation
4. Waits — possibly for days — for the operator's reply with instructions
5. Parses the instructions and creates a Reply All draft in the operator's Drafts folder
6. Notifies the operator that the draft is ready

The operator opens Gmail, reviews the draft, edits if needed, and clicks Send. Alfred's job ends with the draft.

---

## Architecture

```
inbound email → Workspace mailbox
        ↓ Gmail API watch()
Google Cloud Pub/Sub topic
        ↓ authenticated push subscription
AWS API Gateway: POST /alfred/webhook
        ↓
Lambda: alfred-intake               (verify OIDC token, enqueue SQS)
        ↓
SQS: alfred-email-events            (durability, decoupling)
        ↓
Lambda: alfred-history-fetcher      (gmail history.list, start Step Functions)
        ↓
Step Functions: AlfredEmailWorkflow (Standard Workflow)
    ├── Triage              → Lambda + Bedrock Haiku (classify, summarize)
    ├── Forward             → Lambda + Gmail API (send Alfred note to operator)
    ├── AwaitInstruction    → waitForTaskToken (blocks days until operator replies)
    ├── ParseInstruction    → Lambda + Bedrock Haiku (interpret operator's reply)
    ├── CheckIntent         → Choice state (draft_reply / no_action / clarify)
    ├── CreateDraft         → Lambda + Gmail API (drafts.create in operator's mailbox)
    ├── Notify              → Lambda + Gmail API (confirmation to operator)
    └── Closed              → Succeed

DynamoDB:
    alfred-work-items       (state per email, GSI on alfred_forward_message_id)
    alfred-mailbox-state    (historyId tracking, watch expiration)

Secrets Manager:
    alfred/workspace-oauth-token   (Workspace mailbox refresh token)
    alfred/gmail-oauth-token       (operator's personal Gmail refresh token)

EventBridge:
    Scheduled rule every 6 days → Lambda: alfred-watch-renewer
    (Gmail watch expires every 7 days)
```

---

## Key design decisions

### Push over polling

Gmail API `watch()` + Google Cloud Pub/Sub sends near-real-time notifications when new mail arrives. Polling `messages.list` every 60 seconds wastes API quota and adds latency. The push pattern is Google's recommended approach for server-side Gmail integrations.

### Step Functions Standard Workflow (not Express)

Express Workflows max out at 5 minutes. The `AwaitInstruction` state can block for hours or days waiting for the operator's reply. Standard Workflows support executions up to one year and have native `waitForTaskToken` for human-in-the-loop patterns.

### OAuth2 user credentials (not domain-wide delegation)

Google's own documentation recommends against DWD when alternatives exist. DWD cannot be scoped to a single user — it grants impersonation of the entire Workspace domain. OAuth2 user credentials with stored refresh tokens work for both accounts (Workspace and personal Gmail) with the same code path.

### SQS between intake and Step Functions

Decouples the Pub/Sub ingestion layer from the processing layer. If Step Functions is temporarily unavailable or throttled, notifications queue safely and are retried.

### Bedrock Haiku for triage and instruction parsing

Triage is classification and summarization — not complex reasoning. Haiku is ~20× cheaper than Sonnet and fast enough. Full Sonnet is available for the path if triage quality needs improvement.

### Server-side tool use, not bash sidecars

The earlier design considered routing model decisions through a Python or bash sidecar that would interpret the model's intent and call Gmail. The chosen pattern instead implements each action — `forward`, `create_draft`, `notify` — as a discrete Lambda. The model's tool calls go directly to AWS service infrastructure, with no intermediate translation layer to break or bug.

---

## Workflow status machine

```
received → triaged → forwarded → awaiting_instruction
         → instruction_received → draft_created → closed
```

- Error states: `error`
- Timeout state: `closed` (AwaitInstruction `HeartbeatTimeout` → Succeed, no action)

---

## Thread correlation

Alfred's forwarded email `Message-ID` is stored in DynamoDB under `alfred_forward_message_id`. When the operator replies, the reply's `In-Reply-To` header contains that `Message-ID`. This is the lookup key — no heuristics, no fuzzy matching.

The Step Functions task token is stored in DynamoDB at the same key, enabling `instruction-receiver` to call `SendTaskSuccess` and resume the paused execution.

---

## Draft threading (RFC 2822)

Draft created in the operator's mailbox with:

- `In-Reply-To: {original_external_message_id}` — the original sender's message, **not** Alfred's forward
- `References: {original_references} {original_message_id}` — the full chain per RFC 2822
- `Subject: Re: {original_subject}` — exact match required by Gmail's threading
- `To`: original sender; `CC`: original CC minus the Workspace mailbox

The draft lands in the operator's Drafts folder. When the operator sends it, the recipient's mail client threads it as a reply to the original email — no awkward "Fwd: Fwd:" mangling, no broken thread.

---

## Defaults

| Question | Default |
|---|---|
| Forward everything or filter spam? | Forward everything; flag probable spam (≥0.85 confidence) in the note |
| Accept operator instructions only from the operator's address? | Yes — strict allowlist |
| Multiple operator replies — newest wins? | Yes — delete old draft, create new one |
| Redraft = same draft or new version? | Delete old, create fresh |
| Final draft from operator's address or send-as Workspace address? | Operator's address only (v1) |

---

## File layout

```
tools/alfred/
├── lambdas/
│   ├── shared/
│   │   ├── gmail_client.py              Build authenticated Gmail service from Secrets Manager
│   │   ├── dynamo.py                    DynamoDB helpers for work items and mailbox state
│   │   └── requirements.txt             google-auth, google-api-python-client, boto3
│   ├── intake/handler.py                Verify Pub/Sub OIDC token, enqueue SQS
│   ├── history_fetcher/handler.py       gmail history.list, start Step Functions execution
│   ├── triage/handler.py                Bedrock Haiku: classify and summarize
│   ├── forwarder/handler.py             Gmail API: send Alfred note; store task token
│   ├── instruction_receiver/handler.py  Operator reply → correlate via In-Reply-To → SendTaskSuccess
│   ├── instruction_parser/handler.py    Bedrock Haiku: interpret operator's instruction
│   ├── drafter/handler.py               Gmail API: construct MIME draft with RFC 2822 headers
│   ├── notifier/handler.py              Gmail API: send confirmation / error / clarification
│   └── watch_renewer/handler.py         Re-register Gmail watch every 6 days
├── infra/
│   ├── alfred.yaml                      CloudFormation: all AWS resources
│   └── statemachine.asl.json            Step Functions ASL definition
├── scripts/
│   ├── setup_oauth.py                   One-time OAuth2 browser flow for both accounts
│   └── setup_watch.py                   Register Gmail watch + Pub/Sub push subscription
└── tests/
    ├── unit/test_triage.py              Bedrock classification, fallback on malformed JSON
    ├── unit/test_forwarder.py           Forward format, spam flag, task token storage
    ├── unit/test_drafter.py             RFC 2822 headers, Reply All recipients, draft deletion
    └── integration/test_workflow.py     ASL structure validation, local Step Functions execution
```

---

## AWS resources

| Resource | Name | Notes |
|---|---|---|
| DynamoDB | `alfred-work-items` | pk: message_id; GSI on alfred_forward_message_id |
| DynamoDB | `alfred-mailbox-state` | pk: mailbox; tracks historyId and watch expiration |
| Secrets Manager | `alfred/workspace-oauth-token` | Workspace mailbox OAuth2 refresh token |
| Secrets Manager | `alfred/gmail-oauth-token` | Operator's personal Gmail OAuth2 refresh token |
| SQS | `alfred-email-events` | With DLQ, 3 retries |
| API Gateway | `alfred-webhook` | POST /alfred/webhook |
| Lambda | `alfred-intake` | Pub/Sub verification + SQS enqueue |
| Lambda | `alfred-history-fetcher` | Gmail history.list + SFN start |
| Lambda | `alfred-triage` | Bedrock Haiku triage |
| Lambda | `alfred-forwarder` | Gmail send + task token storage |
| Lambda | `alfred-instruction-receiver` | Operator reply handler + SendTaskSuccess |
| Lambda | `alfred-instruction-parser` | Bedrock Haiku instruction parse |
| Lambda | `alfred-drafter` | Gmail drafts.create |
| Lambda | `alfred-notifier` | Gmail send confirmation/error |
| Lambda | `alfred-watch-renewer` | Re-register Gmail watch |
| Step Functions | `AlfredEmailWorkflow` | Standard Workflow, 30-day max wait |
| EventBridge | `alfred-watch-renewal` | Scheduled: every 6 days |

---

## One-time setup sequence

1. Create a Google Cloud Project; enable Gmail API and Pub/Sub API
2. Create OAuth2 credentials (web app type); download the client JSON
3. Deploy the CloudFormation stack — provisions all AWS infrastructure
4. Run the OAuth2 browser flow for both accounts; refresh tokens stored in Secrets Manager
5. Create a Google Cloud Pub/Sub push subscription pointing to the CloudFormation `WebhookUrl` output
6. Register Gmail `watch()` on both mailboxes; store initial `historyId`

---

## Non-goals (v1)

- No autonomous email sending
- No calendar integration
- No CRM
- No multi-operator support
- No send-as from the Workspace address (operator's address only)
- No delegation to other people

These are explicit non-goals, not omissions. Each one is a deliberate scope cut to keep v1 small and verifiable. Subsequent versions can add them once the core loop is reliable.

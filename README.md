# Lead Friendly MCP servers

Lead Friendly runs an MCP server that lets ChatGPT, Claude or any MCP client make phone calls, send SMS and read and update CRM records through one account, under the same compliance checks a salesperson gets.

The calls are placed by the account's own AI voice agents and the texts go out from its own number. An assistant is held to one rule a salesperson is not: a marketing text or call needs a consent record on the contact. Everything it does, and everything it is refused, is logged for the account owner.

This repository documents the two hosted servers. There is nothing to install: both run at leadfriendly.com and speak Streamable HTTP.

- Website: https://www.leadfriendly.com/mcp
- Developer reference: https://www.leadfriendly.com/developers
- Step-by-step connection guide: https://www.leadfriendly.com/guides/connect-your-ai-assistant

## Endpoints

| Server | URL | Authentication | Registry name |
| --- | --- | --- | --- |
| Account server: calls, SMS and CRM for one account | `https://www.leadfriendly.com/api/mcp/v1` | API key | `com.leadfriendly/lead-friendly` |
| Public server: questions about Lead Friendly itself | `https://www.leadfriendly.com/api/mcp` | None | `com.leadfriendly/lead-friendly-info` |

Both use Streamable HTTP: JSON-RPC 2.0 over POST, request and response, with no SSE stream. Both answer `initialize`, `tools/list`, `tools/call` and `ping`, and accept a batch of messages in one request. The account server defaults to protocol version `2025-06-18` and the public server to `2024-11-05`; both echo the version a client asks for. A `GET` on either URL returns a small descriptor.

## Quick start

### 1. Create a key

An owner or admin of a Lead Friendly account opens **Settings → API keys** and creates one key per assistant, at one of three access levels:

- **Read only**: looks things up and changes nothing. The assistant sees the 19 read tools.
- **Read + setup**: can also build and edit AI agents, set business hours, route numbers and create campaigns and automations, all switched off. It cannot text, call or book anyone, buy numbers, or change contacts or deals. 30 tools.
- **Read + write**: everything, including texting, calling, booking and buying numbers, through the compliance checks below. All 38 tools.

The key (`lf_live_…`) is shown once; Lead Friendly stores only a hash. Next to it you get a connection address with the key built in:

```
https://www.leadfriendly.com/api/mcp/v1/k/lf_live_…
```

### 2. Claude

In Claude, go to **Customize → Connectors**, add a custom connector and paste the connection address as the server URL. On Team and Enterprise plans an Owner adds the connector first, under **Organization settings → Connectors**. Switch it on in a conversation from the **+** menu, under **Connectors**.

### 3. ChatGPT

On the web: **Plugins → Add → Create custom MCP server**. Give it a name, paste the connection address as the server URL and choose **No authentication**. ChatGPT asks you to confirm write actions by default.

The connection address contains the key, so treat it like a password. Only `lf_live_` keys work in an address, and revoking the key in Settings → API keys kills the address at once. The safer choice for an address is a read-only key; use a read + write key only when you want the assistant to text, call or book.

### 4. Claude Code and other MCP clients

A client that can send a header should send the key as a header rather than in the address:

```bash
claude mcp add --transport http lead-friendly https://www.leadfriendly.com/api/mcp/v1 \
  --header "Authorization: Bearer lf_live_…"
```

Any other Streamable HTTP client: URL `https://www.leadfriendly.com/api/mcp/v1`, header `Authorization: Bearer lf_live_…`. If your client reserves the `Authorization` header for its own sign-in, send `X-LF-Api-Key: lf_live_…` instead.

### 5. Try it with curl

The public server needs no key, so this works as written:

```bash
curl -s https://www.leadfriendly.com/api/mcp \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

Call a tool on it:

```bash
curl -s https://www.leadfriendly.com/api/mcp \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"get_pricing","arguments":{}}}'
```

The same `tools/list` against your own account:

```bash
curl -s https://www.leadfriendly.com/api/mcp/v1 \
  -H 'Authorization: Bearer lf_live_…' \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

## Tools

### Account server: 38 tools

A key's access level decides which tools an assistant can see: Read only is shown the 19 read tools, Read + setup the 19 read and 11 setup tools, and Read + write all 38. Any tool outside the key's level is refused. Tools marked *(dealerships)* cover deals, F&I and the desk, and apply to car dealers only. Every tool carries MCP annotations (`readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`) in `tools/list`.

#### Read: 19 tools that change nothing

| Tool | What it does |
| --- | --- |
| `search_contacts` | Find contacts by name, phone, email or company, with tag and status filters. Shows who is on do-not-call. |
| `get_contact` | One contact in full: the timeline of calls, texts, appointments and notes, plus consent and opt-out status. |
| `list_deals` | Vehicle deals on the desk by stage, with price, down payment, term and trade. *(dealerships)* |
| `get_deal` | One deal in full, including the documents still missing or unsigned and the lender's decision. *(dealerships)* |
| `list_appointments` | Upcoming appointments in date order, with the contact each one belongs to. |
| `list_calls` | Recent calls: who called, which AI agent answered, the outcome and a short summary. Filter by callback needed or topic. |
| `get_call` | One call's transcript, outcome and compliance flags, with a recording link that expires after five minutes. |
| `check_outreach_basis` | Whether one contact may be texted or called for a given purpose, and on what grounds, without sending anything. |
| `get_org_status` | The account's readiness to act: wallet, identity verification, 10DLC registration, calling window, live agents and numbers. |
| `list_ai_agents` | The account's AI voice agents and whether each one is live. |
| `get_ai_agent` | One agent's full configuration: scripts, greetings, knowledge, voice, booking and transfer setup. |
| `list_agent_templates` | The industry templates a new AI agent can be built from. |
| `get_business_hours` | Opening hours, time zone and dated exceptions such as holidays. |
| `list_campaigns` | Outbound calling campaigns with status, schedule and outcome totals. |
| `list_workflows` | Automations with their triggers, status and current enrolments. |
| `list_phone_numbers` | The account's numbers, which agent each one rings, SMS capability and health. |
| `list_fi_products` | The F&I product menu: GAP, service contracts, maintenance and the rest. *(dealerships)* |
| `get_desk_defaults` | The fees, tax rates and term and rate presets used to structure a deal. *(dealerships)* |
| `search_available_numbers` | Phone numbers available to buy, with the monthly cost. Buys nothing. |

#### Write: 8 tools that reach customers, records or money

| Tool | What it does |
| --- | --- |
| `create_contact` | Create a contact. Dedupes on phone, then email, so running the same list twice is safe. |
| `update_contact` | Change a contact's status, name, email or company, or add a tag. Phone numbers stay a human-only change. |
| `log_activity` | Write a note, task or outcome onto a contact's timeline, marked as written by an AI agent. |
| `send_sms` | Text a contact from the account's own number, after the compliance checks below. |
| `start_outbound_call` | Have one of the account's AI voice agents call a contact, after the pre-dial checks below. |
| `book_appointment` | Book an appointment. The booking service handles conflicts and sends the customer's confirmation. |
| `update_deal_stage` | Move a deal to a new stage. The move is checked against the legal transitions and carries its filing and inventory side effects. *(dealerships)* |
| `buy_number` | Buy a phone number. The first call only quotes the cost; a second call with confirm:true buys it. |

#### Setup: 11 tools that build things switched off

Setup tools run on a Read + setup or Read + write key.

| Tool | What it does |
| --- | --- |
| `create_agent_from_website` | Build an AI voice agent from a template and the business's website. It starts in review and answers nothing. |
| `update_agent_script` | Edit an agent's script, greeting, knowledge or voice. It cannot switch an agent on. |
| `assign_number_to_agent` | Route one of the account's numbers to an agent. |
| `set_business_hours` | Set the daily opening hours and time zone. |
| `create_campaign` | Create an outbound calling campaign as a draft that dials nobody until a person turns it on. |
| `create_workflow` | Create an automation, paused. |
| `pause_campaign` | Pause a running campaign. There is no tool that starts one. |
| `pause_workflow` | Pause a running automation. There is no tool that starts one. |
| `create_fi_product` | Add a product to the F&I menu. *(dealerships)* |
| `set_fi_product_active` | Show or hide an F&I product. There is no delete. *(dealerships)* |
| `set_desk_defaults` | Change the store's fees, tax rates and deal presets. *(dealerships)* |

### Public server: 5 tools, no key

| Tool | What it does |
| --- | --- |
| `get_pricing` | Current plans and prices, read from the same source as the pricing page. |
| `list_industries` | The industries Lead Friendly ships an AI agent template for, with an optional keyword filter. |
| `check_capability` | Whether Lead Friendly can do a given job, answered from a curated list. |
| `book_demo` | Request a demo for a prospect. Returns the self-serve booking link. |
| `create_lead` | Hand an interested prospect to the Lead Friendly sales team. Never writes into a customer's CRM. |

The public server reads no customer data and is rate-limited per IP address: one request may carry at most 10 JSON-RPC messages, and `book_demo` and `create_lead` are limited to 5 calls an hour per IP.

## Safety model

The rules live in the server, not in the prompt, so they hold when the model gets something wrong.

- **The key decides the account.** Each key belongs to one Lead Friendly account, and the server works out the account from the key, never from anything in the request. An id from another account is refused as not found.
- **Access levels.** A key is shown only the tools its level allows, and any other tool it names is refused. Only an owner or admin can create keys. Revoking a key cuts that assistant off on its next request and leaves other keys working.
- **The same checks as a text or call from the app.** Every text runs 10DLC registration, identity verification, do-not-call, STOP opt-outs, and 8am to 8pm in the contact's own time zone, narrower for marketing in some states. Every call runs the pre-dial checks: wallet balance, identity verification, an E911 address on the calling number, the account's AI calling window, limits on how often one number can be dialled in a day and in 14 days, and prior express written consent for AI marketing calls. The AI voice agent always says it is an AI, and nothing on the MCP server can turn that off.
- **One rule stricter than for people.** A marketing text or call from an assistant needs an affirmative consent record on the contact. An assistant can call a message transactional, but that claim is checked against a real appointment, deal or earlier inbound message before it is believed. `check_outreach_basis` runs the same check without sending anything.
- **What it builds starts switched off.** Agents an assistant builds land in review, campaigns as drafts and workflows paused. No tool switches any of them on, deletes a record, releases a number or changes the plan. `buy_number` needs a second call with `confirm: true` after a person approves the quoted cost.
- **Sensitive data stays out.** Credit-application identifiers, income and bureau scores are encrypted and never returned. Call recordings come only as signed links that expire after five minutes.
- **Everything is logged.** Every tool call, refusals included, is recorded and shown to the account owner in **Settings → AI agents**: the tool, the result and the reason, plus the names the connected clients gave themselves. Arguments are stored only as a hash, so message text and customers' phone numbers are not copied into the log.

## Refusals are answers

A refused tool call is not a protocol error. It comes back as a normal tool result with `isError: true` and a structured body, so the assistant can tell a person what to fix instead of retrying:

```json
{
  "ok": false,
  "code": "COMPLIANCE_BLOCKED",
  "reason": "<what happened, in one sentence>",
  "remediation": "This contact replied STOP. Under the TCPA they cannot be texted again until they opt back in by texting START themselves. Do not retry.",
  "detail": { "guard_reason": "opted_out" }
}
```

Codes: `UNAUTHENTICATED`, `FORBIDDEN_SCOPE`, `AGENT_DISABLED`, `RATE_LIMITED`, `INVALID_INPUT`, `NOT_FOUND`, `COMPLIANCE_BLOCKED`, `INSUFFICIENT_WALLET`, `UPSTREAM_ERROR`, `INTERNAL_ERROR`. Every refusal carries a `remediation`.

## Limits

Account server defaults, which Lead Friendly can adjust per account:

| Limit | Default | Counted |
| --- | --- | --- |
| Tool calls per minute | 120 | per key |
| Changes per day | 500 | per key |
| Texts and calls per day | 200 | per account, across every connected assistant |

Only successful actions count toward the daily ceilings; refusals do not. Daily counters reset at midnight UTC. One HTTP request to the account server may carry at most 20 JSON-RPC messages, and they run one after another.

## Pricing

- The public server is free and needs no account.
- Connecting an assistant to a Lead Friendly account costs nothing, and there is no charge per tool call.
- Texts and AI calls it places are billed like any other text or call on the account: the plan's included minutes and messages first, then the prepaid wallet at the plan's rate. If the balance cannot cover a send, the tool refuses with `INSUFFICIENT_WALLET`.
- A phone number bought through `buy_number` costs $3 a month, charged to the wallet when it is bought and every 30 days after.

Plans and rates: https://www.leadfriendly.com/pricing. The public server's `get_pricing` tool returns them live.

## Contact

- Product and support: hello@leadfriendly.com
- Security reports: security@leadfriendly.com

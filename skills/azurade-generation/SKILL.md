---
name: azurade-generation
description: "Generate images and video through the Azurade AI API: list models and prices, estimate the credit cost of a generation, submit it, poll it and download the result; screen or rewrite a prompt on its own. Use when a task needs text-to-image, image-to-image, text-to-video or image-to-video generation against a credit balance."
---

# Azurade AI generation API

Azurade AI runs image and video models behind one account and one credit
balance. The web app and the API are the same product: an API key draws on the
same balance, at the same per-generation prices, with no separate developer
plan and no separate price list.

Base URL: `https://azurade.com`
OpenAPI 3 document: `https://azurade.com/api/v2/openapi.json` (generated from the
running code; the authoritative field list, enums and error codes)
Interactive docs: `https://azurade.com/swagger/`

## Authentication

Every route below requires an authenticated caller, except the model catalog
and the OpenAPI document. Two ways:

- **API key**: `Authorization: Bearer sk-...`. The same secret is also
  accepted as `X-API-Key: sk-...`.
- **User session**: a PocketBase auth token in the `Authorization` header,
  which is what the web app uses.

Keys are created and deleted by the account holder on the profile page
(`https://azurade.com/profile/`). The secret is shown once, stored hashed and cannot
be read back, so save it when you create it. A key signs the routes below and
can revoke itself (`POST https://azurade.com/api/oauth/revoke`); it cannot create keys,
delete other keys, pay, cancel a subscription or change the account. Accounts are created at `https://azurade.com/login/`.

MCP clients can also connect through OAuth instead of a pasted key; see the
MCP section below and `https://azurade.com/auth.md`.

## MCP server

The same API is an MCP server at `https://azurade.com/mcp` (streamable HTTP, one
stateless POST per call). Clients that support MCP authorization (Claude,
ChatGPT, Claude Code, Codex, Gemini CLI, Cursor, VS Code) only need the URL: the first
connection opens an Azurade page in the browser where the person signs in and
clicks Allow. The connection is an API key named after the client, listed on
the profile page, where deleting it disconnects the client.

- Claude Code: `claude mcp add --transport http azurade https://azurade.com/mcp`, then
  `/mcp` in a session to sign in.
- Codex: `codex mcp add azurade --url https://azurade.com/mcp`, then
  `codex mcp login azurade`.
- Gemini CLI: `gemini mcp add --transport http azurade https://azurade.com/mcp`, then
  `/mcp auth azurade` in a session.
- Cursor (`~/.cursor/mcp.json`):
  `{"mcpServers": {"azurade": {"url": "https://azurade.com/mcp"}}}`.
- VS Code (`.vscode/mcp.json`):
  `{"servers": {"azurade": {"type": "http", "url": "https://azurade.com/mcp"}}}`.
- Claude (web and desktop): Customize > Connectors > + > Add custom connector,
  with the URL `https://azurade.com/mcp`.
- ChatGPT: turn on Developer mode (Settings > Security and login), then create
  an app with the URL `https://azurade.com/mcp`.
- A client without OAuth sends a key in a header instead:
  `Authorization: Bearer sk-...`.

Tools: `list_models`, `get_model`, `estimate_generation`,
`create_generation`, `get_generation`, `list_generations`,
`create_upload_url`, `enhance_prompt`, `moderate_prompt`,
`get_account`, `buy_credits`. They run the same handlers as the HTTP routes below, with the
same prices, refunds and error codes (a failed call is a tool result with
`isError` and the API's `{"error", "code"}` body). Model parameters may be
sent as strings (`"5"`, `"true"`, one URL for a list of URLs); the server
converts them to the field's type. `create_generation` and
`get_generation` wait up to about 50 seconds for the job to finish; a job still
running comes back `pending` with its id and a `next` hint, and
`get_generation` waits again. Pass `"wait": false` to `create_generation`
to get the pending job back as soon as it is submitted. A request that fails to start comes back with
code `generation_failed`, the reason and the job id, and costs nothing. Every
generation carries `charged`, what it took from the balance (0 once failed),
and `create_generation` and `get_generation` carry the current `balance`.
`list_generations` takes `status`, `model` and `type` filters.
A finished image comes back with its link and an inline preview, a finished
video with its poster as the preview. For a file on
the agent's machine, `create_upload_url` returns a link and a curl command
that uploads it without a key. When the balance runs out, `buy_credits`
returns a checkout link to give the user. Server card: `https://azurade.com/mcp/server-card`.

## Models and prices

Public, no key needed:

```
GET /api/v2/models
GET /api/v2/models/{id}
```

Each model lists its name, type (image or video), `best_for` (what it is
good at, in one line), `inputs` (text, image, video, audio),
`max_resolution` and `max_duration` (seconds) where it has them, the fields
it accepts with defaults, allowed values (matched without case, so `4k` is
`4K`) and, for numbers, `minimum` and `maximum`, and `pricing.credits`:
the total for a request that sends only a prompt, the same figure the estimate
answers for it (a video model's is the whole default clip, not a per-second
rate). `pricing.per` names the unit the price multiplies by, `second` of
video or `image` asked for, at `pricing.unit_credits` each; it is absent for
a price per request. `pricing.varies` is true when a request can cost
something other than `pricing.credits`: the price is per unit, or a change
listed in `pricing.changes` costs something else. `pricing.changes` lists the
prompt-only request with one field changed, where that changes the price, as
`field`, `value` (`any` for an input file, meaning one is given) and
`credits`, the whole request's total with every other field at its default;
estimate any other combination before submitting. `requires_image` is true only for models that cannot run
without an input image or video; any model with an `image`, `image_input` or
`last_image` field also takes one, which is how a video model does
image-to-video. Limits that are not published (prompt length, how many input
images) come back from the estimate as `400 validation_failed` with the
allowed range in the message, at no charge.

## Generating

One endpoint for images and videos; the model decides which:

```
POST /api/v2/generations/estimate
POST /api/v2/generations
Content-Type: application/json

{"model": "<model id>", "prompt": "..."}
```

The estimate runs the checks the generation runs and answers `{"model", "type", "cost": <credits>, "variant", "balance", "affordable", "shortfall"}`
without charging; a missing prompt is the same `400 validation_failed`
there. Input images and videos are optional on the estimate, so price before
uploading; where an input changes the price, send it. Read
`variant` as well as `cost`: it names the price tier the request bills
against. `affordable` says whether the balance covers the cost, and
`shortfall` how many credits are missing when it does not. The generation answers `202` with `{"id", "model", "type",
"status": "pending", "cost"}` and reserves the credits. A job that cannot be
started answers `502 generation_failed` with its `id` and charges nothing;
`GET /api/v2/generations/{id}` carries the reason a few seconds later.
Parameters beyond
`model` vary per model (`GET /api/v2/models/{id}`); a field the model does
not take, a missing required field or an invalid value answers
`400 validation_failed` naming the field. Before anything is charged, the
generation fetches the first byte of every input URL; one that does not
download, or is a web page or text rather than a file, is a
`400 validation_failed` naming the field.
Set `"enhance_prompt": true` on any model that takes a prompt to have
the prompt rewritten with the model's own prompting guide first, at no extra
charge; the generation's `prompt_used` then shows what the model was given.
Send `"idempotency_key"` (any string up to 255 characters) with a generation:
repeating the request with the same key, after a timeout say, returns the first
job, marked `"idempotent_replay": true`, instead of starting and charging a
second, and while the first is still
being submitted the repeat answers `409 idempotency_in_progress`. A different
request under a key already used answers `422 idempotency_key_reused`.
Send `"callback_url"` (an https URL) to skip polling: when the job is final,
completed with its outputs or failed or canceled with its error, it is POSTed
there once, as JSON, the same body `GET /api/v2/generations/{id}` answers. A network error,
a 5xx or a 429 is retried for about six minutes. Nothing signs the request, so
read the job back before trusting it.

Input images, videos or audio (mp3, wav, m4a, aac, ogg) are uploaded first as
multipart form data in a field named `file`, up to 95 MB, one file per request:

```
POST /api/v2/uploads
```

Pass the returned `url` in the model's image, video or audio field (for example
`image`, `image_input`, `video_input` or `audio`; each such field's `input` in
`GET /api/v2/models/{id}` names the kind of file it takes).

Generation is asynchronous. Poll:

```
GET /api/v2/generations/{id}
GET /api/v2/generations?limit=20&cursor=...&status=failed&model=<id>&type=video
```

`status` moves from `pending` to `completed` or `failed` (also
`stalled` or `canceled`). The files are attached a moment after the status
turns `completed`, so keep polling until `status` is `failed`, or
`completed` with a non-empty `outputs` array. A completed generation carries `outputs`, each
with a `url`: an unguessable link that downloads without credentials, so
anyone you share it with can fetch the file. A
failed one carries `error` with the reason a caller can act on. Both routes
are owner-scoped: a key only ever sees the generations of its own account, and
another account's id answers 404.

## Prompt tools

Two standalone tools, each charged per call (their credit price is the
`prompt-enhance` and `prompt-moderate` rows of the public price list, 1
credit each at launch):

```
POST /api/v2/prompts/enhance   {"prompt": "...", "model": "<optional model id>"}
POST /api/v2/prompts/moderate  {"prompt": "..."}
```

Enhance answers the rewritten prompt, using the model's prompting guide when a
model is given. Moderate runs the same content-policy screen a generation runs
and answers `decision`: `allow` or `deny`, without generating.
Both answer the `balance` left after the charge. Nothing is charged when the
upstream service fails (503).

## Account

`GET /api/v2/account` answers the account id, email and current credit
balance.

`POST /api/v2/checkout` with `{"pack": ...}`, one of the packs in the
OpenAPI document's `CheckoutRequest`, answers 201 with a `checkout_url`: a single-use card checkout for the
account holder, and its price in `currency`. Send their `country` (ISO
3166-1 alpha-2, such as `DE`) for what they will pay: `subtotal`, the sales
tax or VAT as `tax`, and `total`, in that country's currency, or in
`currency` (ISO 4217, such as `EUR`) when sent. The checkout opens with the
same country and currency and does not let the person change the country; a
code it does not take, or a country payments are not available in, answers
400 `invalid_billing`. Without a country only `subtotal` comes back, before
tax: the checkout adds the tax of the country the person picks there. The
OpenAPI description lists each pack's base price. Tell the person the price and give
them the link when a call answers `insufficient_credits`; the credits are
added the moment they pay, with no further call.

## Billing behavior worth relying on

- Credits do not expire, do not reset monthly and are not capped by a rollover
  limit. A batch job that runs once a quarter draws on a balance topped up
  months earlier.
- A failed generation is refunded to the balance automatically. No ticket, no
  approval step.
- A model priced per image that delivers fewer images than asked refunds the
  rest; the generation's `cost` is what it finally took.
- One-time credit packs work without a subscription.

## Errors

Every non-2xx answer is `{"error": "<text>", "code": "<snake_case>"}`.
Codes worth handling: `unauthorized` (401), `unknown_model`,
`validation_failed` and `invalid_body` (400), `insufficient_credits`
(402, the body also carries `cost` and `balance`; buy more through
`POST /api/v2/checkout`), `not_found` (404).

Every prompt is screened against our content policy before it reaches a
model. A blocked prompt answers 400 with `code` `prompt_rejected` and
charges nothing; revise the prompt rather than retry
it. If the screener itself is down the answer is 503 `moderation_unavailable`,
which is safe to retry; the same holds for 503 `enhance_unavailable`.

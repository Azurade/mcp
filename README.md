# Azurade MCP server

[![Install in Cursor](https://cursor.com/deeplink/mcp-install-dark.svg)](https://cursor.com/en/install-mcp?name=azurade&config=eyJ1cmwiOiJodHRwczovL2F6dXJhZGUuY29tL21jcCJ9)
[![Install in VS Code](https://img.shields.io/badge/VS_Code-Install_Server-0098FF?style=flat-square)](https://insiders.vscode.dev/redirect/mcp/install?name=azurade&config=%7B%22type%22%3A%22http%22%2C%22url%22%3A%22https%3A%2F%2Fazurade.com%2Fmcp%22%7D)

Let your agent make images and videos. Azurade puts over 30 image and video models behind one MCP server (Veo 3.1, Seedance 2.5, Wan 2.7, Nano Banana Pro, GPT Image 2.5, Imagen 4 Ultra, FLUX Kontext, Seedream 4.5, Qwen Image and more) and bills them all from one credit balance.

The server is hosted at `https://azurade.com/mcp`, so there's nothing to install; this repository is documentation only.

| | |
|---|---|
| Endpoint | `https://azurade.com/mcp` |
| Transport | Streamable HTTP |
| Sign-in | OAuth 2.1 from the client, or an API key sent as `Authorization: Bearer sk-...` |
| Registry | [`com.azurade/mcp`](https://registry.modelcontextprotocol.io/v0.1/servers?search=com.azurade/mcp) in the official MCP registry |
| Pricing | Pay per generation. Credits never expire, and a failed generation is refunded automatically |

## Connect

Add the URL and the client opens an Azurade page where you sign in and click Allow. Each connection shows up on your [profile page](https://azurade.com/profile/) as a key named after the client, such as "Claude (connected app)", and deleting that key disconnects the client. Sign-up is free, with a few credits to try it.

**Claude (web and desktop).** Open Customize > Connectors, click +, choose Add custom connector and paste `https://azurade.com/mcp`. On Team and Enterprise plans an owner adds it first under Organization settings > Connectors.

**ChatGPT.** Turn on Developer mode in Settings > Security and login, press + on the Plugins page and create an app with the URL, then pick it from the Developer mode tool in a chat. Developer mode is available on Plus, Pro, Business, Enterprise and Edu, on the web.

**Claude Code.**

```sh
claude mcp add --transport http azurade https://azurade.com/mcp
```

Then run `/mcp` in a session and pick azurade to sign in.

**Codex.**

```sh
codex mcp add azurade --url https://azurade.com/mcp
codex mcp login azurade
```

**Gemini CLI.**

```sh
gemini mcp add --transport http azurade https://azurade.com/mcp
```

Then run `/mcp auth azurade` in a session.

**Cursor.** Use the Install in Cursor button above, or put this in `~/.cursor/mcp.json`:

```json
{ "mcpServers": { "azurade": { "url": "https://azurade.com/mcp" } } }
```

**VS Code.** Use the Install button above, or run MCP: Open User Configuration and add:

```json
{ "servers": { "azurade": { "type": "http", "url": "https://azurade.com/mcp" } } }
```

**A client that only speaks stdio** can go through [mcp-remote](https://www.npmjs.com/package/mcp-remote), which runs the same sign-in in your browser:

```json
{ "mcpServers": { "azurade": { "command": "npx", "args": ["-y", "mcp-remote", "https://azurade.com/mcp"] } } }
```

**With an API key.** Clients that cannot sign in send a key instead. Create one on your profile page:

```sh
claude mcp add --transport http azurade https://azurade.com/mcp --header "Authorization: Bearer sk-..."
```

An agent that has no key and no OAuth-capable client can ask for one through [auth.md](https://azurade.com/auth.md). It shows you a link and a six-digit code, you sign in and type the code, and the agent receives a key on your account. Agents never create accounts.

## Tools

| Tool | What it does |
|---|---|
| `list_models` | Every model with its type and the credit price of a default request |
| `get_model` | One model's parameters: names, defaults, allowed values |
| `estimate_generation` | The exact credit cost of a request. Free, starts nothing |
| `create_generation` | Starts an image or video and waits up to about 50 seconds for it |
| `get_generation` | Status and file links; waits again while the job runs |
| `list_generations` | Your history, newest first |
| `create_upload_url` | A one-off link the agent's shell uploads a local file to with curl |
| `enhance_prompt` | Rewrites a short prompt using the target model's prompting guide |
| `moderate_prompt` | Runs the content-policy screen on a prompt without generating |
| `get_account` | Your credit balance |
| `buy_credits` | A card checkout link for a credit pack, for you to pay when the balance runs out |

## How a generation runs

Most MCP clients give up on a tool call after 60 seconds, so `create_generation` returns after about 50. An image usually finishes inside that first call and comes back with its download link and a small preview the agent can look at. A video can take a few calls, because the first one returns `pending` with an id and `get_generation` waits another 50 seconds each time it's called.

Credits are taken when a job is accepted and refunded automatically if it fails, and an `idempotency_key` makes a retried call return the first job instead of paying twice. Input images, videos and audio are public URLs; for a file on your machine the agent calls `create_upload_url` and uploads it with curl. Every prompt is screened against the content policy.

Only you can pay. When the balance runs out, `buy_credits` hands you a checkout link with the price, you pay by card and the credits are on your balance right away. There's no developer plan either: the MCP server, the REST API and the web app spend the same balance at the same prices, listed on the [pricing page](https://azurade.com/prices/).

## Try it with curl

With an API key you can call any tool in one request. The server is stateless, so it doesn't need an `initialize` round trip first:

```sh
curl -s https://azurade.com/mcp \
  -H "Authorization: Bearer $AZURADE_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"estimate_generation","arguments":{"model":"nano-banana-pro","prompt":"a lighthouse at dusk, film photo"}}}'
```

## The same service as a REST API

| Document | |
|---|---|
| [Developer guide](https://azurade.com/developers/) | MCP setup, the REST quick start, limits |
| [OpenAPI](https://azurade.com/api/v2/openapi.json) | `/api/v2`, generated from the running code ([Swagger UI](https://azurade.com/swagger/)) |
| [Model catalogue](https://azurade.com/api/v2/models) | Every model with its fields and credit price, no key needed |
| [llms.txt](https://azurade.com/llms.txt) | The service described for language models |
| [Agent skill](https://azurade.com/.well-known/agent-skills/azurade-generation/SKILL.md) | The REST walkthrough as an agent skill |
| [Server card](https://azurade.com/mcp/server-card) | MCP server card (SEP-2127) |
| [OAuth metadata](https://azurade.com/.well-known/oauth-authorization-server) | RFC 8414; the protected resource is described at [/.well-known/oauth-protected-resource](https://azurade.com/.well-known/oauth-protected-resource) |

## Support

support@azurade.com. [Terms of use](https://azurade.com/terms-of-use/), [privacy policy](https://azurade.com/privacy-policy/).

## License

The documentation in this repository is [MIT](./LICENSE). The service itself is governed by the terms of use on azurade.com.

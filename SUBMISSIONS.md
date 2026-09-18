# Catalog submissions — owner checklist

> **INTERNAL. NOT CUSTOMER-FACING.** A working checklist for the repository
> owner. It is safe to keep in the public repository (no secrets), but nothing
> here is product copy.

Order: **1. push this repo → 2. xAI → 3. Anthropic → 4. Cursor → 5. claude.ai
connectors directory → 6. ChatGPT apps (last).**

This repository is the single canonical source every catalog points at. The
product repository is private; the generated half of this repository is written
from it by `pnpm gen:agent-skill` and pinned by the `pluginRepo` test.
**Never hand-edit `SKILL.md`, `skills/`, `server.json`, `.mcp.json` or a
manifest** — regenerate there and push here. Hand-written and safe to edit:
this file, `README.md`, `LICENSE`, `assets/`.

## 0. Before anything

- [x] Public repository created: https://github.com/fortuneflick/postdashpro-claude-plugin (MIT).
- [x] `claude plugin validate .` passes locally.
- [x] Pushed. SHA at the xAI submission: `f6132719051f1c3d5b0e46002b32a0b22a77d714`.
      Re-read it after each push with
      `git ls-remote https://github.com/fortuneflick/postdashpro-claude-plugin.git HEAD`

## Live check — the server

Ran locally against the built server on 2026-09-18, before deploy. **Re-run
against production after the deploy and replace this table with what it says.**

| Check | Result |
|---|---|
| `POST /api/mcp` with no credentials | **401**, JSON-RPC error -32001 |
| `WWW-Authenticate` on that 401 | **present** — `Bearer realm="postdashpro-mcp", resource_metadata="…/.well-known/oauth-protected-resource"` |
| `GET /.well-known/oauth-protected-resource` | **200** (also on the `/api/mcp` and `/mcp` scoped aliases) |
| `GET /.well-known/oauth-authorization-server` | **200** (also `openid-configuration`, and the scoped aliases, with the issuer derived from the path) |
| `POST /oauth/register` | **201** with a `client_id` |
| `GET /oauth/authorize` with no parameters | **400**, not 404 |
| Transport | Streamable HTTP, stateless, JSON responses (no SSE) |
| Tools | 11 |

Re-run before any submission:

```bash
curl -si -X POST https://postdashpro.com/api/mcp \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | head -20
curl -s https://postdashpro.com/.well-known/oauth-protected-resource
curl -s https://postdashpro.com/.well-known/oauth-authorization-server
curl -si -X POST https://postdashpro.com/oauth/register \
  -H 'Content-Type: application/json' \
  -d '{"client_name":"probe","redirect_uris":["https://claude.ai/api/mcp/auth_callback"]}' | head -1
```

## OAuth — built

Authentication is OAuth 2.1 (authorization code + PKCE S256), with the bearer
API key and the path-segment key kept working for clients that want them.

What ships:

1. `GET /.well-known/oauth-protected-resource` — 200, naming the resource
   (`https://postdashpro.com/api/mcp`), the authorization server and the scope.
   Served on the scoped aliases too, because different clients derive different
   candidate URLs.
2. `WWW-Authenticate: Bearer realm=…, resource_metadata=…` on every 401 from
   the MCP endpoint, so a client discovers item 1 rather than giving up.
3. `GET /.well-known/oauth-authorization-server` (and `openid-configuration`),
   RFC 7591 dynamic client registration at `POST /oauth/register` returning
   201, `GET|POST /oauth/authorize` with a consent page behind the normal
   PostDashPro sign-in, `POST /oauth/token` for the code and refresh grants
   with rotation, and `POST /oauth/revoke` (RFC 7009).
4. Tokens resolve to an API key row, so they reach exactly the account and
   surface a key reaches — and deleting that key from Settings kills them.

**Steps 5 and 6 below are unblocked by this.** Step 6 still needs the ChatGPT
domain-verification challenge route, which is a separate code change.

## The plan gate — say this in every review note

Which agent drives the server is a pricing tier, checked on `initialize` from
`clientInfo.name`:

- Creator and Growth plans: Claude and ChatGPT.
- Agency plan: Claude Code, Cursor, Codex, Grok and every other MCP client.

So a reviewer on a Creator plan testing from Claude Code gets a 402 with
`agent_client_not_on_plan`, which reads as a broken server if they were not
told. **Give every reviewer an Agency-plan test account**, and say in the notes
that the refusal is a plan message, not a fault.

## 1. Push this repository

Regenerate first, from the product repository:

```bash
pnpm gen:agent-skill            # writes into ~/Documents/postdashpro-claude-plugin
pnpm gen:agent-skill --check    # exits 1 if anything is stale
```

Then commit and push here, and record the SHA.

## 2. xAI plugin marketplace (Grok Build)

1. Fork https://github.com/xai-org/plugin-marketplace, branch `add-postdashpro`
   from upstream `main`.
2. Append the entry to `.grok-plugin/marketplace.json`, with `source.sha` set
   to the 40-char lowercase SHA of this repository's `main` (a tag or branch is
   rejected).
3. `python3 scripts/generate-plugin-index.py`, then
   `python3 scripts/validate-catalog.py` and
   `python3 scripts/generate-plugin-index.py --check` — all three must pass.
4. PR title `Add postdashpro`. Keywords and domains are brand-scoped on
   purpose; xAI rejects generic terms such as `social media`.
5. After any change to this repository, open a follow-up PR bumping `sha`.
   **Never a parallel entry.**

**Status 2026-09-18:** PR open — https://github.com/xai-org/plugin-marketplace/pull/787,
from fork `fortuneflick/plugin-marketplace`, branch `add-postdashpro`, pinned to
`f6132719051f1c3d5b0e46002b32a0b22a77d714`. Branched from upstream `main` at
1581c90; the diff is the catalog entry and its generated index rows only. Expect
the "official org vs personal account" question that the sibling submissions
drew; the answer offered there is to move the repository to a `postdashpro` org
and re-pin.

## 3. Anthropic plugin directory (Claude Code / Cowork)

- Portal: https://platform.claude.com/plugins/submit (Console; works on an
  individual account — the claude.ai-side path needs a Team/Enterprise org).
- Repository URL: `https://github.com/fortuneflick/postdashpro-claude-plugin`
- Plugin name `postdashpro` · marketplace name `postdashpro` · manifest
  `.claude-plugin/plugin.json` · category `productivity` · license MIT
- Homepage: `https://postdashpro.com/guide`
- Description: use the `description` in `.claude-plugin/plugin.json` verbatim.
- Note for the reviewer: the plugin ships the skill and a `.mcp.json` that
  reads `POSTDASHPRO_API_KEY` from the environment. No credential is written to
  a config file and none is in this repository. Claude Code is an Agency-plan
  client (see the plan gate above) — the test account must be on Agency.
- Pushes to this repository are picked up automatically. **Never open a second
  submission.**

## 4. Cursor marketplace

- Portal: https://cursor.com/marketplace/publish (clicking Submit accepts the
  Cursor Publisher Terms — owner only).
- Repository URL `https://github.com/fortuneflick/postdashpro-claude-plugin`,
  manifest `.cursor-plugin/plugin.json`, marketplace file
  `.cursor-plugin/marketplace.json`, logo
  `https://raw.githubusercontent.com/fortuneflick/postdashpro-claude-plugin/main/assets/logo.png`,
  org name `PostDashPro`, handle `postdashpro`, contact `hello@postdashpro.com`,
  website `https://postdashpro.com`.
- The Cursor plugin is skill-only; the one-click server install is the deeplink
  in README.md. Cursor is an Agency-plan client.

## 5. claude.ai connectors directory

Packet to have ready:

| Field | Value |
|---|---|
| Name | PostDashPro |
| Slug | `postdashpro` |
| Tagline (≤55 chars) | `Schedule posts to 12 networks from your agent` (45) |
| Description | The README's opening two paragraphs |
| Categories | Productivity, Marketing |
| MCP server URL | `https://postdashpro.com/api/mcp` |
| Auth | OAuth 2.1 — authorization code + PKCE, dynamic client registration. Add by URL; the directory discovers the rest. |
| Docs URL | `https://postdashpro.com/guide` |
| Privacy URL | `https://postdashpro.com/privacy-policy` |
| Terms URL | `https://postdashpro.com/terms-of-service` |
| Support | `hello@postdashpro.com` |
| Icon | `assets/icon-192.png` |
| Example prompts | "What accounts do I have connected?" · "Draft three posts for this week and schedule them for 9am" · "Make an image for this post and attach it" · "What is queued for tomorrow?" · "Write this in my brand voice and put it in drafts" |
| Test account | An Agency-plan workspace with several networks connected, a set timezone, a few items in the media library, and no MFA |

## 6. ChatGPT apps (last) — needs the domain-verification route

- Portal: https://platform.openai.com/ (OpenAI org login with Apps Management
  access and a verified publisher identity; there is no public status check).
- Requirements: a `/.well-known` challenge file served from
  `postdashpro.com` with the token the portal issues; honest tool annotations;
  a fully featured demo account **without MFA**; positive and negative test
  cases; privacy, terms and support URLs; tested in Developer Mode on desktop
  and mobile.
- **Owner steps before submitting:** the challenge route, which does not exist
  yet — a code change in the product repository, not a DNS record. OAuth itself
  is done.
- Tool descriptions are kept under 1024 characters because the Chat Completions
  tool schema rejects longer ones and drops the whole server. Keep it that way.
- Policy note for the review: PostDashPro schedules posts to accounts the user
  connected themselves. It has no follower scraping, no bulk DM and no
  unsolicited-contact tooling, and the tools never mention plans or credits.

## Not doing

- **npm.** There is no stdio launcher package; the hosted server is the only
  runtime. If one is ever published, add it to `server.json` as a `packages`
  entry alongside the existing `remotes`.
- **MCP registry (registry.modelcontextprotocol.io).** `server.json` declares
  `com.postdashpro/mcp-server`. A `com.*` namespace is proved by a DNS TXT
  record, which is an owner step:
  `_mcp-registry.postdashpro.com  TXT  v=MCPv1; k=ed25519; p=<public key>`.
  The no-DNS alternative is to rename the server to
  `io.github.fortuneflick/postdashpro-mcp-server` and prove it with
  `mcp-publisher login github`.

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
- [x] Pushed. First xAI pin was `f6132719051f1c3d5b0e46002b32a0b22a77d714`;
      PR 787 is re-pinned to `main` after every push. Re-read HEAD with
      `git ls-remote https://github.com/fortuneflick/postdashpro-claude-plugin.git HEAD`

## Live check — the server

Re-ran on 2026-09-19 after product deploy `ef83935` (Coolify
`elo5aim6o410gf7bvla7erbi` finished, container `zy38e8d-010640571124`, image
`zy38e8d:ef83935362594b229ee8f95885e9fcc1bd905e76`). A Settings-style API key
minted on the review-account row `MCP: Claude (review)` (`numan@roasbeast.com`)
got `initialize` 200 and `tools/list` 200 with 10 tools and no `generate_media`.
Browser consent was not walked.

| Check | Result |
|---|---|
| `POST /api/mcp` with no credentials | **401**, JSON-RPC error -32001 — `"Unauthorized: sign in through your AI client, or provide a valid API key"` |
| `WWW-Authenticate` on that 401 | **present** — `Bearer realm="postdashpro-mcp", resource_metadata="https://postdashpro.com/.well-known/oauth-protected-resource"` |
| `POST /api/mcp` with an unknown Bearer | **401** and the same challenge |
| `GET /.well-known/oauth-protected-resource` | **200** — resource `https://postdashpro.com/api/mcp`, scope `postdash.autopilot` |
| `GET /.well-known/oauth-authorization-server` | **200** — issuer `https://postdashpro.com`, grants `authorization_code` + `refresh_token` |
| `POST /oauth/register` | **201** with a `client_id`, `token_endpoint_auth_method: none` |
| `GET /oauth/authorize` with no parameters | **400** `unsupported_response_type`, not 404 |
| Transport | Streamable HTTP, stateless, JSON responses (no SSE) |
| Tools | 10 (as generated into this repository). `generate_media` is not on MCP — the claude.ai connectors directory requires the attestation that the server does not use AI models to generate images, video, or audio; image generation stays on the dashboard only. |
| `POST /api/mcp` tools/list with a valid API key | **200** — 10 tools (`add_media_from_url` … `wait_for_upload`, no `generate_media`) on the review account after deploy `ef83935` |
| Browser consent → token → initialize | **not verified live** — needs the claude.ai popup |

### Fix note — consent was access_denied after the trial started (2026-09-19)

Claude's directory completed OAuth consent for `numan@roasbeast.com`
(`b38973e6-96bc-44db-9bef-ab37c8546b00`) and then reported "Authorization with
the MCP server failed" twice (`ofid_662fa3bc87b837a0`, `ofid_c8e3e0d7c6190ff9`,
~00:40–00:50 UTC). Production logs on `zy38e8d-002450185937` (image `1065ee5`,
which already contained `b73de24`):

- `[trial] free trial started` at `2026-10-03T00:41:03.877Z` for that user
- the same request then logged `mcp_oauth_connection_refused` and 302'd

The trial door worked. The refusal was the Creator agent-connection cap (1).
Yesterday's Approve had minted `api_keys` `75d83173` (`MCP: Claude (1e8_DG)`);
Claude runs DCR on every attempt, so today's Approve tried to mint a second
key and `checkAutopilotLimit` redirected `access_denied`.

Fix in the product repo (`68b1982`): re-approving the same client family
reuses that leftover key; a full slot that is not this client is named on the
consent page before anything is minted. `oauth_09` reproduces the leftover-key
state and walks consent → token → initialize.

Production DB after live verify (review account):

- `users.trial_ends_at` left at `2026-10-03 00:41:03.877+00` (set by the live
  consent path; not rewritten)
- deleted leftover `api_keys` `75d83173`; inserted `e3e37665`
  `MCP: Claude (review)` so the next directory Approve reuses that family

### Fix note — Claude's issued token was 402ed (2026-09-18)

Consent completed and `/oauth/token` returned 200. Claude's three initialize
calls (`ofid_9324adc74ff8679d`, `ofid_6a42fe48bf2a19d0`, `ofid_479e97a7c1146b3a`)
were then refused. That was **402**, not 401: `mcp_oauth_tokens.last_used_at`
and the matching `api_keys.last_used_at` were stamped, so `resolve.ts` accepted
the bearer. The account (`numan@roasbeast.com`) had signed up in the OAuth
popup and never started a trial (`users.trial_ends_at` null).

Fix in the product repo (`6f99156`, on `main` as `b73de24`): approving the
connection — and the first authenticated MCP request — starts the card-free
trial with the same gates as the dashboard button. `oauth_09` walks DCR → PKCE
S256 → consent → token → initialize through `DbMcpOauthStore` on real Postgres.

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

## Access refusals — say this in every review note

Every account that can use PostDashPro may drive the server from any MCP
client. What is limited is how many agents can be connected at once — one
connection per API key, and an OAuth sign-in mints one the same way.

A reviewer who hits that ceiling, or whose account cannot use the scheduler,
gets a refusal that tells them a person enables or restores access in
PostDashPro. It does not name a tier, a price or a checkout. That still reads
as a broken server if they were not told. **Give every reviewer a test account
that already has access, with no agent connections already in use.** A lapsed
or past-due account is refused with HTTP 402, so the demo account must be in
good standing for the whole review window.

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

**Status 2026-09-19:** PR open — https://github.com/xai-org/plugin-marketplace/pull/787,
from fork `fortuneflick/plugin-marketplace`, branch `add-postdashpro`. Re-pin
`source.sha` to this repository's `main` after every push (a tag or branch is
rejected). Branched from upstream `main` at 1581c90; the live diff is the
catalog entry and its generated index row only. Expect the "official org vs
personal account" question that the sibling submissions drew; the answer
offered there is to move the repository to a `postdashpro` org and re-pin.

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
  a config file and none is in this repository. See the plan gate above — the test
  account needs spare connection headroom.
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
  in README.md, and Cursor can also sign in over OAuth with no key at all.

## 5. claude.ai connectors directory

Packet to have ready:

| Field | Value |
|---|---|
| Name | PostDashPro |
| Slug | `postdashpro` |
| Tagline (≤55 chars) | `Social post scheduling for AI agents` (36) |
| Description | The `PLUGIN_DESCRIPTION` in the product `artifacts/api-server/src/lib/pluginRepo.ts` (README lede, every manifest, `server.json`) |
| Categories (Claude directory) | Productivity, Sales And Marketing, Developer Tools |
| Categories (Cursor marketplace) | Productivity, All Automations, Canvas |
| MCP server URL | `https://postdashpro.com/api/mcp` |
| Auth | OAuth 2.1 — authorization code + PKCE, dynamic client registration. Add by URL; the directory discovers the rest. |
| Docs URL | `https://postdashpro.com/guide` |
| Privacy URL | `https://postdashpro.com/privacy-policy` |
| Terms URL | `https://postdashpro.com/terms-of-service` |
| Support | `hello@postdashpro.com` |
| Icon | `assets/icon-192.png` |
| Example prompts | "What accounts do I have connected?" · "Draft three posts for this week and schedule them for 9am" · "Attach this image from a URL" · "What is queued for tomorrow?" · "Write this in my brand voice and put it in drafts" |
| Test account | A top-plan workspace with spare connection headroom, several networks connected, a set timezone, a few items in the media library, and no MFA |

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

### ChatGPT review — test cases

Use these:

- List connections
- Schedule a draft
- Add media from a URL
- Queue a future post in the account timezone

Do not ask the reviewer to:

- Upgrade to Agency
- Post DMs
- Scrape followers
- Subscribe to post to X

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

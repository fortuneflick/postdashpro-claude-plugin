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
- [ ] Record the 40-char SHA after each push:
      `git ls-remote https://github.com/fortuneflick/postdashpro-claude-plugin.git HEAD`

## Live check — the server (2026-09-18)

| Check | Result |
|---|---|
| `POST https://postdashpro.com/api/mcp` with no credentials | **401**, `{"jsonrpc":"2.0","error":{"code":-32001,"message":"Unauthorized: provide a valid API key"}}` |
| `WWW-Authenticate` on that 401 | **absent** |
| `GET /.well-known/oauth-protected-resource` | **404** |
| `GET /.well-known/oauth-authorization-server` | **404** |
| Transport | Streamable HTTP, stateless, JSON responses (no SSE) |
| Tools | 11 |

Re-run before any submission:

```bash
curl -si -X POST https://postdashpro.com/api/mcp \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | head -20
```

## The one blocker: no OAuth

Authentication today is a bearer API key, or the key as a path segment
(`/api/mcp/k/<key>`) for connector interfaces that offer only "OAuth" or "no
authentication".

**Both the claude.ai connectors directory and the ChatGPT apps review require
working OAuth discovery.** A path-segment key is a workaround for a user adding
the server by hand, not something either review accepts: the reviewer is
checking that a new user can connect without pasting a secret into a URL.

What the server would need, in order:

1. `GET /.well-known/oauth-protected-resource` — 200, naming the resource
   (`https://postdashpro.com/api/mcp`), the authorization server and the
   scopes.
2. A `WWW-Authenticate: Bearer resource_metadata="…"` header on the 401, so a
   client discovers step 1 instead of giving up.
3. An authorization server with dynamic client registration (RFC 7591),
   PKCE, and the authorize/token endpoints under
   `/.well-known/oauth-authorization-server`.
4. Tokens that resolve to the same account a key does, so the existing
   per-account scoping holds unchanged.

Items 3 and 4 are the work; 1 and 2 are a few lines once 3 exists. **Steps 5
and 6 below stay blocked until this ships.** Steps 2, 3 and 4 do not need it.

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

## 5. claude.ai connectors directory — BLOCKED on OAuth

Packet to have ready:

| Field | Value |
|---|---|
| Name | PostDashPro |
| Slug | `postdashpro` |
| Tagline (≤55 chars) | `Schedule posts to 12 networks from your agent` (45) |
| Description | The README's opening two paragraphs |
| Categories | Productivity, Marketing |
| MCP server URL | `https://postdashpro.com/api/mcp` |
| Auth | **OAuth 2.1 — not built yet. See the blocker above.** |
| Docs URL | `https://postdashpro.com/guide` |
| Privacy URL | `https://postdashpro.com/privacy-policy` |
| Terms URL | `https://postdashpro.com/terms-of-service` |
| Support | `hello@postdashpro.com` |
| Icon | `assets/icon-192.png` |
| Example prompts | "What accounts do I have connected?" · "Draft three posts for this week and schedule them for 9am" · "Make an image for this post and attach it" · "What is queued for tomorrow?" · "Write this in my brand voice and put it in drafts" |
| Test account | An Agency-plan workspace with several networks connected, a set timezone, a few items in the media library, and no MFA |

## 6. ChatGPT apps (last) — BLOCKED on OAuth

- Portal: https://platform.openai.com/ (OpenAI org login with Apps Management
  access and a verified publisher identity; there is no public status check).
- Requirements: a `/.well-known` challenge file served from
  `postdashpro.com` with the token the portal issues; honest tool annotations;
  a fully featured demo account **without MFA**; positive and negative test
  cases; privacy, terms and support URLs; tested in Developer Mode on desktop
  and mobile.
- **Owner steps before submitting:** OAuth (above), and the challenge route,
  which does not exist yet — a code change in the product repository, not a DNS
  record.
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

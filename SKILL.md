---
name: postdashpro
description: PostDashPro is social post scheduling for AI agents. Find or attach the image, write the caption per network, preflight character limits, and queue the post in the user's timezone for them to review. Use when asked to plan, draft or schedule social posts, when a post needs a picture, or when asked what is already queued or published.
---

# Scheduling social posts as an agent

You are queueing posts that will go out under someone else's name, to their
real audience. Two rules cover most of it: **nothing you schedule publishes
instantly**, and **the clock you are reading is not theirs**.

Call `whoami` first. Reputation-affecting writes (`schedule_post`,
`update_scheduled_post`, `update_brand_voice`) are two-step: the first call
returns a preview and a `confirmToken`; the second call with that token
performs the write. Destructive tools require `confirm: true`.

## Before the first post: what is connected?

`list_connections` tells you which of the 12 networks this account has linked
and which account is active on each. Posts always go out from the active
account. Omitting `platforms` on `schedule_post` means *every connected
network*, so call this first rather than guessing what "post it everywhere"
covers.

If nothing is connected, say so and stop. Connecting an account is a person's
job in PostDashPro; no tool here does it.

## Times: never convert, never append Z

Call `get_current_time` before anything relative — "tonight", "tomorrow at
9", "in two hours". Your clock and theirs differ, and the date may already have
rolled over for one of you.

Then pass a **wall-clock time with no offset** to `schedule_post`
(`"2026-08-03T02:00"`). It is read in the user's own PostDashPro timezone. A
guessed offset is what lands a post hours from where they expected it. Repeat
back the local time the tool reports, not the one you sent.

If `get_current_time` says the timezone is not set, it is a UTC fallback and
not their real zone. Scheduling a bare wall-clock time is refused until they
set it in PostDashPro under Configuration. Tell them that instead of retrying.

## Pictures: three ways in, in this order

Instagram, TikTok, Pinterest and YouTube cannot publish text on its own, so a
post aimed at them needs media attached. `schedule_post` says so up front
rather than failing hours later at publish time.

1. `list_media` — something suitable is already in their library. Check here
   before saying you can only post text.
2. `add_media_from_url` — anything already on the public web. The URL must
   point at the file itself, not at a page that displays it.
3. `create_upload_link` — last resort. It stops the run dead until a person
   acts, so do not reach for it while they are away or have asked you to work
   on your own. Give them the link, then call `wait_for_upload` immediately:
   it returns by itself the moment the file lands, so they never have to come
   back and say "done".

**You cannot send a file yourself.** An image attached in your chat is pixels
to you, not bytes. There is no tool that forwards it — ask for a URL instead
of trying.

Pass the ids you get back to `schedule_post` as `mediaIds`, up to 10, in
order.

Call `preflight_post` before scheduling to check per-network character
limits and media rules. Then `schedule_post` once for the preview, show it,
and call again with `confirmToken`.

## Nothing publishes instantly

A future `scheduledAt` queues the post as *scheduled*. Anything else lands as
a **draft** for a person to review. There is no "post this now" tool, and that
is deliberate: the review step is the product.

So when someone says "post this", the honest answer is that you have queued it
and where they can see it — not that it went out.

`list_scheduled_posts` and `list_calendar` answer what is queued.
`list_published_posts` answers what already went out, per network.
`cancel_scheduled_post` holds a queued item as a draft.

## When you are refused

Read the error, and treat these as decisions rather than failures:

- **This client is not enabled for this account** — a person enables it in
  PostDashPro. Retrying under a different name is lying about who you are.
- **A 402** — this account cannot use the scheduler until a person restores
  access in PostDashPro. No tool here fixes it.
- **A limit** — connected accounts, brand slots or the monthly X post count.
  Waiting does not cure it. Say what the message names.

A platform that cannot take the post says so at scheduling time, not at
publish time. Fix what the message names and schedule again.

## What you cannot do here

- Publish immediately, or bypass the draft-and-review step.
- Connect or disconnect a social account.
- Send someone else's file to PostDashPro.
- Raise a limit, or change which clients the account allows.

## Tools

| Tool | What it does |
|---|---|
| `add_media_from_url` | Import media from a URL |
| `cancel_scheduled_post` | Cancel scheduled post |
| `create_idea` | Create idea |
| `create_upload_link` | Create an upload link |
| `delete_idea` | Delete idea |
| `delete_media` | Delete media |
| `get_account_settings` | Get account settings |
| `get_brand_voice` | Get brand voice |
| `get_current_time` | Get current time |
| `get_post` | Get post |
| `get_post_analytics` | Get post analytics |
| `list_calendar` | Calendar view |
| `list_connections` | List connected accounts |
| `list_ideas` | List ideas |
| `list_media` | List media library |
| `list_published_posts` | List published posts |
| `list_scheduled_posts` | List scheduled posts |
| `preflight_post` | Preflight a draft |
| `schedule_post` | Schedule post |
| `update_brand_voice` | Update brand voice |
| `update_idea` | Update idea |
| `update_scheduled_post` | Update scheduled post |
| `wait_for_upload` | Wait for the user's upload |
| `whoami` | Who am I |

Written from the live server. 24 tools.

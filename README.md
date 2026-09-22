# Journify · custom chat window + self-hosted proactive nudge

A single-file mockup of [shop.myjournify.com/support/contact-us](https://shop.myjournify.com/support/contact-us)
showing the pattern Journify asked for:

1. The **page** owns the chat window, so it can be any size it likes.
2. The **page** owns the proactive nudge bubble — it appears after the visitor has dwelled
   on the page for N seconds.
3. Accepting the nudge sets the meta field **`triggerNudge = true`** and opens the bot.

Live: **https://jiapengchua.github.io/journify-nudge-mockup/**
Bot: `journify-sandbox`

---

## The flow

```
page load          window.adaSettings = { lazy: true }     ← nothing loads, no chatter created
   │
   ├─ dwell timer (paused while the tab is hidden)
   │
   ▼
N seconds          Journify's own bubble appears           ← still no Ada call
   │
   ├── dismissed ──► Ada is never loaded at all
   │
   ▼ accepted
                   triggerNudge = true
                   adaEmbed.start({ handle, parentElement, metaFields })
                   dock opens
```

The dock is revealed **before** `start()` resolves, so the visitor gets instant feedback and
watches the bot boot inside the window rather than clicking into nothing.

## Two things Ada never does here

**The bubble is not an Ada campaign.** It is ordinary page markup on a `setInterval`.
That is the point — Journify controls the copy, the timing, the trigger conditions and the
styling, and no Ada session exists until someone clicks. A dismissed nudge costs nothing.

**The window size is not an Ada setting.** `parentElement` mounts the chat iframe inside
`#jf-chat-mount`, and the iframe fills whatever box the page gives it. The S / M / L buttons
in the dock header change two CSS variables — that is the entire resize mechanism.

```css
.jf-dock { width: var(--dock-w); height: var(--dock-h); }
#jf-chat-mount iframe { width: 100% !important; height: 100% !important; }
```

## Why `triggerNudge` goes into `start()`, not `setMetaFields()`

On a cold start the conversation is created **by** `start()`. A `setMetaFields()` call fired
immediately afterwards can land after the greeting has already been generated, so a playbook
branching on `triggerNudge` would miss it on the very turn it matters.

So the page passes the flag into `start()` when the bot is not yet running, and only falls
back to `setMetaFields()` when the bot is already up:

```js
if (!adaStarted) {
  adaEmbed.start({ handle, parentElement: 'jf-chat-mount',
                   metaFields: { triggerNudge: true } });
} else {
  adaEmbed.setMetaFields({ triggerNudge: true });
}
```

## `toggle()` and `parentElement` are mutually exclusive

Per the [SDK reference](https://docs.ada.cx/chat/web/sdk-api-reference#toggle):
*"You cannot use this method with the `parentElement` option."*

There is no drawer to toggle — the chat is inline in our own element — so the page opens and
closes its own container with a CSS class. The conversation stays alive while the dock is
closed; only `reset()` or `stop()` tear it down.

The **Ada drawer** option in the demo panel switches to the stock drawer and drives it with
`toggle()` instead, so you can compare the two side by side. The size presets do nothing in
that mode, which is exactly the limitation that motivated the custom dock.

## Demo controls

Bottom-left **⚙ Demo controls** (delete the `jf-dev` block for production):

| Control | What it does |
| --- | --- |
| Widget mode | Custom dock (`parentElement`) vs Ada drawer (`toggle()`) |
| Dwell before nudge | Seconds of *visible* time before the bubble fires. Demo default 20s; real sites use 120–300s |
| Meta fields | Live readout of what has been sent to Ada |
| SDK call log | Every `adaEmbed.*` call and its result, timestamped |
| Show nudge now / Restart timer | Skip or restart the wait |
| Reset conversation | `reset({ metaFields: { triggerNudge: false } })` — new chatter |
| Stop & unload | `stop()` — removes Embed2 entirely, back to a clean page |

## Wiring the Ada side

The page only *sends* the flag. To make the agent behave differently on a nudge-initiated
chat, create a variable named exactly `triggerNudge` on `journify-sandbox` — meta fields
populate the matching variable — and branch on it. A nudged visitor arrived from a support
form they had not finished, so the useful opening is context-aware
("Still working on that form? …") rather than the generic greeting.

Without that, `triggerNudge` is still recorded and visible in the **Meta variables** panel on
each conversation, which is enough to measure nudge-attributed chats.

## Before it will run

**The origin serving this page must be on the bot's iframe allow list** — Ada CSP-gates the
chat iframe. Add `https://jiapengchua.github.io` under Settings → Security on
`journify-sandbox`, **with no trailing slash** (a trailing `/` silently kills the frame).

If the chat does not come up within 12s the dock says so and names the origin to allow-list,
rather than spinning forever.

`file://` will not work. Serve over HTTPS, or `python3 -m http.server` for local work
(localhost also has to be allow-listed).

## Files

```
index.html                    the whole mockup — page, nudge, dock, demo panel
assets/journify-logo.png      real asset from shop.myjournify.com
assets/contact-banner.png     real asset from shop.myjournify.com
```

## Status

Verified: page layout against the live original, dwell timer, bubble show/dismiss, the
`triggerNudge` → `start()` call sequence, dock open/close, size presets, and the
allow-list failure path. The chat rendering *inside* the dock has not been verified — the
Ada embed does not complete its handshake in headless Chrome, so that needs one pass in a
real browser once the origin is allow-listed.

Unofficial mockup, built for an Ada demo. Not affiliated with Journify or Malaysia Aviation Group.

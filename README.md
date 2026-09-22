# Journify · custom chat widget + idle nudge

A one-page mockup based on
[shop.myjournify.com/support/contact-us](https://shop.myjournify.com/support/contact-us),
stripped to the page shell. Ada's chat is mounted inside **Journify's own 400×620 window**,
and when the visitor goes idle the page shows **its own nudge bubble**. Accepting it sets
the meta field `triggerNudge = true` and opens the window.

Live: **https://jiapengchua.github.io/journify-nudge-mockup/** · Bot: `journify-sandbox`

## The integration

```js
// parentElement MUST be declared here, not passed to start().
window.adaSettings = {
  handle: "journify-sandbox",
  parentElement: document.getElementById("chat-mount"),
  adaReadyCallback: armIdleTimer
};

function openChat(viaNudge) {
  dock.classList.add("open");                                  // our own window
  if (viaNudge) window.adaEmbed.setMetaFields({ triggerNudge: true });
}
```

The page owns the launcher, the bubble, the window and its size. Ada's iframe just fills
whatever box it is given:

```css
#dock { width: 400px; height: 620px; }
#chat-mount iframe { width: 100% !important; height: 100% !important; }
```

## `parentElement` mechanics, measured

| how it is passed | what happens |
| --- | --- |
| in `adaSettings`, no `lazy` | ✅ mounts into your element, no default button |
| in `start()` | ❌ **never resolves**, nothing mounts, no error |
| `lazy` + in `adaSettings` + `start({handle})` | ❌ `parentElement` ignored — you get the default button |

`start()` options **replace** `adaSettings` rather than merging, and a `parentElement` handed
to `start()` hangs, so `lazy` and `parentElement` cannot be combined. The chat therefore
initialises on page load — which is why the docs advise against putting `parentElement` on
every page. The window stays hidden behind a CSS class until the visitor wants it.

`toggle()` is also unavailable in this mode, so the page shows and hides `#dock` itself. The
conversation survives a close; only `reset()` or `stop()` end it.

Ada's own proactive campaigns cannot be used with a custom window either: `triggerProactive`
has no button to anchor its teaser to and no drawer to open, so it resolves successfully and
does nothing. That is why the nudge here is the page's own and the agent is told about it
through a meta field.

## Idle, not elapsed time

Any of mousemove / mousedown / keydown / scroll / wheel / touchstart restarts the clock, and
a backgrounded tab accrues no idle time — so this fires on *"stopped, and probably unsure"*
rather than *"has been here 20 seconds"*. `IDLE_MS` is 20s for demo purposes; a real page
would use 45–90s.

## Wiring the Ada side

The page only *sends* the flag. Create a variable named exactly `triggerNudge` on
`journify-sandbox` — meta fields populate the matching variable — and branch on it. A nudged
visitor did not come looking for chat; the page interrupted them, so that earns a different
opening from someone who clicked the launcher themselves.

Either way the flag is recorded and shows in the **Meta variables** panel on each
conversation, which is enough to measure nudge-attributed chats.

## Before it will run

The origin must be on the bot's iframe allow list (Settings → Security).
`https://jiapengchua.github.io/` is on it — the trailing slash is fine.
`http://localhost:3000` is too, which is the quickest way to work on this locally.

## Files

```
index.html                    the whole mockup
assets/journify-logo.png      real asset from shop.myjournify.com
assets/contact-banner.png     real asset from shop.myjournify.com
```

## Status

Verified against the live bot in a driven browser session: the chat mounts inside the custom
window, the nudge appears only after real idle, clicking it opens the window and
`getMetaFields()` returns `{"triggerNudge": true}`.

The banner is Journify's real image and still reads "…in the contact form below", which no
longer matches now that the form is gone.

Unofficial mockup, built for an Ada demo. Not affiliated with Journify or Malaysia Aviation Group.

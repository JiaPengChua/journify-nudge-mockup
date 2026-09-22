# Journify · custom chat widget + idle nudge

A one-page mockup based on
[shop.myjournify.com/support/contact-us](https://shop.myjournify.com/support/contact-us),
stripped to the page shell. Ada's chat is mounted inside **Journify's own 400×620 window**,
and when the visitor goes idle the page shows **its own nudge bubble**. Accepting it sets
`triggerNudge = true` and opens the window.

Live: **https://jiapengchua.github.io/journify-nudge-mockup/** · Bot: `journify-sandbox`

## Does `triggerProactive` work with a custom widget? No.

Tested directly against the live bot, both ways:

| | default Ada widget | custom widget (`parentElement`) |
| --- | --- | --- |
| `triggerProactive` renders | `#ada-intro-frame` teaser beside Ada's button | **nothing** |
| Proactive text reaches the transcript | yes, when the visitor clicks the teaser | **no** — transcript is the standard greeting |
| Return value | Promise resolving `null` | Promise resolving `null` — resolves either way, so it looks like it worked |

It needs Ada's own button to anchor the teaser to and its own drawer to open, and
`parentElement` removes both. It fails silently: the call resolves, and nothing happens.

Worth knowing even on the default widget: opening the drawer yourself with `toggle()` after
firing a proactive **discards it** — the visitor gets the normal greeting. Only a click on
Ada's teaser carries the message into the conversation.

So with a custom window the nudge has to be the page's own, and the way to tell the agent
about it is a meta field.

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

The page owns the launcher, the window and its size; Ada's iframe just fills the box.

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

So `lazy` and `parentElement` cannot be combined: `start()` options replace `adaSettings`
rather than merging, and a `parentElement` handed to `start()` hangs. The consequence is that
the chat initialises on page load, which is why the docs advise against putting
`parentElement` on every page. The window stays hidden behind a CSS class until wanted.

`toggle()` is also unavailable in this mode, so the page shows and hides `#dock` itself. The
conversation survives a close; only `reset()` or `stop()` end it.

## Idle, not elapsed time

Any of mousemove / mousedown / keydown / scroll / wheel / touchstart restarts the clock, and
a backgrounded tab accrues no idle time — so this fires on *"stopped, and probably unsure"*
rather than *"has been here 20 seconds"*. `IDLE_MS` is 20s for demo purposes; a real page
would use 45–90s.

## Wiring the Ada side

Create a variable named exactly `triggerNudge` on `journify-sandbox` — meta fields populate
the matching variable — and branch on it. The `idlepdp` proactive's copy
("Hi There! Need here to find the perfect flypass?") is a good model for what that branch
should open with, since the proactive itself cannot be used here.

The flag is recorded either way and shows in the **Meta variables** panel on each
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

# Journify · custom chat widget + idle nudge

A one-page mockup based on
[shop.myjournify.com/support/contact-us](https://shop.myjournify.com/support/contact-us),
stripped to the page shell. Nothing Ada-related loads until the visitor asks for it. When
they go idle the page shows **its own nudge bubble**; accepting it starts the conversation
with `triggerNudge = true` already set, and opens a **400×620 window**.

Live: **https://jiapengchua.github.io/journify-nudge-mockup/** · Bot: `journify-sandbox`

## The integration

```js
// Nothing loads until openChat(): no chatter, and the conversation-start
// playbook does not run until the visitor actually opens the chat.
window.adaSettings = { lazy: true };

function openChat(viaNudge) {
  if (started) {
    if (viaNudge) window.adaEmbed.setMetaFields({ triggerNudge: true });
    return window.adaEmbed.toggle();
  }
  started = true;
  // start() is what creates the conversation, so meta fields passed here are on
  // the chatter *before* the conversation-start playbook runs.
  window.adaEmbed.start({
    handle: "journify-sandbox",
    metaFields: { triggerNudge: !!viaNudge },
    adaReadyCallback: function () { window.adaEmbed.toggle(); },
    toggleCallback: function (isOpen) { launcher.classList.toggle("open", isOpen); }
  });
}
```

## Custom window size without `parentElement`

Ada's drawer is a `position: fixed` iframe on the host page, so the page can just restyle it.
This keeps `lazy` and `toggle()`, which `parentElement` would take away:

```css
#ada-button-frame { display: none !important; }   /* we supply our own launcher */

#ada-chat-frame {
  width: 400px !important;  max-width: 400px !important;
  height: 620px !important; max-height: calc(100vh - 110px) !important;
  right: 24px !important;   bottom: 94px !important;
}
```

`max-width` is the one that catches you out: Embed2 sets `max-width: 375px` **inline**, so
overriding `width` alone silently clamps back to 375px. A stylesheet `!important` beats an
inline declaration, so both are needed.

## Why not `parentElement`

`parentElement` also gives a custom size, but it forces the conversation to start on page
load — which is exactly the problem this build exists to avoid. Measured:

| how it is passed | what happens |
| --- | --- |
| in `adaSettings`, no `lazy` | mounts into your element, **but the chat initialises immediately** |
| in `start()` | **never resolves**, nothing mounts, no error |
| `lazy` + in `adaSettings` + `start({handle})` | `parentElement` silently ignored |

`start()` options **replace** `adaSettings` rather than merging, and a `parentElement` handed
to `start()` hangs, so `lazy` and `parentElement` cannot be combined. It also removes the
default launcher and disables `toggle()`.

Ada's own proactive campaigns are unusable with `parentElement` too: `triggerProactive` has
no button to anchor its teaser to and no drawer to open, so it resolves successfully and does
nothing. The CSS approach above keeps the real drawer, so proactives would work here — but
this page uses the meta field, because the nudge is the page's own.

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

Verified against the live bot in a driven browser session: **no Ada DOM node exists before
the click** (no chatter, no playbook run), the nudge appears only after real idle, clicking
it opens a 400×620 window, and `getMetaFields()` returns `{"triggerNudge": true}`.

The banner is Journify's real image and still reads "…in the contact form below", which no
longer matches now that the form is gone.

Unofficial mockup, built for an Ada demo. Not affiliated with Journify or Malaysia Aviation Group.

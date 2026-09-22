# Journify · proactive nudge → Ada

A one-page mockup based on
[shop.myjournify.com/support/contact-us](https://shop.myjournify.com/support/contact-us),
stripped to the page shell so nothing competes with the thing being shown:

1. Journify renders **its own nudge bubble** after the visitor has been on the page a while.
2. Clicking it sets the meta field **`triggerNudge = true`**.
3. Then it opens the bot.

Live: **https://jiapengchua.github.io/journify-nudge-mockup/**
Bot: `journify-sandbox`

## The whole integration

```js
window.adaSettings = { handle: "journify-sandbox" };

// 1. nudge after the visitor has been on the page a while
setTimeout(function () { nudge.classList.add("show"); }, DWELL_MS);

// 2. accepted -> set the meta field, then open the bot
nudge.onclick = function () {
  nudge.classList.remove("show");
  window.adaEmbed.setMetaFields({ triggerNudge: true })
    .then(function () { window.adaEmbed.toggle(); });
};
```

That is all of it. The bubble is ordinary page markup on a `setTimeout` — Ada knows nothing
about it, so Journify owns the copy, the timing and the trigger conditions. Dismissing it
does nothing at all.

`toggle()` opens Ada's standard chat window, and Ada's standard chat button is on the page
from the start, so a visitor who wants chat before the nudge fires can still get it.

## Wiring the Ada side

The page only *sends* the flag. To make the agent behave differently on a nudge-initiated
chat, create a variable named exactly `triggerNudge` on `journify-sandbox` — meta fields
populate the matching variable — and branch on it. A nudged visitor did not come looking for
chat; the page interrupted them. That earns a different opening from someone who clicked the
chat button themselves.

Either way the flag is recorded and visible in the **Meta variables** panel on each
conversation, which is enough to measure nudge-attributed chats.

## Before it will run

**The origin serving this page must be on the bot's iframe allow list** — Ada CSP-gates the
chat iframe. Add `https://jiapengchua.github.io` under Settings → Security on
`journify-sandbox`, **with no trailing slash** (a trailing `/` silently kills the frame).

`file://` will not work. Serve over HTTPS, or `python3 -m http.server` locally — localhost
has to be allow-listed too.

## If you want a custom-sized chat window

This version uses Ada's standard drawer. To size the window yourself, mount it into your own
element with [`parentElement`](https://docs.ada.cx/chat/web/sdk-api-reference#parentelement)
and give that element whatever dimensions you like. Two caveats that come with it:
`toggle()` does not work in that mode (there is no drawer — you show and hide your own
container), and no default chat button is rendered, so the page has to supply one.

## Files

```
index.html                    the whole mockup
assets/journify-logo.png      real asset from shop.myjournify.com
assets/contact-banner.png     real asset from shop.myjournify.com
```

## Status

Verified: the page renders, the dwell timer fires, the bubble shows and dismisses. The
`setMetaFields` → `toggle` pair has not been seen running against the live bot — the Ada
embed does not complete its handshake in headless Chrome, so that needs one pass in a real
browser once the origin is allow-listed.

The banner is Journify's real image and still reads "…in the contact form below", which no
longer matches now that the form is gone.

Unofficial mockup, built for an Ada demo. Not affiliated with Journify or Malaysia Aviation Group.

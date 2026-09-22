# Journify · proactive nudge → Ada

A one-page mockup based on
[shop.myjournify.com/support/contact-us](https://shop.myjournify.com/support/contact-us),
stripped to the page shell. When the visitor **goes idle**, the page fires an Ada
**proactive** and Ada takes it from there.

Live: **https://jiapengchua.github.io/journify-nudge-mockup/**
Bot: `journify-sandbox` · Proactive: `idlepdp`

## The whole integration

```js
window.adaSettings = {
  handle: "journify-sandbox",
  adaReadyCallback: armIdleTimer
};

function armIdleTimer() {
  var timer, fired = false;

  function fire() {
    if (fired) return;
    fired = true;
    window.adaEmbed.triggerProactive({ messageKey: "idlepdp" });
  }

  function reset() {
    if (fired) return;
    clearTimeout(timer);
    if (!document.hidden) timer = setTimeout(fire, IDLE_MS);
  }

  ["mousemove", "mousedown", "keydown", "scroll", "wheel", "touchstart"]
    .forEach(function (evt) { document.addEventListener(evt, reset, { passive: true }); });
  document.addEventListener("visibilitychange", reset);

  reset();
}
```

Ada draws its own teaser bubble next to the chat button, and clicking it opens the chat with
the proactive message already at the top of the transcript.

## Idle, not elapsed time

Any interaction restarts the clock, and a backgrounded tab accrues no idle time, so this
fires on *"stopped, and probably unsure"* rather than *"has been here 20 seconds"*. Someone
reading steadily down the page is never interrupted.

Verified both ways: sit still past the threshold and the teaser appears; keep a mousemove
going every 2s against an 8s threshold and it stays away for as long as the activity lasts,
then appears once the visitor goes quiet.

**The campaign does not fire itself.** `idlepdp` has `url_trigger_conditions: []`, so Ada
never shows it on its own — it appears only because the page calls `triggerProactive`. The
name describes the intent; the page supplies the idle detection.

## What testing this turned up

**`messageKey` is the campaign's `key`, not the id in the dashboard URL.** The URL
`/proactives/6a0d1c47e8764eb2ae1bd606` is the id; the key is `idlepdp`. Passing the id
silently does nothing. `adacli journify-sandbox proactives get <id>` prints both.

**Call it after `adaReadyCallback`.** Fired on a bare `setTimeout` from page load it can beat
the embed's boot, and then nothing renders and nothing errors. This was the one real bug
found while wiring it up.

**Do not call `toggle()` afterwards.** Opening the drawer programmatically throws the
proactive away — the visitor gets the standard greeting and the proactive text never appears.
Verified: open via `toggle()` → transcript is the normal 4-message greeting; open by clicking
the teaser → the proactive is the first message, greeting follows.

**Do not draw your own bubble as well.** `triggerProactive` renders Ada's own
`#ada-intro-frame`. A page-drawn bubble on top of it means two bubbles.

**It returns a Promise**, despite the reference giving the signature as `void`. It resolves
`null` once the teaser is up.

## Proactive vs. the meta-field approach

| | `triggerProactive` | `setMetaFields({ triggerNudge: true })` |
| --- | --- | --- |
| Who draws the bubble | Ada | your page |
| Who owns the copy | the Ada dashboard | your page |
| Opens the chat | visitor clicks the teaser | your page calls `toggle()` |
| Agent knows why | the message is in the transcript | branch on a variable |

Use the proactive when the nudge is a message. Use the meta field when you need the page's
own bubble — a different position, your own styling, or conditions Ada cannot express — and
want the agent to *behave* differently rather than say something specific. The meta-field
version is left commented in `index.html`.

## Before it will run

The origin serving the page must be on the bot's iframe allow list (Settings → Security on
`journify-sandbox`). `https://jiapengchua.github.io/` is on it. `http://localhost:3000` is
too, which is the easiest way to work on this locally.

Contrary to what I said earlier in this repo's history, the **trailing slash is fine** —
`https://jiapengchua.github.io/` is on the list with one and the widget loads.

## Files

```
index.html                    the whole mockup
assets/journify-logo.png      real asset from shop.myjournify.com
assets/contact-banner.png     real asset from shop.myjournify.com
```

## Status

Verified end to end against the live bot in a real browser session: page renders, the
proactive fires after the dwell, Ada's teaser appears with the dashboard copy
("Hi There! Need here to find the perfect flypass?"), and clicking it opens the chat with
that message first in the transcript.

The banner is Journify's real image and still reads "…in the contact form below", which no
longer matches now that the form is gone.

Unofficial mockup, built for an Ada demo. Not affiliated with Journify or Malaysia Aviation Group.

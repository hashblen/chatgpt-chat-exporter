# v1.1.1 — exports now work on guest (not-signed-in) ChatGPT pages

## What was broken

chatgpt.com's **guest** transcript — the page you get before signing in at
`chatgpt.com` — is rendered by a client different from the account version:

- turns are `<ol data-conversation-transcript>` → `<li
  data-message-role="user|assistant">`, with **no** `data-message-author-role`,
  no `conversation-turn-*` `data-testid`s, and no `data-message-id` — none of
  the selectors that identified a ChatGPT message;
- the assistant's answer is a `<div data-assistant-markdown>`;
- the user's typed prompt is a `<p data-user-message-copy>` nested **inside a
  clickable `<button data-user-message-bubble>`** — exactly the shape the
  exporter strips as UI chrome, so serializing the turn was guaranteed to lose
  the question;
- the sr-only turn label is `<h4 class="_wdUoQG_srOnly">`, which the legacy
  `[class*="sr-only"]` strips do not match (case-sensitive `srOnly` vs
  `sr-only`).

A guest-page export read **no messages at all** ("No messages found"), while
the same conversation signed-in exported fine — which is what made the failure
look like a page problem rather than an exporter one.

## The fix

The ChatGPT adapter now knows the guest shape too:

- `li[data-message-role]` joins the turn and message selector cascades;
- `[data-user-message-copy], [data-assistant-markdown]` join the content roots
  (page CSS sets no `white-space: pre-wrap` on the assistant copy, so markdown
  formatting survives untouched);
- `identifySender` reads `data-message-role`;
- a `<button>` that owns a `[data-user-message-copy]` descendant is treated as
  content, not chrome;
- guest-style `<li>` turns pick their nested copy node directly as the content
  root, so action bars, avatar chrome, and sr-only labels are never serialized
  in the first place.

The cascade is additive: the guest selectors match only the guest renderer, so
signed-in ChatGPT and Gemini selectors are untouched. Everything else — the
DOM sweep, the streaming wait, HTML/PDF output, the userscripts — is unchanged.

## Notes and limitations

- Guest pages render citation pills as `<button>`s rather than `<a>` links, so
  References lists stay empty there; they work as before when signed in.
- Signed-out engagements never take the private-API payload path, so every
  guest export reads the DOM (with the completeness caveat that applies to any
  swept export).
- The selector-doctor's drifted-page regression test now expects the fallback
  to be named at its new 1-indexed cascade position, because the guest selector
  sits one entry earlier in the list.

## Verification

Regression tests cover the new shape end to end against a live-observed
fixture: both turns found with reliable senders, clean markdown serialization,
no UI/chrome/sr-only leakage, and a `complete` full-path export. The full
jsdom suite (106 tests) passes alongside.
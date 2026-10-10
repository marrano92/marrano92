---
title: "Building a Safari-style Reader Mode as a Chrome Extension (MV3) with DeepL Translation"
description: "How I built Reader Mode AI: article extraction, shadow DOM rendering, a translation pipeline that never breaks images, and what the Chrome Web Store taught me about permissions."
tags: chromeextension, javascript, webdev, tutorial
published: false
---

# Building a Safari-style Reader Mode as a Chrome Extension (MV3) with DeepL Translation

I read a lot of long-form articles, half of them in other languages. Safari Reader is
beautiful — and nothing in the Chrome world combined a calm reading view with in-place
translation. So I built it: **Reader Mode AI**, shipped on the Chrome Web Store and fully
open source. Here is the engineering behind it, and the mistakes worth avoiding.

## What it does

- Turns any article into a distraction-free reading view (centered column, serif, themes)
- Translates the text in place into 5 languages via the DeepL API — **images, video and
  charts keep their position and are never sent to the provider**
- Caches every translation locally: reopening a translated article costs **zero API calls**

## 1. Extracting the article: Mozilla Readability plus a safety net

Extraction *is* the product: a reader lives or dies on how well it picks the article out of
navigation, ads and sidebars. I used [Mozilla Readability](https://github.com/mozilla/readability)
(the engine behind Firefox Reader View) and added three things on top:

```js
// Keep every video/embed provider: media must survive extraction.
const READABILITY_OPTIONS = { allowedVideoRegex: /.*/i };

// A listing page is not an article: require real prose (2 paragraphs of 120+ chars).
if (!hasEnoughProse(content)) return null;
```

1. **A prose threshold.** Readability happily "finds" articles on product listing pages.
   Rejecting results without at least two paragraphs of real prose fixed that.
2. **Title cleanup.** Extracted titles often drag the site name along
   ("Headline — The Daily Chronicle"). If the title starts with the in-article `<h1>`, use
   the `<h1>`.
3. **A heuristic fallback** that picks the paragraph-densest container and strips known noise
   for the pages where Readability gives up.

## 2. Rendering: shadow DOM, not iframes

The reader renders over the page in a **shadow root**, so no page stylesheet can leak in —
and none of ours leaks out. What we keep from the extracted content is a deliberate choice:
media stays (decision: *never translate images away*), active content goes.

```js
for (const el of fragment.querySelectorAll("script, style, link, meta, base, noscript")) el.remove();
for (const el of fragment.querySelectorAll("*")) {
  for (const attr of [...el.attributes]) {
    if (/^on/i.test(attr.name)) el.removeAttribute(attr.name);
    if (/^(href|src)$/i.test(attr.name) && /^\s*javascript:/i.test(attr.value)) el.removeAttribute(attr.name);
  }
}
```

Plus a small Apple-like touch: the view fades in while the toolbar, header and content rise
into place with a staggered ease-out — and the whole thing is skipped under
`prefers-reduced-motion`.

## 3. MV3: the service worker that dies

Manifest V3 terminates your service worker whenever it feels like it. The fix is
architectural, not a trick: **no long-lived state**. A translation is a sequence of
independent, one-request-per-chunk messages from the content script; if the worker dies
between chunks, the next message simply starts it again. The translation cache lives in
`chrome.storage.local` (keyed by an FNV-1a hash of each string plus the target language),
so a re-read is free — and a per-article memory (`url → language`) auto-applies the
translation when the reader reopens. Zero API calls, zero clicks.

## 4. Translating without breaking images

The naive approach — translate `innerHTML` — loses images or sends them to the provider.
Instead, each block is split into **text runs around media nodes**:

```text
<p>before <strong>bold</strong> <img src="chart.png"> after</p>
        └──── run 1 ────┘   (not sent)  └run 2┘
```

Media nodes never leave the browser. The text runs are sent as HTML fragments
(`tag_handling=html` preserves inline markup), and each translated run is swapped back into
the exact node range it came from — the `<img>` keeps its identity and position. Full text
replacement, structure intact.

## 5. Testing an extension like real software

- **Unit**: extraction, chunking and the pipeline run against fixture pages — including a
  fully styled fake news site with display ads and a cookie banner, asserting the ads never
  reach the reader.
- **End-to-end**: Playwright loads the built extension in a persistent Chromium context with
  `--load-extension`, and a **local stub of the DeepL API** answers translation calls (the
  endpoint is configurable) — the suite proves the whole flow including the cache-hit path
  with **zero real API calls**.

## 6. Shipping: what the Chrome Web Store taught me

- **Generic host permissions slow you down.** A content script on `<all_urls>` triggers an
  "in-depth review" warning. The fix is `activeTab` + `chrome.scripting`: inject on user
  gesture only. Same UX, faster review, and a much less scary install prompt.
- **The privacy form is the real test.** Single purpose, remote code, per-permission
  justifications, data transmission to DeepL — write them plainly and honestly.
- **Trademarks live in your listing copy.** "Safari-inspired" is fine in a README; in the
  store listing it is a review risk. We rebranded the listing to "elegant, distraction-free
  reader mode".

## Takeaways

1. The extraction quality *is* the product; invest there first.
2. Design for a service worker that dies and nothing surprises you.
3. Cache aggressively: "already translated, zero calls" is a feature users feel.
4. Ship with a stub server: your e2e suite can cover the whole AI flow without spending a
  cent or leaking a byte.

The full source is on GitHub: [marrano92/reader-mode-ai](https://github.com/marrano92/reader-mode-ai).
Feedback and issues welcome — that is how open source works.

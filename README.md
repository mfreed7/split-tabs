# Opening links in a split tab

Authors: [Mason Freed](https://github.com/mfreed7) (@mfreed7)

Based on, and with credit to, the original explainer
["Allow target navigation at split tab"](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/SplitTab/NavigationInSplitTab/explainer.md)
by [XU Zhengyi](mailto:zhengyixu@microsoft.com) (Microsoft). This document
re-works that proposal, most significantly by moving the opt-in from the
`target` parameter to the `windowFeatures` parameter of `window.open()`.

## Status of this Document

This document is a starting point for engaging the community and standards
bodies in developing collaborative solutions fit for standardization. As the
solutions to problems described in this document progress along the
standards-track, we will retain this document as an archive and use this
section to keep the community up-to-date with the most current standards venue
and content location of future work and discussions.

* This document status: **Active**
* Expected venue: [W3C Web Incubator Community Group](https://wicg.io/)
* **Current version: this document**

## Introduction

Split screen scenarios are increasingly common. In addition to dual screen
devices (i.e. physical split screen), browsers including Chrome, Edge, Arc,
Whale, and Vivaldi ship split tab (i.e. virtual split screen) features, where
two tabs are tiled side-by-side within a single browser window. When using a
split tab, there are multiple cases in which users want to open a link and view
it side-by-side with the source page (see [Use Cases](#use-cases)).

Today there is no way for a web author to request that a navigation open into a
split tab. The existing navigation targets (`_blank`, `_self`, `_parent`,
`_top`) cannot express this, and `windowFeatures` only offers `popup`, which
opens a separate, detached browser window. Without `popup`, `window.open()`
opens the content as a new full-size tab in the current window, which is the
only other option available.

This explainer proposes a way for web authors to *request* that a link open in
a split tab, alongside the current page.

## Goals

- Provide an API that allows websites to request that a navigation open in a
  split tab, alongside the page initiating the navigation.
- Allow the site to express a preference for basic layout aspects of the
  resulting split (which side the new content appears on, and roughly how much
  space it occupies).
- Degrade gracefully (to an ordinary new tab) when the user agent does not
  support split tabs, or when the user agent decides a split is not
  appropriate.

## Non-Goals

- Physical split screen / multi-monitor placement. The
  [Window Management API](https://developer.mozilla.org/en-US/docs/Web/API/Window_Management_API)
  is the appropriate tool for those more complicated cases: it exposes screen
  details and allows precise placement of windows across multiple displays,
  behind a permission prompt. This proposal is deliberately a low-friction,
  permission-free hint for the common "show this next to me" case, within a
  single browser window.
- Guaranteeing a particular window arrangement. Everything here is a request;
  the user agent remains in control.
- Reading back the resulting layout, or detecting whether the request was
  honored. See [Other things considered](#other-things-considered).

## Use Cases

Any site that wants a link to be viewed next to its source page:

- On a search engine results page (SERP), the user can open results in a split
  tab without navigating away from the SERP.
  ([example image from the original explainer](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/SplitTab/NavigationInSplitTab/split_tab_example.png))
- An in-page agent or assistant can open a page it is working with in a split
  tab and remain visible alongside it — e.g. the agent's UI on the right, the
  page it is driving on the left — so the user can watch and intervene instead
  of losing the agent behind a tab switch. Because the opener relationship is
  preserved (see [Behavior](#behavior)), the two views can coordinate with
  `postMessage` and the agent retains a handle to navigate or close the view it
  opened. Anything deeper — observing or automating a *cross-origin* page — is
  out of scope here and would need a separate API; this proposal covers
  placement, not inspection.
- A design, layout, or coding tool can present its source/editor pane in one
  split and a live preview of the result in the other, as two real tabs rather
  than an `iframe`. Unlike an `iframe`, the preview gets a full top-level
  navigable: its own viewport size, top-level-only APIs, and no ancestor
  styling or embedding restrictions. The editor keeps a handle to the preview,
  so it can push updates to it as the source changes.
- On a video site, the user can watch a video in a split tab while continuing
  to browse other results in the source page.
- On a news feed, social media, or shopping site, the user can open an
  article/post/product in a split tab and keep browsing the feed or list.
- On an office automation (OA) system or email site, the user can preview
  external document links in a split tab.
- Documentation sites can open a linked reference or sample next to the tutorial
  the user is reading.

## Proposed Solution

Add a set of features to the `windowFeatures` (third) parameter of
[`window.open()`](https://html.spec.whatwg.org/multipage/nav-history-apis.html#dom-open-dev).

The table below lists the complete set of window features after this proposal.
The three rows marked **New** are the additions; the rest are the existing
mechanisms, included for context. Note the symmetry: `popup` selects a detached
window and is configured by `width`/`height`/`left`/`top`, while `split`
selects a split tab and is configured by `splitposition`/`splitsize`.

| Feature | Value | Meaning | New? |
| --- | --- | --- | --- |
| *(none)* | — | Default. With an empty or absent feature string, the content opens as a new full-size tab in the current window. | |
| `popup` | boolean | Open the content in a separate, detached browser window ("popup") instead of a tab. | |
| `width` / `innerWidth` | integer CSS pixels | Requested viewport width of the popup. Only applies when a popup is requested. | |
| `height` / `innerHeight` | integer CSS pixels | Requested viewport height of the popup. Only applies when a popup is requested. | |
| `left` / `screenX` | integer CSS pixels | Requested screen x-position of the popup. Only applies when a popup is requested. | |
| `top` / `screenY` | integer CSS pixels | Requested screen y-position of the popup. Only applies when a popup is requested. | |
| `noopener` | boolean | The new navigable has no opener, and `window.open()` returns `null`. | |
| `noreferrer` | boolean | Omit the `Referer` header; also implies `noopener`. | |
| `location`, `toolbar`, `menubar`, `resizable`, `scrollbars`, `status` | boolean | Legacy, and ignored as such. They survive only as inputs to [checking if a popup window is requested](https://html.spec.whatwg.org/multipage/nav-history-apis.html#popup-window-is-requested). | |
| `split` | boolean | Request that the new navigable be opened as a split tab, alongside the opener. | **New** |
| `splitposition` | `left`, `right`, `top`, `bottom` | Which side of the current tab the new content should appear on. Optional; the user agent picks a default (likely `right`, or the inline-end side). Only applies when a split is requested. | **New** |
| `splitsize` | percentage, e.g. `30` | The share of the split, in percent, to give to the **new** content. The current page gets the remainder. Optional; the user agent picks a default (likely `50`). Only applies when a split is requested. | **New** |

`split` is deliberately parallel to `popup`: it is a boolean window feature
requesting a particular kind of "container" for the new content, and it is
mutually exclusive with `popup`.

### Examples

```js
// Basic: request a split tab.
window.open("https://www.example.com", "_blank", "split");

// Open on the left, taking 30% of the width.
window.open("https://www.example.com", "_blank",
            "split,splitposition=left,splitsize=30");

// Stacked split, new content on the bottom.
window.open("https://www.example.com", "_blank",
            "split,splitposition=bottom");

// The opener relationship is intact, as with any window.open() call, so the
// two views can coordinate. This is what the agent and live-preview use cases
// are built on.
const preview = window.open("/preview", "_blank", "split,splitsize=50");
preview.postMessage({source}, origin);

// Opt out of the opener relationship explicitly, if desired.
window.open("https://www.example.com", "_blank", "split,noopener");
```

### Behavior

- **Opt-in only, and user-gesture gated.** Split tab requests are subject to
  the same popup-blocking rules as any other `window.open()` call, i.e. they
  generally require [transient activation](https://html.spec.whatwg.org/multipage/interaction.html#transient-activation).
- **The opener is preserved**, as with any other `window.open()` call. A split
  tab is *not* forced to `noopener`: `window.opener` is set, and
  `window.open()` returns a window handle, just as it does for a popup or a new
  tab today. Authors who want the two pages disconnected write
  `"split,noopener"`, and sites can protect themselves with
  [`Cross-Origin-Opener-Policy`](https://html.spec.whatwg.org/multipage/browsers.html#cross-origin-opener-policies).
  - Forcing `noopener` would remove a capability the same call already has.
    `window.open(url, "_blank")` hands back a working window handle today, with
    no permission and no opt-in. Showing that content in a split instead of a
    tab does not change what the handle can do.
  - It would break the best use cases. The agent and live-preview scenarios
    need the handle. It also would not stop same-origin pages from talking, as
    they can use `BroadcastChannel` regardless — so it mostly just blocks the
    cross-origin `postMessage` case.
  - Side-by-side content does raise a spoofing question, addressed in
    [Privacy and Security Considerations](#privacy-and-security-considerations).
    The answer there is browser UI, not an API restriction.
- **Mutually exclusive with `popup`.** Because both features describe the
  container for the new content, specifying both is an authoring error.
  Proposed resolution: `split` wins, and the `popup` request is ignored.
- **Requests, not commands.** The user agent may ignore `split`,
  `splitposition`, and/or `splitsize` entirely — for example on a narrow
  window, on mobile, when the window is already split, or based on user
  settings. It may also clamp `splitsize` to a usable range.
- **Fallback.** When a split is not created, the navigation opens as an
  ordinary new tab (not a popup window, and not a new browser window). The one
  exception: if the author specified both `split` and `popup`, and the split is
  not created, the fallback is a popup, since that is what the author asked for
  otherwise.
- **Named targets still win.** As today, if the `target` parameter names an
  existing navigable, that navigable is navigated and the window features are
  ignored. Authors should use `_blank`.
- **`splitsize` parsing.** Parsed as an integer percentage; invalid or
  out-of-range values are ignored (treated as "user agent default"), consistent
  with how other window features tolerate garbage input.

### Spec sketch

The changes are confined to the
[window open steps](https://html.spec.whatwg.org/multipage/nav-history-apis.html#window-open-steps)
and
[the rules for choosing a navigable](https://html.spec.whatwg.org/multipage/document-sequences.html#the-rules-for-choosing-a-navigable);
no new navigable target names are introduced.

1. In the window open steps, after
   [tokenizing](https://html.spec.whatwg.org/multipage/nav-history-apis.html#concept-window-open-features-tokenize)
   *features*, let *splitRequested* be the result of
   [checking if a window feature is set](https://html.spec.whatwg.org/multipage/nav-history-apis.html#window-feature-is-set)
   given *tokenizedFeatures*, "`split`", and false.
2. If *splitRequested* is true and the user agent supports and chooses to
   create a split tab, then:
   - Set *windowType* to "`split tab`", or "`split tab with no opener`" if
     *noopener* was set by the `noopener` or `noreferrer` features.
   - Let *splitPosition* be *tokenizedFeatures*["`splitposition`"] if it is one
     of "`left`", "`right`", "`top`", "`bottom`"; otherwise a user-agent
     defined default.
   - Let *splitSize* be the result of parsing
     *tokenizedFeatures*["`splitsize`"] as an integer percentage; if that fails
     or is out of range, a user-agent defined default.
   - These values are passed to the user agent as hints when creating the new
     [top-level traversable](https://html.spec.whatwg.org/multipage/document-sequences.html#top-level-traversable).
3. [Check if a popup window is requested](https://html.spec.whatwg.org/multipage/nav-history-apis.html#popup-window-is-requested)
   returns false if a split tab was created in step 2. This matters for two
   reasons:
   - It makes `split` and `popup` mutually exclusive in a well-defined way,
     while still letting `popup` apply if the split is not created.
   - Without it, `split` would be an "unrecognized" non-empty feature string
     and would therefore trigger popup behavior — which is exactly the
     legacy-browser hazard described below.

### Legacy fallback hazard (and mitigation)

In browsers that do not implement this proposal, `"split"` is simply an
unrecognized feature name, but the feature string is *non-empty*, so
[checking if a popup window is requested](https://html.spec.whatwg.org/multipage/nav-history-apis.html#popup-window-is-requested)
returns **true**. In other words, on today's browsers
`window.open(url, "_blank", "split")` opens a **popup window**, which is not
the desired fallback.

Authors can opt out of that explicitly, because an explicit `popup=0` short-
circuits the popup check:

```js
// Split tab where supported; ordinary tab everywhere else.
window.open("https://www.example.com", "_blank", "popup=0,split");
```

Whether this ergonomic wart is acceptable, or whether the boolean should be
spelled in a way that avoids it, is an [open question](#open-questions). It is
worth noting that the equivalent problem in the `target`-based proposal (an
unexpected opener, or navigating a same-named frame) is harder to work around,
since it has no `popup=0`-style escape hatch.

### Pros

- `split` sits naturally next to `popup`: both describe *how* the new content
  is presented, and the mutual exclusivity is obvious from the syntax.
- No new reserved `target` keyword, so there is no risk of colliding with
  existing frames named `_split`, and no interaction with
  [valid navigable target names](https://html.spec.whatwg.org/multipage/document-sequences.html#valid-navigable-target-name).
- The `windowFeatures` string is naturally extensible, so `splitposition` and
  `splitsize` (and future hints) fit without new syntax.
- Unrecognized features are ignored by design, so the additional hints degrade
  gracefully on their own.

### Cons

- `windowFeatures` is only available via `window.open()`, so this proposal has
  no declarative (HTML) form. See [Open Questions](#open-questions).
- The legacy popup fallback described above requires `popup=0` to be written
  defensively.
- `windowFeatures` is an unstructured, legacy-flavored string format. It is not
  the format anyone would choose today, but it is the format that already
  exists for exactly this purpose.

## Alternatives Considered

### A new `target` keyword (`_split`) — the original proposal

The [original explainer](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/SplitTab/NavigationInSplitTab/explainer.md)
proposed a new navigable target name, `_split`, usable both as
`<a target="_split">` and as `window.open(url, "_split")`, falling back to
`_blank` when split tabs are unsupported.

**Pros**
- Works declaratively in HTML, with no JavaScript required.
- A natural extension of an attribute authors already understand.

**Cons**
- Target names are not extensible: there is no natural way to express
  "left/right" or a split percentage.
- `_split` and `_blank`/`_self` are mutually exclusive, but that is implicit; a
  window feature makes the exclusivity with `popup` explicit.
- On unsupported browsers, `_split` is treated as an ordinary frame name. If
  the page already has a frame named `_split`, that frame is navigated instead.
  ([Names beginning with `_` are reserved](https://html.spec.whatwg.org/multipage/document-sequences.html#valid-navigable-target-name),
  but content in the wild may not respect that.)
- Opener behavior would differ between supported and unsupported browsers. The
  original proposal forced `noopener` for `_split`, but an unsupported browser
  treating `_split` as an ordinary name does not, so the same markup would
  silently produce an opener on some browsers and not others. The
  `windowFeatures` form has no such split-brain: the opener behaves the same
  everywhere, and is controlled by the existing `noopener` feature.

### A new content attribute on `<a>`

Add e.g. `<a href="..." split>`. This has no effect on unsupported browsers,
which is a nice property, but it leaves the relationship between `target` and
the new attribute unspecified (what does `<a target="_self" split>` mean?), and
it still needs a separate mechanism for the additional hints. A declarative
form is probably still desirable eventually; see
[Open Questions](#open-questions).

### The Window Management API

[Window Management](https://developer.mozilla.org/en-US/docs/Web/API/Window_Management_API)
already solves the general "place this content somewhere specific" problem, and
is the right answer for multi-monitor and precise placement scenarios. It is
not a good fit here because:

- It requires the `window-management` permission, which is a heavy prompt for
  "open this link next to me".
- It positions *browser windows* on *screens*; it has no concept of a split tab
  inside a single browser window, which is the user-visible feature being
  targeted here (including its tab-strip grouping, resize affordance, and
  lifetime).
- It is not available in all browsers, and is deliberately gated because it
  exposes substantial screen information.

## Other things considered

### Feature detection (`navigator.splitTabSupported`)

The original proposal included a `navigator.splitTabSupported` boolean so that
authors could branch on split tab availability. That is omitted here, because:

- It adds a fingerprinting surface: whether a split tab is available is a
  function of browser, browser version, window size, form factor, and possibly
  user settings — all correlated bits that are otherwise not exposed.
- It is unreliable in practice. Support is dynamic (a window that is too narrow
  now may be wide enough later), so a boolean read at page load is easily
  stale.
- The request/fallback model above means authors mostly do not need it: the
  call is safe to make unconditionally, and falls back to a normal tab.

If a compelling use case for detection appears, options that leak less include
a CSS media feature, or reporting only after a user gesture. Notably, the
value would still be a UA hint rather than a guarantee.

### Reporting the actual result

An API to explicitly report whether the split was created, or what size it
ended up at, is intentionally not proposed, for the same fingerprinting reasons
as above. Note that this is only a soft guarantee now that the opener is
preserved: a page holding a window handle can infer a little about the result,
just as it can for a popup today. That is no worse than `window.open()` today,
and not worth breaking the main use cases to prevent.

### Closing or resizing the split afterwards

Out of scope. Once opened, the split is under user and user-agent control. The
opener does keep the ordinary window handle abilities — `close()`, `focus()`,
and navigation — but none of those are split-specific.

### Forcing `noopener`

Considered and rejected; see [Behavior](#behavior).

## Privacy and Security Considerations

### Privacy

- No new information is exposed to the page. The page cannot learn whether a
  split tab was created, what size it is, or whether the feature is supported.
  This is a deliberate change from the original proposal, which exposed
  `navigator.splitTabSupported`.
- Preserving the opener does not change this. A handle to a cross-origin window
  exposes nothing readable beyond what `window.open()` already exposes.

### Security

- **No new capability.** A split tab is reached through `window.open()`, and
  grants exactly what `window.open()` grants today: an opener relationship
  unless `noopener`/`noreferrer` is specified, subject to the usual user
  activation and popup blocking rules. The only new ingredient is where the
  content is painted.
- **Adjacent-pane spoofing is the one genuinely new consideration.** Because
  the two views are visible simultaneously and one can navigate the other (in
  either direction, via the opener relationship), an attacker-controlled pane
  can change what is displayed immediately next to a victim pane. Two
  aggravating factors:
  - Browser UI may show the URL only for the *focused* pane, so a navigation of
    the unfocused pane can be less visible than the equivalent navigation of a
    foreground tab.
  - Combined with `splitsize`, a page could request a small adjacent pane that
    reads as part of its own UI.

  The mitigation is browser UI rather than an API restriction: user agents
  should make the origin of *each* pane clear, and should make the boundary
  between panes visually unambiguous. This is required regardless of this
  proposal, since a user can already place two unrelated tabs in a split
  manually, and reverse tabnabbing via `window.open()` already exists.
- **`splitsize` should be clamped** by the user agent. A page should not be able
  to request a 1% sliver, or a 99% split that effectively hides the opener.
- **Implementation note (not security):** keeping the opener means the two
  views stay related, so same-site panes may end up sharing a renderer process,
  and therefore a main thread. Both panes are visible at once in a split, so
  one janking the other is more noticeable than for a background tab. That is
  work for implementers, not a reason to restrict the API — and sites that want
  the separation can ask for it with `Cross-Origin-Opener-Policy` or
  `noopener`.

## Open Questions

- **Naming.** `split` / `splitposition` / `splitsize` are placeholders.
  Alternatives: `splittab`, `splitside`, `splitratio`, `splitwidth`.
- **`popup=0` ergonomics.** Should the spec do something to make the legacy
  fallback less sharp, or is documenting `"popup=0,split"` sufficient?
- **Precedence.** If both `popup` and `split` are specified, which wins? This
  document proposes `split`.
- **Opener controls.** The opener is preserved by default (see
  [Behavior](#behavior)). Are the existing controls — `noopener` for the
  opener's choice, `Cross-Origin-Opener-Policy` for the openee's — sufficient,
  or is there a case for split-specific severance? Relatedly, should a split
  *opened by* a cross-origin navigation chain behave any differently?
- **Declarative form.** Should there be an HTML equivalent, and if so, what?
  Note that a declarative form may be important for the SERP use case, where
  links are plain anchors.
- **Position values.** Are `top`/`bottom` worth specifying, given that not all
  browsers support stacked splits? Should the values instead be logical
  (`inline-start` / `inline-end` / `block-start` / `block-end`) so they respect
  writing mode and directionality?
- **Sandboxed iframes.** Should split tab requests be restricted inside
  sandboxed iframes, and if so, does that need a new sandbox token
  (e.g. `allow-split-tab-navigation`), or is
  `allow-popups` sufficient?
- **Existing splits.** What should happen if the opener is already in a split?
  Replace the other side, create a new split elsewhere, or fall back to a new
  tab?

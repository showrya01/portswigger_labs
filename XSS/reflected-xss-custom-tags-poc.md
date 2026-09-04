# PoC: Reflected XSS into HTML Context With All Tags Blocked Except Custom Ones

**Lab:** PortSwigger Web Security Academy
**Category:** Reflected Cross-Site Scripting (XSS)
**Status:** Solved ✅

---

## 1. Vulnerability Overview

The application reflects the `search` GET parameter directly into the page's HTML body:

```html
<h1>0 search results for '<search-term>'</h1>
```

A server-side filter blocks known/standard HTML tag names (`<script>`, `<img>`, `<svg>`, `<body>`, etc.) and returns an error for them. However, the filter only checks against a **blocklist of known tag names** — it does not block arbitrary/custom (non-standard) tag names. Since the browser still parses HTML attributes on unrecognized tags, event-handler attributes like `onfocus` remain functional even inside a custom tag.

---

## 2. Reconnaissance Steps

### Step 1 — Confirm reflection point
Submitted a baseline value in the search box and observed it reflected unescaped inside an `<h1>`:

```
GET /?search=<test> HTTP/2
```

Response:
```html
<h1>0 search results for '<test>'</h1>
```

### Step 2 — Test standard tags
Submitting recognized tags (e.g. `<script>`, `<img>`) returned a JSON error from the server:

```json
"Tag is not allowed"
```

This confirmed a **tag-name blocklist/allowlist filter** was in place, not a full HTML sanitizer.

### Step 3 — Test a custom (non-standard) tag
```
GET /?search=<test onload="alert(document.cookie)"> HTTP/2
```

Response (reflected, unescaped, and unfiltered):
```html
<h1>0 search results for '<test onload="alert(document.cookie)">'</h1>
```

This proved:
- Custom/unknown tag names bypass the filter entirely.
- Event-handler attributes on the custom tag are preserved as-is.

> Note: `onload` does not fire on arbitrary custom elements (it's not a recognized lifecycle event for them), so this confirmed *injection* but not yet *execution*. The next step was to find an event that reliably fires.

---

## 3. Building a Working Trigger

`onfocus` fires on **any** element that can receive focus — including custom tags — *if* that element is actually given focus. Two things are required:

1. Make the custom tag focusable: add `tabindex="1"`.
2. Force the browser to focus that specific element automatically, with no user interaction.

Simply adding `autofocus` to a custom tag is unreliable across browsers/specs for non-standard elements. The reliable technique (and PortSwigger's intended solution) is to:

- Host an **iframe** on the exploit server.
- Point the iframe's `src` at the vulnerable URL with the payload, appended with a **URL fragment (`#id`)** matching the injected element's `id`.
- In the iframe's `onload` handler, call `this.contentWindow.focus()`.

When the iframe navigates to a URL ending in `#id`, the browser scrolls to and can be forced to focus that element; calling `.focus()` on the iframe's `contentWindow` completes the trigger, firing `onfocus` on the injected tag — with zero clicks from the victim.

---

## 4. Final Payload

Injected parameter value:
```html
<test onfocus=alert(document.cookie) id=x tabindex=1>
```

URL-encoded, targeting the vulnerable endpoint with a fragment identifier:
```
https://YOUR-LAB-ID.web-security-academy.net/?search=%3Ctest%20onfocus%3Dalert(document.cookie)%20id%3Dx%20tabindex%3D1%3E#x
```

---

## 5. Delivery Exploit (hosted on exploit server)

```html
<iframe
  src="https://YOUR-LAB-ID.web-security-academy.net/?search=%3Ctest%20onfocus%3Dalert(document.cookie)%20id%3Dx%20tabindex%3D1%3E#x"
  onload="this.style.width='100px';this.contentWindow.focus()">
</iframe>
```

**How it works end-to-end:**
1. Victim loads the exploit-server page containing the iframe.
2. The iframe requests the vulnerable search page with the malicious `<test>` payload and a `#x` fragment.
3. The reflected response renders `<test onfocus=alert(document.cookie) id=x tabindex=1>` in the page — the custom tag bypasses the tag-name filter.
4. On `iframe.onload`, JavaScript calls `contentWindow.focus()`, which (combined with the `#x` fragment targeting the element with `id=x`) shifts focus onto the injected element.
5. Focusing the element fires `onfocus`, executing `alert(document.cookie)` — arbitrary JavaScript execution in the victim's session, satisfying the lab objective.

---

## 6. Steps to Reproduce (Exploit Server)

1. Go to **Go to exploit server**.
2. Paste the HTML from Section 5 into the **Body** field (update `YOUR-LAB-ID`).
3. Click **Store**.
4. Click **View exploit** to confirm the alert fires against yourself.
5. Click **Deliver exploit to victim** — the lab is marked solved once the victim's simulated browser executes the payload.

---

## 7. Root Cause & Fix Recommendations

- **Root cause:** The server sanitizer uses a denylist of known HTML tag names rather than a proper HTML parser/allowlist of safe elements and attributes. Unknown tags pass through unmodified, along with all their attributes (including event handlers).
- **Fix:**
  - Contextually encode user input when reflecting it into HTML (`&lt;`, `&gt;`, `&quot;`, etc.) rather than filtering tag names.
  - If tag filtering is required, use an allowlist of both tag names **and** attributes, and strip all `on*` event-handler attributes regardless of tag name.
  - Apply a strict Content-Security-Policy (CSP) disallowing inline event handlers (`script-src` without `'unsafe-inline'`).

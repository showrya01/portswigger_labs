# PoC: Reflected XSS into a JavaScript String With Angle Brackets and Double Quotes HTML-Encoded and Single Quotes Escaped

**Lab:** PortSwigger — Reflected XSS into a JavaScript string with angle brackets and double quotes HTML-encoded and single quotes escaped
**Status:** Solved ✅

## Vulnerability

The search term is reflected inside a JavaScript string:

```js
var searchTerms = 'hello';
```

## Steps

1. **Test all special characters** — searched `hello"hello/hello'`, and viewed the source:
   ```js
   var searchTerms = 'hello&quot;hello/hello\'';
   ```
   This showed `<`/`>`/`"` get HTML-encoded and `'` gets backslash-escaped to `\'`. Tag-breakout and simple quote-breakout are both blocked.

2. **Spot the escaping bug** — the app escapes `'` by prepending a backslash, but it doesn't escape a backslash that's *already* in the input. So sending a literal `\` right before the `'` causes the app to add its own `\`, producing `\\'` — two backslashes followed by a quote.

   In JavaScript, `\\` is just an escaped literal backslash inside the string — it doesn't escape the character after it. So the `'` that follows is no longer escaped and closes the string early.

3. **Build the payload**:
   ```
   \'-alert(1)//
   ```
   - `\` — makes the app's auto-escaping produce `\\` (a literal backslash, not an escape).
   - `'` — now closes the string for real.
   - `-alert(1)` — valid JS on its own; the leading `-` turns it into a harmless expression statement.
   - `//` — comments out the rest of the original line (the trailing `'` and everything after).

4. **Result** — reflected as:
   ```js
   var searchTerms = '\\'-alert(1)//';
   ```
   Which JS parses as: string `'\\'` (a single backslash), followed by `-alert(1)`, with `//'` commented out. `alert(1)` executes.

## Payload

```
https://YOUR-LAB-ID.web-security-academy.net/?search=\'-alert(1)//
```

URL-encoded:
```
https://YOUR-LAB-ID.web-security-academy.net/?search=%5C'-alert(1)%2F%2F
```

## Fix

Escaping `'` alone isn't enough — the backslash used as the escape character must itself be escaped first (`\` → `\\`) before escaping quotes, otherwise attacker-supplied backslashes can neutralize the escaping. Proper JSON-style string encoding (escaping `\` before `'`/`"`) or moving user input out of inline `<script>` blocks entirely (e.g. via a `data-*` attribute read by JS) avoids this class of bug.

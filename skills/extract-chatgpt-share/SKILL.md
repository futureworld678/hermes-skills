---
name: extract-chatgpt-share
description: Extract conversation text from a ChatGPT share link (chatgpt.com/s/...) without a browser or login. The share page is server-rendered with react-router streaming, so the conversation content is embedded in script tags as escape-encoded strings. Use when the user sends a ChatGPT share URL and asks you to read/summarize/process the conversation.
---

# Extracting ChatGPT Share Link Content

When the user sends a `https://chatgpt.com/s/t_<token>` URL and wants the content, **you usually cannot just curl it and `grep` the body** — the visible HTML is a React shell with ~580 chars of placeholder text. The actual conversation is serialized into a `window.__reactRouterContext.streamController.enqueue("...")` call as a JSON-encoded string.

## Quick recipe (works as of 2026-04)

```bash
# 1. Fetch the page
curl -sL -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 \
(KHTML, like Gecko) Chrome/120.0 Safari/537.36" \
  "https://chatgpt.com/s/t_XXXXXXXXXXXXXXXX" -o /tmp/cgpt_share.html
```

Check the title to confirm it loaded:
```bash
grep -oP '(?<=<title>)[^<]+' /tmp/cgpt_share.html | head -1
# Should print the conversation title, not "ChatGPT"
```

## 2. Extract Chinese/CJK content (simplest path when content is Chinese)

A surprisingly reliable shortcut: just grep the raw HTML for CJK runs with surrounding punctuation. Content strings end up inline and deduplicated.

```python
import re
from pathlib import Path
raw = Path('/tmp/cgpt_share.html').read_text()
runs = re.findall(
    r'[\u4e00-\u9fff][\u4e00-\u9fff\s\w，。、！？；：""''（）【】《》\-.,;:()\[\]]{10,}',
    raw,
)
# Many will be duplicates; dedupe preserving order
seen = set()
uniq = [x for x in runs if not (x in seen or seen.add(x))]
print('\n'.join(uniq))
```

Then manually concatenate the longest unique run — it's usually the full assistant reply.

## 3. Proper extraction (for English or mixed content)

The heavy script tag is indexed by character length. Find it:

```python
import re
from pathlib import Path
raw = Path('/tmp/cgpt_share.html').read_text()
scripts = re.findall(r'<script[^>]*>(.*?)</script>', raw, re.DOTALL)
# The streaming payload is a large script containing `enqueue("...")` calls.
# In 2026-04 it's usually index 7 of ~15 scripts. Don't hardcode — find by content:
candidates = [s for s in scripts if 'streamController.enqueue' in s]
payload_script = candidates[0] if candidates else None
```

The enqueued string is double-escaped (JS string inside HTML attribute). Extract and decode:

```python
import re, json
m = re.search(r'enqueue\("(.*?)"\)', payload_script, re.DOTALL)
payload = m.group(1)
# First decode: JS \\-escapes back to \x, \u etc
try:
    decoded = json.loads('"' + payload + '"')  # re-wrap so json.loads treats it as string
except json.JSONDecodeError:
    decoded = payload.encode().decode('unicode_escape')
```

The `decoded` structure is react-router's streamed format: a giant array where strings and objects are referenced by numeric IDs like `{"_5":6,"_7":8}`. **Reconstructing the full tree is complex and fragile** — don't bother unless you need precise role/ordering. For most use cases, regex-extracting long strings out of `decoded` is enough:

```python
long_strs = re.findall(r'"((?:[^"\\]|\\.){50,})"', decoded)
messages = []
for s in long_strs:
    try:
        messages.append(json.loads('"' + s + '"'))
    except json.JSONDecodeError:
        messages.append(s)
```

## Pitfalls

1. **Don't waste time with `browser_navigate` first** unless Chrome is actually installed. `agent-browser install` downloads a full Chromium — heavy. Curl + regex works for 90% of cases.

2. **`__NEXT_DATA__` doesn't exist** on modern chatgpt.com share pages (they migrated to react-router). Don't search for it.

3. **User-Agent matters.** A bare `curl` (no UA) sometimes gets a different bot page. Use a real Chrome UA.

4. **The page is not JS-rendered for bots** — the content IS in the HTML. Just encoded in streaming chunks. Don't assume you need a headless browser.

5. **Login-gated conversations** (starred/private) return a login page. The URL format `/s/t_...` is for public shares; these never require login. If you see `"authStatus":"logged_out"` AND no CJK/long text runs, the share may have been deleted or set to private.

6. **Anti-bot** — chatgpt.com occasionally Cloudflare-challenges. If curl returns HTML with "Just a moment..." or < 50KB with no content, wait a minute and retry. As of 2026-04 this is rare for share pages.

7. **HTML size check**: a real share page is typically 200KB-500KB. Anything under 50KB is almost certainly an error page or CF challenge.

## Verification checklist

After extraction, verify:
- Title from `<title>` tag matches what the user expects
- Extracted text length is reasonable (share pages are usually 2-20KB of actual content)
- Speaker turns make sense (look for `"author":{"role":"user"}` and `"role":"assistant"` markers in the decoded stream if you need attribution)

## When to fall back to a browser

Only use `browser_navigate` if:
- Text extraction returned garbage/too little
- You need to capture rendered media (screenshots, images)
- The share URL redirects (rare — might be a new format the regex doesn't handle)

In that case, `agent-browser install` first, then navigate and use `browser_snapshot(full=true)`.

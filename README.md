# LLM Output Sink Scanner

Paste a model response and see every URL your renderer would silently GET, every cell a spreadsheet would run as a formula, and every terminal escape that overwrites what you read.

**Live demo:** https://0xelitesystem.github.io/llm-output-sink-scanner/

## Use

1. Paste the raw model response (assistant message, tool result or agent transcript) into the first box, or press **Load sample**.
2. Under **Declare the sinks**, tick where that text goes: rendered as HTML, written to CSV or a spreadsheet, or printed to a terminal. Optionally list allowlisted hosts for the render sink.
3. Press **Scan output**.
4. Read the findings, grouped by sink.

## Why this exists

LLM output gets rendered, exported and printed by code that trusts it. A markdown image can send data to an outside host the moment a chat bubble renders, a cell that starts with `=` can run as a formula, and a terminal escape can rewrite a line you already read. This tool checks a response against those sinks before your app passes it on. It is one HTML file that runs in your browser, with no tracking and no server, under the MIT license.

## Features

The input is a **model response**: the assistant message, tool result or agent transcript your
application just received. You declare where that text goes, and you get per-sink proof.

- **Render sink (the hero).** Finds the URLs a browser fetches the moment the message appears, with
  no click and no hover: markdown images, `img src` and `srcset`, `source srcset`, `video poster`,
  `object data`, `embed src`, `input type=image`, `iframe src` and `srcdoc`, `link rel=preload`,
  `stylesheet`, `icon`, `shortcut icon`, `prefetch`, `prerender`, `modulepreload` and `manifest`,
  SVG `image href`, `xlink:href` and external `use`, the legacy `background` attribute, `url()` in a
  style attribute, and `url()`, `@import` and `@font-face src` in a style element. Two rel values
  that are widely assumed to fetch were measured and did not: `apple-touch-icon`, and `preload` with
  no `as`. Both are reported without an auto-fetch claim rather than dropped or overstated.
- **Reference-style markdown is resolved, not regexed.** Inline, full reference, collapsed and
  shortcut forms are all resolved against link reference definitions collected from the *whole*
  document, so a definition sixty lines below its use is still found. A filter that inspects one
  line at a time is exactly what this walks past.
- **CSV and spreadsheet sink.** Flags cells that begin with any of the seven ASCII characters OWASP
  names, not just `=`: equals, plus, minus, at, tab (0x09), carriage return (0x0D) and line feed
  (0x0A). The same OWASP page also lists the full-width variants of the four formula characters,
  which some locales treat as formula starters, so those four are checked too, for eleven in total.
  Checked three ways: the whole response as one cell, each line as a cell, and each comma-delimited
  field. A cell that is a plain number after its sign, such as `-1` or `+1,234.50`, is still reported
  because the OWASP rule names the character, but at low severity: a finance export is full of those
  and calling them attacks is how a scanner gets ignored.
- **Terminal sink.** Finds OSC 8 hyperlinks whose visible label is unrelated to the URI they open,
  OSC 52 writes to your clipboard, OSC 0 and OSC 2 window-title changes, and the cursor-movement and
  erase sequences that let agent output overwrite a line a human already read. It reconstructs what
  a terminal would actually display and shows it beside the raw text.
- **An allowlist that refuses to print a green check.** An allowlisted host is reported as permitted,
  never as contained. If the URL carries another URL inside it, which is the shape of a preview,
  unfurl, embed or proxy endpoint, the finding escalates instead of clearing.
- **False-positive discipline.** An image inside a fenced code block, an indented code block, a code
  span or an HTML comment is not a finding. Flagging it teaches you to ignore the tool.
- **Zero requests.** Not "we try not to". The mechanism is described below and was measured.

## How it works

**The zero-request guarantee is the entire pitch, and the obvious implementation breaks it.** MDN
states plainly, on `DOMParser.parseFromString()`, that "while the document can download resources
specified in `<iframe>` and `<img>` elements, it is essentially inert". That is precisely the
construct this tool exists to detect, and the same hazard is documented for `template.innerHTML` and
`createHTMLDocument`. So no parser is ever handed the raw text.

The pipeline runs in this order:

1. **Mask code regions.** Fenced blocks, indented blocks, code spans and HTML comments are blanked
   out with spaces. Every offset is preserved, so reported line numbers stay true.
2. **Neutralize the string.** One pass over the raw string rewrites `src`, `srcset`, `srcdoc`,
   `href`, `xlink:href`, `data`, `poster`, `background`, `style`, `action`, `formaction`, `ping`,
   `cite`, `manifest`, `lowsrc`, `dynsrc`, `imagesrcset`, `longdesc`, `usemap`, `codebase`, `archive`
   and `profile` to inert `data-sink-*` names, and renames `style`, `link`, `base` and `meta`
   elements to inert custom elements. The attribute separator class is space, tab, line feed, form
   feed, carriage return **and slash**, because all six were measured to work in a real browser.
3. **Only then parse.** The neutralized string is parsed by a pure tokenizer that needs no DOM, plus
   an optional cross-check against the browser's own HTML parser on that same neutralized string.
4. **Resolve markdown structure.** Escaping HTML is not a fix, because a markdown image contains no
   HTML. Structure is resolved to the elements a renderer would produce, then analysed.
5. **Render results as text nodes only.** Found URLs go into the page with `textContent`. The
   stylesheet contains no `url()` token anywhere, so no rule applied to a finding can become a
   request either.

### What was measured, and how

The enumeration of fetching constructs was not written from memory. A local HTTP server logged every
path it was asked for, one page carried each construct pointing at a unique path, and the page was
loaded once in a real browser: **Microsoft Edge 151.0.4129.72 (Chromium) on Windows 10, measured
2026-08-09**. The tool ships both tables, the constructs that fired and the constructs that did not,
with that date stamp, kept visually separate from the live findings so a row going stale can never
make the scanner's logic look wrong.

The neutralizer was then tested against the same list: every construct that fired was run through it
and the result was inserted into a live, attached `div` in a real page. Zero requests.

Two results changed the answer:

- **External SVG `use` fired the request.** It is widely repeated that external references in `use`
  are same-document-only in Chromium and WebKit. Whatever is true of *rendering*, the external file
  was requested. For an exfiltration sink only the request matters, because the data has already
  left by the time the renderer decides whether to draw anything. MDN separately notes that browsers
  may apply the same-origin policy to `use` and refuse to load a cross-origin URL, which governs
  using the result, not whether the GET happens.
- **Detached nodes still fetch.** The safe-looking implementation for a tool like this is "parse it
  into a `div` I never insert, then walk the DOM". Measured: a detached `div` assigned hostile
  `innerHTML` fired the request, and so did an `img` built with `createElement` and `setAttribute`.
  Being out of the document is not inertness. In the same run, `DOMParser`, `template.innerHTML` and
  `createHTMLDocument` did *not* fire, which is the opposite of what MDN warns is permitted. Both can
  be true at once: documentation says what is allowed, a measurement says what one engine did on one
  day. The neutralize-first design means the tool does not depend on either.

### Case receipt, attached to the render sink only

CVE-2025-32711 is a 2025 CVE, published 2025-06-11, described in the authoritative record as "Ai
command injection in M365 Copilot allows an unauthorized attacker to disclose information over a
network", classified CWE-74 and scored CVSS 3.1 base 9.3, Critical.

Two of the defences this tool checks are the two the reporting researchers walked through. Aim Labs
documented that Copilot redacted external markdown links from the chat history, and that
reference-style markdown links and images were not redacted. They then documented that an `img-src`
content security policy allowlist blocked their own domain, so they used an allowlisted Microsoft
first-party endpoint that performs a fetch on the caller's behalf. That is the whole argument for
refusing to print a green check on an allowlisted host: the allowlist was correct and enforced, and
it was still not a boundary.

What is deliberately **not** claimed is what the fix contained. Microsoft never published it. The
MSRC advisory answers "why are there no links to an update" with "This vulnerability has already
been fully mitigated by Microsoft. There is no action for users of this service to take. The purpose
of this CVE is to provide further transparency." So the only defensible statement is which defences
failed, not which defence replaced them.

This receipt belongs to the render sink. The CSV sink cites OWASP and the terminal sink cites the
XTerm control sequence reference and the iTerm2 escape code documentation. Neither inherits the CVE.

## Verified against

Every external claim in this repo comes from a source that was fetched while building it, on
2026-08-09.

- CVE record for CVE-2025-32711: https://www.cve.org/CVERecord?id=CVE-2025-32711
- MSRC advisory for CVE-2025-32711: https://msrc.microsoft.com/update-guide/vulnerability/CVE-2025-32711
- Aim Labs EchoLeak write-up: https://www.aim.security/lp/aim-labs-echoleak-blogpost
  (checked 2026-08-09: that address now returns a permanent redirect away from the write-up, so the
  quotations were read from the archived 2025-06-16 snapshot, which still serves the original page:
  https://web.archive.org/web/20250616001121/https://www.aim.security/lp/aim-labs-echoleak-blogpost )
- OWASP CSV Injection: https://owasp.org/www-community/attacks/CSV_Injection
- CommonMark Spec 0.31.2: https://spec.commonmark.org/0.31.2/
- XTerm Control Sequences: https://invisible-island.net/xterm/ctlseqs/ctlseqs.html
- iTerm2 escape codes, documenting the OSC 8 hyperlink syntax: https://iterm2.com/documentation-escape-codes.html
- MDN, DOMParser.parseFromString(): https://developer.mozilla.org/en-US/docs/Web/API/DOMParser/parseFromString
- MDN, SVG use element: https://developer.mozilla.org/en-US/docs/Web/SVG/Reference/Element/use

## What this tool does not claim

- An empty result is not a proof of safety. It means none of the checked patterns are present, for
  the sinks you declared.
- The measured tables cover the constructs listed and no others, in one browser, on one date.
- Indented code block detection is an approximation of the CommonMark rule: four or more spaces
  following a blank line. Link labels are matched on a single line.
- The terminal reconstruction is approximate. It covers printable text, carriage return, backspace,
  line feed, cursor up, down, forward, back, next line, previous line, column address and absolute
  position, and erase in line and erase in display. Everything else is skipped. It is capped at 5000
  rows by 1000 columns; past that the tool says the reconstruction was capped and declines to claim
  a difference it cannot stand behind, rather than reporting a truncated view as the whole picture.
- Whether markdown becomes an `img` element depends on your renderer. The tool follows CommonMark.

## Companion tool

This is the output-side half of a pair. The input side, which lints MCP tool definitions for
injection and exfiltration instructions before you install them, is
[mcp-tool-poisoning-scanner](https://0xelitesystem.github.io/mcp-tool-poisoning-scanner/).

## Privacy

Everything runs in your browser tab. There is no backend, no API key, no telemetry, no analytics and
no external dependencies: the entire tool is one HTML file with its CSS and JavaScript inline. The
page issues no network requests of any kind, which was verified by serving it from a request-logging
server and loading it in a real browser with the hostile sample applied. The only requests recorded
were the page itself and the browser's own favicon probe. Your pasted model response never leaves
the machine, and the whole page works offline once loaded.

The core analysis functions are pure: they take a string and a config object and return a plain
result object, with no DOM access, so you can lift them out of the file and run them in Node.

The only thing written to storage is your light or dark theme choice, saved in `localStorage` under the key `losk.theme`. The source links on the page go to external sites only when you click them.

## Run locally

```bash
git clone https://github.com/0xelitesystem/llm-output-sink-scanner
cd llm-output-sink-scanner
```

Open `index.html` in any modern browser. Or serve the folder with `python -m http.server 8000` and visit http://localhost:8000/.

## Build

No build step. The whole tool is one `index.html` file with its CSS and JavaScript inline, and nothing to install.

## License

MIT. See [LICENSE](LICENSE).

## More

- More tools: https://0xelitesystem.github.io/
- https://elitesystem.ai

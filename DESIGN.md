# sitegen22 — Turn 1 recon + anchor (DESIGN.md)

Competition: `site-generator-speed-01-run-20261122` — "Build a static site faster, 22 November 2026".
Prize $100 (10 × $10). Closes 2026-11-24T20:00:00Z (2026-11-25 03:00 WIB).
Account @katzuomaru. Repo `~/workspace/makefun/sitegen22`.

## Anchor

```
makefun anchor e051e7fb24c4b0ec2185fcfd64848670181baa09 round 34194932
```

Produced by `makefun init` as the FIRST tool result in this fresh repo (step 1 of the
turn-1 task). Keypair file contents never printed.

**Sol/Xingchi relay DOWN this run** — 503 on 2026-11-23. No Sol draft; the design below is
done in-session from the reference code (vendor code read fully). Sol-first default does not
apply: the relay is unavailable, and the work is a byte-exact reimplementation, not a draft.

**Kit layout note:** the task brief's paths (`vendor/makefun-site/...`) do not exist at the
kit top level. The reference lives under
`baselines/makefun-site/vendor/makefun-site/` (README.md, index.js, lib/frontmatter.js,
lib/markdown.js, lib/template.js, lib/text.js). RULES.txt (284 lines, read fully),
competition.yaml, kit.json and the case generator
(`program/generator/subject/sites.mjs`, read fully) were also read.

## Baseline verification (turn 1)

- Baseline run on the practice set (local node v24.22.1, shared 1.5 CPU): **~0.95 s wall
  (~1.07 s user CPU) for all 3 cases**. Sandbox is node22, 1 CPU, 10 s wall — expect the
  baseline slower there; measure beside it in turn 3, never against board numbers.
- Baseline output verified **identical to `practice/reference/expected.json` on all 3 cases
  (0 diffs)**. `expected.json` shape: `{format, oracle:{name:"makefun-site",version:"1.0.0"},
  groups, cases:[{id, group, output}]}` — answers are the `output` objects, in case order.

## (a) Pipeline order (index.js `build` + PIPELINE)

`build(input)`:

1. **Templates first, before any file work.** `Object.keys(input.templates).sort(byCodeUnits)`
   (code-unit `<`/`>` comparator, never locale). Each `parseTemplate(name, src)`; a
   `TemplateError` is caught → `report.push(err.message)` and the template is left OUT of
   the map (as if absent). Any other error rethrows.
2. `readSources`: `[...input.files].sort(by path, code-unit)`; path ending `.md` → source,
   `content.replace(/\r\n?/g, '\n')` (CRLF and lone CR → LF); otherwise static (content as-is).
3. `frontMatter` — per source, in sorted source order, checks in THIS order; first failure
   reported, source dropped:
   1. `parseFrontMatter` error → `refused <path>: <reason>`
   2. path under `tags/` or `page/` → `refused <path>: <dir>/ is the generator's own`
      (code: `RESERVED.find(d => d !== 'posts' && path.startsWith(d + '/'))`; `posts/` is fine)
   3. post (`path.startsWith('posts/')`) with no `date` key → `refused <path>: a post needs a date`
   4. layout = `data.layout ?? (post ? 'post' : 'page')` not in parsed-templates map →
      `refused <path>: layout "<layout>" is no template that parses`
   5. `data.draft === true` → `draft <path>` (note: NOT `refused`, no reason suffix)
4. `permalinks`: dir = post ? `'posts'` : `dirOf(path)` (dir before last `/`, else `''`).
   slug = `uniqueSlug(data.slug ?? slugify(data.title), takenIn(dir))`; root dir's taken set
   starts as `RESERVED = ['posts','tags','page']`. `outDir = dir === '' ? slug : dir/slug`;
   `url = base + '/' + outDir + '/'`; `base = site.base_url.replace(/\/+$/, '')` (all trailing
   slashes dropped; base can be empty string only if base_url is all slashes — not generated).
   `bySource.set(source path → source)`.
5. `markdown`: per source, `renderMarkdown(body, env)` with `env.link` (see §(d) links).
   Sets `html`, `toc`, `excerpt`.
6. `collections`: posts sorted by date DESC (code-unit compare on the date string:
   `byCodeUnits(y.date, x.date)`) then slug ASC (code-unit). `newer` = previous in that
   order, `older` = next; null at either end. (Pages have no newer/older — `linkOf(null)`.)
7. `tags`: per post, `ownTags` = its tags with each slug once, first spelling kept, in its
   own order. Walking posts in collection order: first meeting of a slug creates the tag
   `{name, slug, url: base + '/tags/' + slug + '/', posts: []}`; tag named as first met.
   `b.tags` sorted by post count DESC, then slug ASC (code-unit). Page tags: same slugs,
   linked to the posts' tag of that slug, else `{name, slug, url: '', posts: []}`.
8. `layouts`:
   - `cards` map built ONLY for posts: `cardOf(post)`. `site = {title, base_url: base}`;
     `allTags` = tags as `{name, slug, url, count}`; `recent` = first `site.recent` posts' cards.
   - `render(name, vars)` → `null` if template missing (no output written for that page).
   - Every kept source: `output[outDir + '/index.html'] = render(layout, {page: pageOf(s)})`.
   - Index: `per = site.per_page`, `pages = max(1, ceil(posts/per))`; `indexUrl(k) =
     k===1 ? base+'/' : base+'/page/'+k+'/'`; pager `{number:k, total:pages, url:indexUrl(k),
     newer_url: k===1?'':indexUrl(k-1), older_url: k===pages?'':indexUrl(k+1)}`;
     page k renders `posts.slice((k-1)*per, k*per)` as cards; written to `index.html` (k=1)
     or `page/<k>/index.html` — only if the `index` template exists.
   - Each tag: `render('tag', {tag: tagOf})` → `tags/<slug>/index.html` if template exists.
   - `render('tags', {})` → `tags/index.html` if template exists.
   - Statics: if `output` already has the path → report
     `shadowed <path>: a built page has its path`; else `output[path] = content`.
9. `report`: `lines = [...b.report, ...b.broken].sort(byCodeUnits)` — template errors,
   refusals, drafts, shadowed lines AND broken-link lines all sorted together — then push
   `built <pages> pages, <posts> posts, <tags> tags and <statics> static files` where
   pages = kept non-post sources, posts = kept posts, tags = `b.tags.length`, statics =
   `b.statics.length` (ALL statics, including shadowed ones). `output['build-report.txt'] =
   lines.join('\n') + '\n'` (every line ends in `\n`, including the last).

`b.broken` is a `Set` of `broken-link <source> -> <target>` — each distinct string once.
Output object key order is insertion order (pages, then index/tag/tags pages, then statics,
then report last) — scorer compares as JSON values (objects key-order-insensitive), but keep
the same order anyway.

## (b) Front matter — grammar + exact error strings

`parseFrontMatter(text)` — `text` already has `\n` line ends:

- `lines = text.split('\n')`. `lines[0] !== '---'` → `{error: 'no front matter'}`.
- Close = first later line exactly `'---'`; none → `{error: 'the front matter is not closed'}`.
- Lines `1 .. close-1`, in order, first failure wins:
  - blank (`/^[ \t]*$/`) or starts with `#` → skipped (comment = `#` first, no space needed).
  - Else must match `LINE = /^([a-z_]+):(?: (.*))?$/` — key `[a-z_]+`, colon, optional ONE
    space + rest. No match → `line <i+1> is not "key: value"` (line number counted from 1
    at the opening `---`, i.e. 0-based index + 1).
  - `raw = (m[2] ?? '').replace(/[ \t]+$/, '')` — trailing spaces/tabs dropped, LEADING
    spaces kept. Consequence: `key:  value` (two spaces) yields raw `' value'` — title keeps
    the leading space; tags fails (`' ['` doesn't start with `[`); date/draft/layout/slug fail
    their grammars. `key:` (no space, no value) → raw `''`.
  - Checks per line, in order: unknown key → `unknown key "<k>"`; duplicate →
    `key "<k>" is given twice`; raw `''` → `key "<k>" has no value`; value error → that error.
  - Known keys: `title date tags draft layout slug summary`.
- After the loop: no `title` key → `{error: 'no title'}`.
- Success: `{data, body: lines.slice(close + 1).join('\n')}` — body keeps everything after
  the closing line, including a leading blank line.

Value grammars (`valueOf`, errors prefixed `key "<k>": ` for string/date/tags/draft/layout/slug):

- `title`/`summary` — `stringOf(raw)`: starts with `"` → must be length ≥ 2 and end with `"`,
  else `a quoted string is not closed`. Scan `1 .. len-2`: `\` must be followed by `"` or
  `\` and not be the last char, else `a quoted string holds an escape other than \" or \\`;
  a bare `"` inside → `a quoted string holds a stray quote`; otherwise copy. Bare starting
  with `[` → `a bare string opens on [`. Bare otherwise = raw verbatim. Quoted `""` → `''`.
- `date` — `/^(\d{4})-(\d{2})-(\d{2})$/`, real Gregorian day, 1970-01-01..2099-12-31
  (leap rule `(y%4===0 && y%100!==0) || y%400===0`; 30-day months 4,6,9,11). Else
  `key "date": not a date from 1970-01-01 to 2099-12-31 (YYYY-MM-DD)`. Stays a string.
- `tags` — must start `[` and end `]` else `key "tags": not a list [a, b]`. Inner trimmed
  empty → `[]`. Else split `,`, each trimmed; any empty or matching `/[^"[\]]/` fail →
  `key "tags": an empty tag, or one holding " [ or ]`. (Tags may NOT contain `"`, `[`, `]`.)
- `draft` — `'true'`/`'false'` else `key "draft": true or false`.
- `layout` — `/^[a-z0-9-]+$/` else `key "layout": not a template name`.
- `slug` — `/^[a-z0-9]+(?:-[a-z0-9]+)*$/` and length ≤ 64 else `key "slug": not a slug`.

Edge notes: `Title: Capitalised` fails LINE (uppercase key) → `line n is not "key: value"`,
NOT "unknown key". A `--- ` (trailing space) line is neither a close (exact match only) nor
a key line. FM close search finds the FIRST exact `---` after line 0.

## (c) slugify / uniqueSlug / longDate

`escapeHtml`: single-pass `text.replace(/[&<>"']/g, c => ESCAPES[c])` with
`&`→`&amp;` `<`→`&lt;` `>`→`&gt;` `"`→`&quot;` `'`→`&#39;`. Nothing else.

`slugify(text)`:
1. `text.normalize('NFKD')`, drop combining marks (`/\p{M}/gu`).
2. Iterate by CODE POINT (`for (const ch of plain)`): A–Z lowered; a–z/0–9 kept; any other
   char → gap: one `-` between kept runs, never at either end (gap only set when `out`
   non-empty). NFKD expansions matter: `ﬁ`→`fi`, `Ｗ`→`W`, `Æ`→`AE`, `œ`→`oe`; `ß`, `Ł`,
   `Δ`, `М`, `東` have no NFKD decomposition → dropped (gap). This is the sandbox node22's
   normalization — do not hand-roll tables.
3. `out.slice(0, 64).replace(/-+$/, '')`; empty → `'untitled'`.

`uniqueSlug(slug, taken)`: first of `slug`, `slug-2`, `slug-3`, … not in `taken`; adds it.
Suffix is NOT re-trimmed to 64 (can exceed 64 chars). Per-directory `taken` sets
(permalinks) and per-page heading-id sets (markdown) are separate.

`longDate(date)`: string-only: split `-`, `` `${Number(d)} ${MONTHS[Number(m)-1]} ${y}` `` —
e.g. `2026-11-09` → `9 November 2026`. Year kept as written, day loses its leading zero.

## (d) Markdown — blocks then inline

`lex(src)`: `src.split('\n')`, scan with index `i`. Block-start order per non-blank line:
fence → heading → hr → blockquote → list → paragraph. Blank lines (`/^[ \t]*$/`) skipped.

- **Fence**: `FENCE_OPEN = /^```([A-Za-z0-9_+-]*)[ \t]*$/` — exactly 3 backticks, optional
  lang from `[A-Za-z0-9_+-]*`, then spaces/tabs only. 4+ backticks is NOT a fence. Body runs
  to `FENCE_CLOSE = /^```[ \t]*$/` or EOF; the close line (or EOF) is consumed (`i++` past
  it). Render: `<pre><code class="language-<lang>">…</code></pre>` (lang escaped; no class
  when lang empty); each body line escaped + `\n`. Unclosed fence eats to EOF.
- **Heading**: `HEADING = /^(#{1,6})[ \t]+(.*?)[ \t]*$/`, text must be non-empty
  (`headingOf` returns null when `m[2]===''`). 7+ `#` or no space after `#` → NOT a heading
  (falls through to paragraph). Closing `#`s are KEPT as text (`# text ##` → text `text ##`).
  `id = uniqueSlug(slugify(text), ids)` — ids unique PER PAGE. Render
  `<hN id="<id>">inline</hN>`. TOC: depth 2 and 3 only, `{level, id, html: inline(text)}`.
- **Thematic break**: `HR = /^(?:---|\*\*\*)[ \t]*$/` → `<hr>`. Trailing spaces OK.
- **Blockquote**: lines starting `>` (`QUOTE = /^>/`). Each: `replace(/^> ?/, '')` (drop `>`
  + ONE following space if present), then `trimLine` (spaces/tabs both ends), empty ones
  dropped. Continues while lines start with `>` (a blank line ends it). Render: no lines →
  `<blockquote>\n</blockquote>`; else `<blockquote>\n<p>${inline(lines.join('\n'))}</p>\n</blockquote>`
  — lines are NOT re-lexed as blocks; joined with `\n` into one inline run.
- **List**: `BULLET = /^[-*] (.*)$/`, `ORDERED = /^(\d{1,9})\. (.*)$/` (1–9 digits, `. `,
  rest may be empty). The FIRST line fixes the kind for the whole list. Following lines:
  same marker → new item; `CONTINUED = /^ {2,}\S/` (2+ spaces then non-space) → trimmed and
  appended to current item after `\n`; anything else ends the list. No nesting. Items joined
  with `\n`, each rendered inline. Render `<ul>\n<li>…</li>\n…</ul>` or `<ol>` /
  `<ol start="n">` when first number ≠ 1 (`start` from the FIRST line's match).
- **Paragraph**: any other line + following lines until a blank line, fence, heading or
  thematic break (`endsParagraph`: BLANK, FENCE_OPEN, headingOf, HR — note: list lines,
  quote lines and `---`-under-text do NOT end a paragraph except via HR; `===` underlines
  are paragraph text). Each line `trimLine`d, joined `\n`. Render `<p>${inline(text)}</p>`.
  **Excerpt** = first paragraph's inline HTML, or `''` if no paragraph exists.
- Blocks written in order, each + `\n`: `html = blocks.map(b => b + '\n').join('')`.
- Indented lines are NOT code; `=` underlines are text; raw HTML is text (escaped inline).

Inline `inline(s, env)`, left to right, index `i`:

- `\` + next char in `ESCAPABLE = {\, `, *, _, [, ], (, ), #, +, -, ., !, >}` → write
  `escapeHtml(that char)`, skip 2. Any other `\` is itself (written escaped = `\`).
- `` ` ``: `j = s.indexOf('`', i+1)` — plain indexOf, escapes NOT honored inside the search
  (an escaped backtick is consumed by the backslash rule before it can open a span). If
  `j > i+1` → `<code>${escapeHtml(slice(i+1, j))}</code>`; else literal `` ` ``. Empty span
  ` `` ` → two literal backticks.
- `*`: if `s[i+1]==='*'` → `j = findUnescaped(s, '**', i+2)`; if `j > i+2` →
  `<strong>${inline(slice(i+2, j))}</strong>` else literal `**`, skip 2. Else single `*` →
  `j = findSingleStar(s, i+1)`; if `j > i+1` → `<em>${inline(...)}</em>` else literal `*`.
  `_` is always itself.
- `findUnescaped(s, what, from)`: scan `j` from `from` to `s.length - what.length`; if
  `s[j]==='\\'` and next char escapable → skip both (`j++` + loop `j++`); if
  `s.startsWith(what, j)` return `j`; else -1.
- `findSingleStar(s, from)`: same escape skipping; a `*` whose NEXT char is also `*` is
  skipped as a pair (`j++` + loop `j++`) — so `**` runs are never single-star closers, and
  `***` contains no single star. Returns first lone `*`, else -1.
- `[`: `k = findUnescaped(s, ']', i+1)`; `m = (k!==-1 && s[k+1]==='(') ? s.indexOf(')', k+2) : -1`
  (plain indexOf for `)`); `target = m===-1 ? '' : slice(k+2, m)`. If `target !== ''` and
  no whitespace in it (`!/\s/.test(target)`) →
  `<a href="${escapeHtml(env.link(target))}">${inline(slice(i+1, k))}</a>`, `i = m+1`.
  Else literal `[`, `i++`. Note: `!` before `[` is NOT special (no image syntax) — `![a](b)`
  renders `!` + link. Link text is rendered inline recursively (nested links/emphasis work
  on the first-`]` rule).
- Every other char: `escapeHtml(c)` (single char).

Link resolution `env.link(target)` (in `markdown()` plugin):
- `/^[A-Za-z][A-Za-z0-9+.-]*:/` (scheme), or starts with `/` or `#` → kept as written.
- Else split at FIRST `#`: `rel`, `frag` (frag includes the `#`). `rel === ''` → path null.
  `resolvePath(dir, rel)`: split `/`; skip `''` and `.`; `..` pops (null if popping above
  root); else push. If path in `bySource` → `page.url + frag`; else if in static paths →
  `base + '/' + path + frag`; else `broken.add('broken-link <source> -> <target>')` and
  return target as written. (So a link to a draft/refused/missing source is broken unless a
  static file has that path. A link whose target is only a fragment never reaches here —
  `#…` is kept. `rel` with trailing `/` resolves fine since `''` segments are skipped.)

## (e) Templates — tokenize / parse / render

Tokenize (`tokenize(name, src)`):
- Scan for `{`. `{` followed by anything other than `{`/`%` → literal `{` copied to text.
- `{{` / `{{{`: raw iff third char is `{`. Close = `indexOf('}}}' / '}}')`; unclosed →
  `template "<name>": a {{{ is not closed` / `a {{ is not closed`. Path = inner slice
  `.trim()` → `pathOf`: must match `PATH = /^[A-Za-z_][A-Za-z0-9_]*(?:\.[A-Za-z_][A-Za-z0-9_]*)*$/`
  else `template "<name>": ${JSON.stringify(text)} is not a path` (JSON.stringify escaping
  matters for the error string). `{{ }}` (empty) is an error.
- `{%`: close = `indexOf('%}', open+2)`; unclosed → `template "<name>": a {% is not closed`.
  **Standalone rule:** `lineStart = src.lastIndexOf('\n', open-1)+1`; `lineEnd` = next `\n`
  after tag end or EOF. Standalone iff only spaces/tabs between lineStart and the tag AND
  only spaces/tabs between tag end and lineEnd. If standalone: the accumulated `text` is
  truncated by `(open - lineStart)` chars (removing the line's leading spaces) and `i`
  jumps past the newline (`nl+1`) or to EOF — the whole line INCLUDING its `\n` is gone.
  Non-standalone: `i = end` (just past `%}`), surrounding text kept.
- Text tokens pushed only when non-empty.

Parse (`parseTemplate`, stack of open for/if nodes):
- `for`: `FOR = /^for ([A-Za-z_][A-Za-z0-9_]*) in (\S+)$/` — exact single spaces; path
  validated by `pathOf`. `if`: `IF = /^if (not )?(\S+)$/`. `include`:
  `INCLUDE = /^include "([a-z0-9-]+)"$/` (quoted, lowercase/digit/hyphen only).
- `else` only when stack top is an `if` with `else === null`, else
  `template "<name>": an else outside an if`.
- `endfor`/`endif` only when top kind matches, else
  `template "<name>": an endfor with no open for` / `an endif with no open if`
  (message uses `tag.slice(3)`).
- Anything else → `template "<name>": {% ${tag} %} is not a tag`.
- End of tokens with stack depth > 1 → `template "<name>": a ${top.kind} is not closed`
  (kind is `for` or `if`). After popping, body restores to parent: root → `body`,
  for → `body`, if → `else ?? then`.

Render (`renderNodes(nodes, scope, ctx, depth)`), scope = `{vars, parent}`:
- `text` → append. `var` → `escape(textOf(lookup))`. `raw` → `textOf(lookup)`.
- `if`: `v = truthy(lookup)`; branch = `v !== not ? then : else`; render branch if non-null.
- `for`: `xs = lookup`; not an array → nothing. Else per element: child scope
  `vars = {[name]: x, loop: {index: k+1, first: k===0, last: k===len-1, length: len}}`,
  `parent: scope`. (If the loop var is named `loop`, the `loop` object wins — it is set
  after `[name]` in the literal.)
- `include`: `t = templates.get(name)`; if defined AND `depth < INCLUDE_DEPTH (8)` →
  render in the SAME scope at `depth+1`. Missing or depth ≥ 8 → nothing. Top render starts
  at depth 0, so 8 nested include levels render (depths 0→7 include, depth-8 include is a no-op).
- `lookup(scope, path)`: walk the scope chain; first scope whose `vars` has OWN property
  `path[0]` wins; then each next name must be an own property of a non-null object value,
  else `undefined`. (`loop` is found before the loop element because both live in the same
  `vars` object and lookup checks `hasOwn(vars, path[0])` — the FIRST name only; shadowing
  is by nearest scope.) Note: `arr.length` resolves (own property of arrays).
- `textOf(v)`: string → itself; number → `String(v)` (JS digits, e.g. `1e+21`); boolean →
  `true`/`false`; array → items mapped through `textOf`, joined `', '`; anything else
  (undefined, null, object) → `''`.
- `truthy(v)`: array → `length > 0`; else `v !== undefined && v !== null && v !== false &&
  v !== '' && v !== 0`. Note: `NaN` is truthy (matches none of the exclusions).

## (f) Practice data scale + template features actually used

Per case (from `practice/input/cases.json`):

| case | files | .md | static | templates | posts | pages | md chars | md lines | ~chars/md |
|---|---|---|---|---|---|---|---|---|---|
| g-0001 | 257 | 250 | 7 | 27 | 163 | 87 | 637,479 | 15,134 | 2,550 |
| g-0002 | 348 | 340 | 8 | 22 | 227 | 113 | 870,520 | 20,511 | 2,560 |
| g-0003 | 486 | 480 | 6 | 21 | 338 | 142 | 1,264,609 | 29,744 | 2,635 |

Totals: ~1,070 sources, ~2.77M body chars. `site`: `per_page` 6/10/5, `recent` 3/3/4,
`base_url` `https://<word>-<word>.example/blog` (no trailing slash in practice).
FM usage: with `summary` 69/79/118, with `date` = posts exactly (163/217/334 — pages rarely
dated), with explicit `slug` 10/12/28, with `layout` 0/0/3, `draft:` lines 44/42/67
(~10% `draft: true` per generator), tags total 385/488/712. Directories:
`` (root), `about`, `docs`, `docs/guides`, `notes`, `posts/2018`–`posts/2026`, `projects`.
Statics: `assets/style.css`, `robots.txt`, 4–6 `assets/img/*.svg`.

Template feature counts (all templates per case): `for` 16/12/9, `if` 42/22/22, `else` 6/4/5,
`include` 27/26/27, `{{{` raw 4/4/4, `{{` var 77/68/61, `if not` 8/1/2, `loop.` 18/9/3.
Core templates (every case): `post page index tag tags head header footer meta nav card toc
pager tag-list` + 0–15 fillers (`promo/aside/note/badge/strip-N`). Include graph (g-0001):
`head→meta`; `header→nav` (70% of sites); `footer→note-1` + extras; `card→tag-list`;
`post→head,header,toc,strip-12×2,aside-10,footer`; `page→head,header,toc,strip-12,footer`;
`index→head,header,card(×posts on page),pager,footer`; `tag→head,header,card,footer`;
`tags→head,header,footer`. Raw uses are only `{{{ p.excerpt }}}`, `{{{ h.html }}}`,
`{{{ page.content }}}`. `nav` loops `recent` on EVERY page render (3–6 cards of links).
Output size: case g-0001 → 276 output paths.

Markdown feature counts (g-0001 bodies): fences 410, headings 566, hr 151, quotes 528,
bullets 628, ordered 400, backticks 2,438, `**` 4,054, single-`*` ≈5,368, `[` 1,492,
backslashes 1,172, raw `<` 2,744, closing-hash headings 35, images 0, 7-`#` headings 0,
empty headings 0, unclosed code/strong/em/link 0 (in practice; the generator never emits
unclosed constructs — but implement the general rules anyway).

## (g) Optimisation plan (for turn 2, solve.js)

Constraints: dependency-free ES module, repo ≤ 4 MiB, Node 22, no network, read-only /bot,
1 CPU, 1024 MB, 10 s wall, executed twice byte-identical. `run(input)` per case, alone.
Keep the house `main.ts` driver as-is (it already sets `--no-incremental-marking` before
importing solve.js). Scored set is drawn after close from the SAME generator (see §(h)),
so the distribution above is the target — but correctness must be general, not fitted to
practice quirks.

1. **Template compile-to-function (biggest win).** Parse each template ONCE per case into a
   JS function (or a flat op list executed by a tight loop) instead of walking node trees
   with `renderNodes` + string `+=` per node. Pre-split paths; inline `textOf`/`truthy`/
   `lookup` as small helpers. Includes: compile each template to `fn(scope, depth)`; an
   include node becomes `depth < 8 ? fns[name](scope, depth+1) : ''` (guard missing
   templates at compile time — a template that failed to parse is absent, so its include
   renders nothing). Preserve the standalone-line semantics at COMPILE time (tokenizer
   output is identical; only the executor changes). Render into an array + `join('')`.
   Compile cost is amortised: ~21–27 templates/case, but `card` renders ~hundreds of times
   (once per post per index/tag page it appears on) and `head`/`header`/`footer` render once
   per page (200–600×).
2. **Card reuse — do NOT recompute `cardOf` inside `pageOf`.** Baseline `pageOf` spreads
   `cardOf(s)` (fresh `wordsOf` regex + `longDate` + tag map per page). Build each source's
   card ONCE (posts and pages alike), reuse for `pageOf`, `tagOf`, index/tag pages, `recent`.
   `reading_minutes`: replace `text.match(/\S+/g)` (allocates an array of every word) with
   a manual run-count scan over the body; compute once per source. `longDate` is cheap but
   called twice per card in baseline (card + linkOf) — compute once, store.
3. **Slugify cache (per case only).** `slugify` is pure; cache by exact input string in a
   per-case `Map`. Hot keys: tag names (~40 distinct, used 385–712×/case), titles (respells
   collide), heading texts (20% reused). **NEVER cache across cases** — each case is
   answered alone; a fresh cache per `run()` call. Same for `longDate` if it shows up.
4. **Front matter without full split.** Don't `text.split('\n')` the whole source and
   `join` the body back. Scan line boundaries with `indexOf('\n')` to find the closing
   `---`, parse only the FM lines, take `body` as a substring. The markdown lexer then
   works on the body substring with index-based line iteration (no `split('\n')` of the
   body either — or one split if profiling shows it cheaper; measure).
5. **Single-pass scanners, no token objects.** Markdown lexer: iterate lines by index,
   emit block HTML strings directly into an array (no `{type,…}` token allocation, no
   second render pass over tokens). Inline renderer: accumulate plain-text runs and escape
   in chunks (find next special char, slice, escape the chunk with one regex replace)
   instead of `escapeHtml(c)` per character. Keep the recursive `inline()` on
   strong/em/link inners (inners are short).
6. **Fast escape.** `escapeHtml` via regex replace is fine for chunks, but avoid calling it
   on strings known to contain no specials: a quick `indexOf` check for `&<>"'` first, or
   a char-code scan that only builds a new string when a special is found.
7. **Deduplicate repeated work in `tags()`:** baseline calls `slugify(name)` twice per tag
   occurrence (filter + map) and re-slugifies in `ownTags` per source. Compute each tag's
   slug once (cache covers it).
8. **Sorting/report:** keep code-unit comparators (cheap). Collect report lines in one
   array, single `.sort(byCodeUnits)`. `broken` as a `Set` (dedup by string).
9. **Avoid allocation churn in hot loops:** reuse the per-case `bySource` Map / static
   Set; build output object directly; avoid spreading objects (`{...cardOf(s)}` allocates
   per page — construct `pageOf` fields onto the cached card object shape directly, or
   build the page object once with card fields copied by assignment).
10. **Measure in turn 3** before micro-optimising further: profile practice runs (baseline
    vs turn-2 solve.js) with `--cpu-prof` or simple timers per phase; the likely hot spots
    are inline rendering, template rendering, and `wordsOf`/slugify — verify, don't guess.
    Keep `main.ts` untouched; only `solve.js` is replaced.

## (h) Scored-set distribution (from the generator — what turn 2 must handle)

Same generator draws the scored set (bands by index%3: [200,333], [334,466], [467,600]
sources; ~70% posts under `posts/YYYY`, pages at root/`docs`/`docs/guides`/`about`/
`projects`/`notes`):

- ~10% drafts; ~5% non-ASCII titles (café, Zürich, Ærøskøbing, Ｗｉｄｅ, ﬁnal, Москва, 東京…);
  ~5% respelled duplicate titles (case/punctuation/spacing variants → slug collisions,
  `-2`/`-3` suffixes, incl. titles that slug to the same value in DIFFERENT directories).
- ~5% refused front matter — the 15 `REFUSALS` variants: unknown key (`author:`), duplicated
  key line, `date: 2026-02-30` / `2025-13-01` / `26-11-19`, missing title, `tags: [one, "two"]`,
  `tags: one, two`, `draft: yes`, unclosed quote, `layout: no-such-layout`, `Title:` (capital),
  `slug: Not A Slug`, `summary:` (no value), post's `date:` line removed (→ post, no date).
  Plus 1% with the opening `---` stripped (→ `no front matter`), 5% CRLF files, 8% blank
  lines inside FM, 10% FM comment lines, quoted titles (30% of titles, escaped `\"`/`\\`).
- `summary:` 25% of sources (bare; a leading `"` or `[` is stripped by the generator before
  writing); explicit `slug:` 5%; `layout: wide` on non-posts when the site has a `wide`
  layout (30% of sites — a `page` clone with `-wide` class).
- Bodies (4–16 blocks, 900–3800 bytes): paragraphs with inline marks incl. nested
  `**word *word* word**`, escaped `\*word\*`, raw HTML `<b>/<span>/<em>/<kbd>`, `&`, quotes,
  apostrophes, `< N > N`; headings levels weighted [2,2,2,3,3,4,1,5,6], 20% heading-text
  reuse (duplicate ids → `-2` suffixes), 10% closing ` ##`; lists (40% ordered, start 1 in
  70% else 2–20, 2–7 items, 15% continuation lines); code fences (langs js/ts/html/css/json/
  sh/python/rust/c++/empty) with escape-heavy bodies; blockquotes 1–4 lines; hr `---`/`***`/
  `---  ` (trailing spaces); setext-style paragraph + `---`/`===`/`***` (`---`/`***` split
  the paragraph into para + hr, `===` stays paragraph text); raw `<div>` blocks; indented
  `    …` lines (paragraph text, NOT code); `#word …` lines (paragraph, NOT heading);
  `N) …` / `+ …` lines (paragraph, NOT lists); triple blank separators 10%.
- Links (8 targets/source): 4 relative page links, 1 with `#frag`, 1 static-file link,
  1 `missing/<word>.md`, 1 `https://<word>.example/<word>?q=<word>&n=<digit>`, 1
  `mailto:<word>@example.com`, 1 `#<word>`, 1 `/<word>/`. Links to drafts, refused sources
  and missing files → `broken-link` lines. Relative links climb with `../` as needed.
- Templates: the 5 layouts + partials + 0–15 fillers; filler bodies in 4 variants (plain
  aside, `recent` loop, `tags` loop with `loop.index`, `page.title`/`page.date_long` if).
  `header` includes `nav` in 70% of sites; `footer` includes 0–2 extras; `post` includes
  `toc` in 80%, `pageTags`/`pageMeta` order randomised (50% reversed in post).
- `site`: `base_url` = `https://<word>-<word>.example` + `/blog` (30%) + trailing `/` (30%);
  `per_page` 5–15; `recent` 3–6; title may end ` & Co.` (20%). Files handed in shuffled
  order (build sorts). Statics: `assets/style.css`, `robots.txt`, 1–6 `assets/img/*.svg`.

## Highest-risk byte-exactness areas (for turn 2 to test hardest)

1. **Inline Markdown scanning** — `findUnescaped`/`findSingleStar` escape-skipping, the
   `**` vs single-`*` interplay (`**a *b* c**`, `***`, `**a***`), code-span search with
   plain `indexOf` (no escape honoring), link rule (first unescaped `]`, `(` must follow
   at once, first `)` via plain `indexOf`, whitespace veto on target), recursive inline on
   strong/em/link inners.
2. **Template standalone `{% %}` line rule + tokenize edges** — standalone detection from
   raw source line boundaries, text truncation by `(open - lineStart)`, `\n` consumption;
   plus include depth-8 cutoff, `lookup` shadowing (`loop` before the element, own-property
   walk, `arr.length`), `textOf`/`truthy` edge values (`NaN` truthy, nested arrays, `0`/`""`
   falsy).
3. **Slugify Unicode semantics + error precedence** — NFKD + `\p{M}` stripping + 64-char
   cut with trailing-`-` drop (node22's `normalize`), `uniqueSlug` suffixes past 64 chars,
   root `RESERVED` set, per-directory taken sets, per-page heading-id uniqueness; and
   front-matter check order (first failing line/check wins) with exact error strings,
   plus the report's single code-unit sort over ALL line kinds (template errors, refusals,
   drafts, shadowed, broken-links) before the `built …` line.

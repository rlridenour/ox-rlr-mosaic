# ox-rlr-mosaic

An Org export back-end that turns an Org buffer into a Typst
presentation built with the
[Mosaic](https://github.com/vincentarelbundock/mosaic) package.

It derives from [`ox-rlr-typst`](https://github.com/rlridenour/ox-rlr-typst),
so every construct that back-end already translates — emphasis, lists,
tables, links, footnotes, citations, source blocks, LaTeX math through
mitex — is inherited unchanged. What this back-end adds is slide
semantics: the Mosaic preamble, the heading-to-slide mapping, explicit
`#m.slide(...)` calls driven by headline properties, multi-cell slide
bodies, incremental reveals, and speaker notes.

## Installation

`ox-rlr-typst.el` must be on your `load-path`; this back-end requires
it.

```elisp
(require 'ox-rlr-mosaic)
```

On the Typst side, the default package spec is Mosaic's development
version, `@local/mosaic:0.0.2`. It carries the presenter console and
fixes the published package does not have, and it is installed from a
clone of the Mosaic repository:

```sh
git clone https://github.com/vincentarelbundock/mosaic.git
cd mosaic
make install
```

On macOS `make install` writes to `~/.local/share`, which Typst does
not read; pass the path it does read:

```sh
make install TYPST_PACKAGE_PATH="$HOME/Library/Application Support/typst/packages"
```

To build against Typst Universe instead — nothing to install, fetched
automatically on first compile — set `org-rlr-mosaic-package` to
`"@preview/mosaic:0.0.1"`, or `#+MOSAIC_PACKAGE: @preview/mosaic:0.0.1`
per deck. The published 0.0.1 has no presenter console, and sets a
space before the comma in a quote credit.

## Usage

- `M-x org-rlr-mosaic-export-to-typst` — export to a sibling `.typ` file.
- `M-x org-rlr-mosaic-export-to-pdf` — export and compile it with `typst`.
- `M-x org-rlr-mosaic-export-as-typst` — export to a temporary buffer.
- From the export dispatcher (`C-c C-e`), under `m` ("Export to Mosaic
  slides").
- `org-rlr-mosaic-publish-to-typst` works as a `:publishing-function`.

Two example decks are included. `examples/demo.org` exercises every
feature below and produces a 19-page PDF; `examples/presenter.org` is
built around speaker notes and a presenter console.

## The heading model

Mosaic builds a deck out of ordinary Typst headings: after
`#show: m.setup`, a level-one heading (`=`) opens a *section slide* and
a level-two heading (`==`) opens a *content slide*. This back-end
preserves that model rather than hiding it, so the mapping is direct:

```org
#+TITLE: A short talk
#+AUTHOR: Ada Lovelace

* Methods            → section slide
Text here becomes that section slide's subtitle.
** Data              → content slide
One slide.
** Model             → content slide
Another slide.
```

`org-rlr-mosaic-slide-level` (default 2) decides which Org level becomes
a content slide, and `#+MOSAIC_SLIDE_LEVEL:` overrides it per file. Set
it to 1 for a flat deck in which every top-level heading is a slide and
there are no section dividers.

Headings **deeper** than the slide level stay ordinary Typst headings
inside the slide body — they do not start a new slide.

Prefer these plain headings. An explicit `#m.slide(...)` is only
emitted when a heading actually needs one, which is what Mosaic's own
guidance recommends.

## Deck-wide configuration

These keywords are passed to `m.setup`:

| Keyword | Effect |
| --- | --- |
| `#+TITLE:`, `#+SUBTITLE:`, `#+DATE:` | Deck identity |
| `#+AUTHOR:` | Split on `;`, or on `,` when there are none, into Mosaic's author array |
| `#+MOSAIC_AUTHORS:` | Raw `authors:` value, for `m.layouts.author(...)` records |
| `#+MOSAIC_THEME:` | Theme facade: `default`, `editorial`, `metropolis`, `manifesto`, `mono` |
| `#+MOSAIC_PAPER:` | `16-9` (default) or `4-3` |
| `#+MOSAIC_COLORS:` | Palette, or overrides of one, e.g. `m.palettes.dark` |
| `#+MOSAIC_LAYOUTS:` | Replace the configured `content`, `title`, or `section` layout |
| `#+MOSAIC_CELLS:` | Recurring cell defaults, e.g. `(footer: [My course])` |
| `#+MOSAIC_BACKGROUND:`, `#+MOSAIC_FOREGROUND:` | Full-slide planes |
| `#+MOSAIC_SPACING:` | e.g. `(inset: 1.5em)` |
| `#+MOSAIC_HANDOUT:` | `t` emits only the final frame of each slide |
| `#+MOSAIC_OUTPUT:` | `slides` (default), `speaker`, `notes`, or `split` |
| `#+MOSAIC_NOTES:` | Note-layout settings, e.g. `(split-inset: 10mm)` |
| `#+MOSAIC_NOTES_PAPER:` | Paper for the printed `speaker`/`notes` outputs (default `us-letter`) |
| `#+MOSAIC_TABLE_RULE_STROKE:` | Stroke for a table's `\|---\|` rules (default `0.8pt + text.fill`) |
| `#+MOSAIC_OVERFLOW:` | `off` (default), `error`, or `record` |
| `#+MOSAIC_FROZEN_COUNTERS:`, `#+MOSAIC_FROZEN_STATES:` | Advance once per logical slide |
| `#+MOSAIC_SETUP:` | Raw extra arguments, one per line, repeatable |
| `#+MOSAIC_PREAMBLE:` | Raw Typst rules emitted just after `m.setup`, one per line, repeatable |
| `#+MOSAIC_PACKAGE:` | Package spec (default `@local/mosaic:0.0.2`) |
| `#+MOSAIC_QUOTE_COMPONENT:` | `t` routes every `#+begin_quote` through Mosaic's quote component |
| `#+MOSAIC_TITLE_SLIDE:` | `nil` for none, or a variant name such as `kicker` |

### Dark decks

Polarity is a palette rather than a theme — Mosaic ships no dark theme,
and every theme adapts to either:

```org
#+MOSAIC_COLORS: m.palettes.dark
```

`palettes` also carries `light`, `parchment`, `sage`, `stone`,
`espresso`, `forest`, and `slate`. Each is an ordinary dictionary, so
tuning one is addition: `m.palettes.dark + (accent: rgb("#b91c1c"))`.
Partial overrides work the same way — `(accent: rgb("#007f73"))` keeps
the theme's other colors.

Write `m.palettes.NAME` rather than the `mosaic.palettes.NAME` spelling
Mosaic's own documentation uses. Every facade exports `palettes`, so the
`m.` form works whether or not the deck names a theme; the `mosaic.`
form only resolves when it does, because the package is aliased as
`mosaic` only alongside a theme facade.

In the presenter outputs a dark deck stays dark on the projected half
while the notes half stays black on white, which is the polarity you
want on a laptop under house lights.

### Deck-wide rules

Deck-wide `set` and `show` rules — typography, cell styling, note
styling — belong in `#+MOSAIC_PREAMBLE:`, which Mosaic expects
immediately after `m.setup`:

```org
#+MOSAIC_PREAMBLE: #set text(font: "EB Garamond", size: 26pt)
#+MOSAIC_PREAMBLE: #show label("mosaic-cell-body"): set align(horizon)
```

A `#+MOSAIC:` line before the first heading lands *after* the title
slide instead, so rules written there miss it.

A title slide is emitted automatically when the document has a title.
`#+OPTIONS: toc:t` adds a table-of-contents slide; unlike most Org
back-ends this is **off** by default, since a deck's TOC is a slide of
its own rather than front matter.

## Per-slide layouts

Any `:MOSAIC_*:` headline property promotes that heading to an explicit
`#m.slide(...)` call and becomes the corresponding Mosaic layout field.
The names in `org-rlr-mosaic-slide-fields` are recognised — `layout`,
`variant`, `columns`, `tracks`, `image`, `caption`, `cells`,
`background`, `numbered`, `invert`, and more.

```org
** Growth since 1950
:PROPERTIES:
:MOSAIC_LAYOUT: image
:MOSAIC_IMAGE: fig/chart.png
:MOSAIC_CAPTION: Estimates from the replication archive
:END:
```

becomes

```typ
#m.slide(layout: "image", image: path("fig/chart.png"),
         caption: [Estimates from the replication archive])[== Growth since 1950]
```

Two further properties are not layout fields:

- `:MOSAIC_ARGS:` — raw arguments appended to the call, for anything
  the field list does not cover (`numbered: false`).
- `:MOSAIC_SCOPE:` — Typst rules that wrap this slide alone, in the
  `#[ ... ]` block Mosaic documents for scoped styling.

### Values

Values are passed through to Typst essentially verbatim, so the whole
Mosaic API stays reachable. A small amount of coercion keeps the common
cases readable: a bare word is quoted for string fields like `layout`
and `variant`, wrapped in `path(...)` for `image`, and wrapped in
`[...]` for content fields like `caption` and `title`. Anything that is
already a Typst expression — starting with `(`, `[`, `#`, `"`, a number,
or a function call — is left alone:

```org
:MOSAIC_IMAGE: (path: path("fig/cover.jpg"), scrim: black.transparentize(55%))
:MOSAIC_LAYOUT: m.layouts.content(variant: "header-body", columns: 2)
```

## Multi-cell slides

`#+MOSAIC: split` divides a slide body into Mosaic's positional cells.
On its own it is enough to make a two-column slide — no property
needed:

```org
** Comparison

- Left column

#+MOSAIC: split

- Right column
```

Add `:MOSAIC_TRACKS: (2fr, 1fr)` for an uneven split, or use the split
to fill the second region of an image layout's `left`/`right`/`top`/
`bottom` variant.

## Incremental reveals and notes

```org
** Findings

- The estimate is positive.

#+MOSAIC: pause

- The interval excludes zero.

#+begin_note
Pause here and take questions.
#+end_note
```

| Org | Typst |
| --- | --- |
| `#+MOSAIC: pause` | `#m.steps.pause` |
| `#+begin_note` | `#m.note[...]` |
| `#+begin_reveal` | `#m.steps.reveal[...]` |
| `#+begin_step 2-4` | `#m.steps.on("2-4")[...]` |
| `#+begin_replace` (alternatives separated by `#+MOSAIC: split`) | `#m.steps.replace[...][...]` |
| `#+begin_callout`, `#+begin_card`, `#+begin_badge` | `#m.components.NAME(...)[...]` |
| `#+begin_quote` | `#quote(block: true)[...]`, or Mosaic's quote component — see below |

A reveal block that holds only a bulleted list is passed to Mosaic one
item per body, `#m.steps.reveal[...][...]`, with each item in its own
block. Mosaic would otherwise rebuild the list with plain bullets in
place of the theme's markers, so nested lists would not match their
parents. Each top-level item is revealed with its sub-items.

A `#+ATTR_MOSAIC:` line supplies further arguments to any of these, with
the same value coercion as headline properties:

```org
#+ATTR_MOSAIC: :role warning :title Careful
#+begin_callout
Components are reachable as named blocks.
#+end_callout
```

Any other `#+begin_NAME` block falls through to the parent back-end,
which calls a same-named Typst function.

## Quotations

Org parses `#+begin_quote` into its own element type rather than a
special block, so it is handled separately. A plain quote block becomes
a native Typst `#quote(block: true)`, which the theme styles as ordinary
block quotation. Adding an `#+ATTR_MOSAIC:` line switches it to Mosaic's
panelled quote component, whose arguments have nowhere else to go:

```org
#+ATTR_MOSAIC: :attribution Ada Lovelace :source Notes, 1843
#+begin_quote
The Analytical Engine weaves algebraic patterns.
#+end_quote
```

`:attribution` and `:source` are content fields, so bare words are
wrapped for you and Typst markup is passed through. `:role`, `:fill`,
`:accent`, and the component's other arguments work too. Mosaic renders
the two on one line separated by a comma.

Values on an `#+ATTR_MOSAIC:` line are **Typst, not Org**, so italicise
a title with Typst's `_..._` rather than Org's `/.../`:

```org
#+ATTR_MOSAIC: :attribution Aristotle :source _Politics_
```

To use the component for *every* quote block, set
`org-rlr-mosaic-quote-component` to `t`, or `#+MOSAIC_QUOTE_COMPONENT: t`
per file.

## Tables

A table has **no cell borders at all**, and a horizontal rule exactly
where the Org source draws one with `|---|`:

```org
| Method | Estimate |   SE |
|--------+----------+------|
| OLS    |     0.42 | 0.11 |
| IV     |     0.38 | 0.19 |
|--------+----------+------|
| Pooled |     0.40 | 0.09 |
```

That deck gets a line under the header and a line above `Pooled`, and
nothing else. A table written without any `|---|` row gets no lines at
all. The rule under a header repeats with the header if the table breaks
across pages, and consecutive `|---|` rows collapse into one line rather
than stacking.

Typst paints a table rule black, which disappears on a dark deck, so the
rules are restated in `org-rlr-mosaic-table-rule-stroke` — by default
`0.8pt + text.fill`, which follows the deck's own text color in either
polarity and under any theme. `#+MOSAIC_TABLE_RULE_STROKE:` sets it per
file; any Typst stroke expression works:

```org
#+MOSAIC_TABLE_RULE_STROKE: 1pt + m.palettes.dark.accent
```

Setting it to `nil` leaves Typst's own stroke alone.

Everything else about tables comes from `ox-rlr-typst` unchanged.
`#+ATTR_TYPST:` passes arguments straight through to `#table(...)`, so a
deck that *wants* a grid can say so, and a rule given its own stroke by
hand is left as written:

```org
#+ATTR_TYPST: :stroke 0.5pt :fill (x, y) => if y == 0 { luma(240) }
| a | b |
|---+---|
| 1 | 2 |
```

A `#+CAPTION:` or `#+NAME:` wraps the table in `#figure(...)`, so it is
numbered as *Table N* and can be referenced with an ordinary Org link.

## Presenting with a console

`examples/presenter.org` is a second deck built around speaker notes;
`org-rlr-mosaic-export-to-pdf` on it produces 16 double-width pages.

A presenter console puts the slide on the projector and the same slide
with its notes, the next slide, and a clock on your laptop. Typst only
produces a PDF, so the console is a separate program —
[pympress](https://pympress.xyz/) and [pdfpc](https://pdfpc.github.io/)
both read what Mosaic writes. **This needs Mosaic 0.0.2**, which is the
default package spec — see [Installation](#installation).

There are two routes, and they are alternatives rather than a sequence.

**Notes beside the slide.** `#+MOSAIC_OUTPUT: split` makes every frame
one double-width page — the slide at true size on the left, its notes on
the right. Both consoles recognise it by the page's proportions:

```sh
pympress presenter.pdf              # splits automatically
pdfpc --notes=right presenter.pdf   # tell pdfpc which half is notes
```

If a note overflows its half the compile fails and names the frame; give
it more room with `#+MOSAIC_NOTES: (split-inset: 6mm)`.

**A pdfpc payload in the deck itself.** Nothing goes in the document
source for this, and no build step either: Mosaic attaches the notes to
every deck that has any, as an embedded `speaker-notes.pdfpc` keyed to
the physical page each note belongs to. The deck stays an ordinary
slide-shaped file you can project or email, and every reader but a
console ignores the attachment.

pdfpc itself reads only a sidecar, so recover one beside the PDF:

```sh
pdfdetach -savefile speaker-notes.pdfpc -o presenter.pdfpc presenter.pdf
pdfpc presenter.pdf                     # finds the notes beside it
```

Notes reach the attachment as text, so a note's words survive and its
layout does not; a note whose shape matters belongs in `split`. pympress
does not read the format at all. The attachment is written whatever
output the deck compiles, so a `split` build carries its notes on the
page *and* as data.

The printed companions are independent of both: `#+MOSAIC_OUTPUT: speaker`
gives pages with a slide thumbnail above its notes, and `notes` the notes
alone.

Mosaic fixes those companions at A4 and has no argument to change it, so
this back-end emits a `#set page(paper: ...)` rule after `m.setup` and
defaults it to **US Letter**. `#+MOSAIC_NOTES_PAPER:` takes any Typst
paper name per file, `org-rlr-mosaic-notes-paper` sets the default, and
nil leaves Mosaic's A4 alone. The rule is confined to the two printed
outputs: `slides` and `split` size their page from the slide itself, and
a `paper:` rule would replace that geometry.

Letter is about 50pt shorter than A4, so a note that just fits an A4
companion may overflow it. Mosaic fails the compile and names the frame
when that happens, rather than clipping silently.

### Making the notes readable

Mosaic sizes notes for a printed A4 companion — 10pt body, 12pt bold
heading — which is too small to read on a console. They are styled by
their own labels, independent of the deck's theme, so scaling them up is
a pair of rules rather than a theme change:

```org
#+MOSAIC_PREAMBLE: #show label("mosaic-note-body"): set text(size: 16pt)
#+MOSAIC_PREAMBLE: #show label("mosaic-note-heading"): set text(size: 16pt)
```

The same labels drive the printed `speaker` and `notes` builds, so one
pair of rules covers all three.

### Where notes go

`#+begin_note` attaches to the slide it sits in, and a note written
after `#+MOSAIC: pause` or inside `#+begin_reveal` belongs to that frame
rather than to the whole slide.

A slide whose body is *only* a note is handled specially, since notes
render nowhere but a block still counts against the layout's cell
budget. Those notes are folded into the slide's own block — or, for a
title layout, emitted just before it — so a bare note works on layouts
with no body cell to spare, such as the image layout's figure variant:

```org
** Growth since 1950
:PROPERTIES:
:MOSAIC_LAYOUT: image
:MOSAIC_IMAGE: fig/chart.png
:END:

#+begin_note
Bars are indexed to 1950 = 100.
#+end_note
```

## Escape hatches

- `#+MOSAIC: <anything else>` passes through as raw Typst, as
  `#+TYPST:` does.
- `#+begin_export typst ... #+end_export` passes through verbatim.
- `:MOSAIC_ARGS:` and `:MOSAIC_SCOPE:` cover per-slide cases the
  properties do not.

## Known limitations

These are Mosaic's, not the exporter's — hand-written Mosaic hits them
identically.

- **Metropolis rejects section subtitles.** Under
  `#+MOSAIC_THEME: metropolis`, text between a `*` heading and the first
  `**` heading fails to compile with *"the configured section layout is
  a raw grid, which cannot carry layout fields"*, because that theme's
  section layout is a raw grid. Leave section headings bare under
  Metropolis, or use another theme. The other four bundled themes are
  fine.
- **`#+MOSAIC_OUTPUT: split`** and the embedded pdfpc notes need Mosaic 0.0.2,
  the default package spec; a deck pinned to the published
  `@preview/mosaic:0.0.1` accepts only `slides`, `speaker`, and `notes`
  — see [Presenting with a console](#presenting-with-a-console).
- Mosaic has no shrink-to-fit for slide bodies by design. An overflowing
  slide means cutting content or splitting the slide; set
  `#+MOSAIC_OVERFLOW: error` to be told about it at compile time rather
  than discovering it while presenting.
- A slide's positional block count has to match its layout's cell count.
  The exporter pins `variant: "header-body"` on explicit content slides
  so that heading-plus-body is always two blocks regardless of theme; if
  you override `:MOSAIC_VARIANT:` with a three-cell variant such as
  `header-body-footer`, supply the third cell with `#+MOSAIC: split`.

## Dependencies

- `ox-rlr-typst`, on the `load-path`.
- The `typst` binary, version 0.15 or newer, for
  `org-rlr-mosaic-export-to-pdf`.
- The Mosaic and (only when the deck contains LaTeX math) mitex Typst
  packages, both fetched automatically from the Typst Universe registry
  on first compile.

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

Nothing needs to be installed on the Typst side. The default package
spec is the version published on Typst Universe, which Typst fetches
automatically the first time you compile.

## Usage

- `M-x org-rlr-mosaic-export-to-typst` — export to a sibling `.typ` file.
- `M-x org-rlr-mosaic-export-to-pdf` — export and compile it with `typst`.
- `M-x org-rlr-mosaic-export-as-typst` — export to a temporary buffer.
- From the export dispatcher (`C-c C-e`), under `m` ("Export to Mosaic
  slides").
- `org-rlr-mosaic-publish-to-typst` works as a `:publishing-function`.

`examples/demo.org` is a deck exercising every feature below; running
`org-rlr-mosaic-export-to-pdf` on it produces an 18-page PDF.

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
| `#+MOSAIC_COLORS:` | Semantic palette overrides, e.g. `(accent: rgb("#007f73"))` |
| `#+MOSAIC_LAYOUTS:` | Replace the configured `content`, `title`, or `section` layout |
| `#+MOSAIC_CELLS:` | Recurring cell defaults, e.g. `(footer: [My course])` |
| `#+MOSAIC_BACKGROUND:`, `#+MOSAIC_FOREGROUND:` | Full-slide planes |
| `#+MOSAIC_SPACING:` | e.g. `(inset: 1.5em)` |
| `#+MOSAIC_HANDOUT:` | `t` emits only the final frame of each slide |
| `#+MOSAIC_OUTPUT:` | `slides` (default), `speaker`, or `notes` |
| `#+MOSAIC_OVERFLOW:` | `off` (default), `error`, or `record` |
| `#+MOSAIC_FROZEN_COUNTERS:`, `#+MOSAIC_FROZEN_STATES:` | Advance once per logical slide |
| `#+MOSAIC_SETUP:` | Raw extra arguments, one per line, repeatable |
| `#+MOSAIC_PACKAGE:` | Package spec (default `@preview/mosaic:0.0.1`) |
| `#+MOSAIC_QUOTE_COMPONENT:` | `t` routes every `#+begin_quote` through Mosaic's quote component |
| `#+MOSAIC_TITLE_SLIDE:` | `nil` for none, or a variant name such as `kicker` |

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
wrapped for you and Typst markup is passed through
(`:attribution [*Grace Hopper*]`). `:role`, `:fill`, `:accent`, and the
component's other arguments work too.

To use the component for *every* quote block, set
`org-rlr-mosaic-quote-component` to `t`, or `#+MOSAIC_QUOTE_COMPONENT: t`
per file.

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
- **`#+MOSAIC_OUTPUT: split`** needs Mosaic 0.0.2; the published 0.0.1
  accepts only `slides`, `speaker`, and `notes`. Set
  `#+MOSAIC_PACKAGE: @local/mosaic:0.0.2` after installing the
  development version from the Mosaic repository.
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

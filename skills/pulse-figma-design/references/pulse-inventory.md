# Pulse / CV-Bot file inventory

File-specific facts observed in `CV-Bot – Designfiler`, file key
`hrX50tyQXpSb7phWJ5y5Dp`. Node IDs are stable; everything else should be
re-verified. **The component descriptions in the file are more current than this
document** — read those first and treat this as an index.

## Pages

| Page | ID |
|---|---|
| Cover | `425:921` |
| 🖋️ Design | `240:1682` |
| 🧱 Tokens | `282:3549` |
| 🧩 Komponenter | `11:130` |
| 🖼️ Ikoner | `692:15086` |
| ✏️ Skisser | `9:2` |
| 🔍 Oversikt og notater | `0:1` |

## Spacing scale — collection `Pulse-Numbers`

| Token | Value |
|---|---|
| XXXS | 1 |
| XXS | 2 |
| XS | 4 |
| S | 8 |
| M | 12 |
| L | 16 |
| XL | 24 |
| XXL | 32 |
| XXXL | 64 |
| Full | 9999 |

**There is no token for 10.** Any `itemSpacing: 10` you find is legacy, baked into
a component. All padding in the library is already token-bound.

## Colour variables in regular use

`Surface/Inverse`, `Surface/Primary`, `Surface/Primary-Light`,
`Surface/Secondary`, `Surface/Tertiary`, `Surface/Highlight`, `Border/Weak`,
`Border/Subtle`, `Border/Disabled`, `Border/Danger`, `Text/Primary`,
`Text/Subtle`, `Text/Disabled`, `Text/Data`, `Feedback/Danger-subtle`.

## Effect styles

`Big softie` (table and card shadow), `Pulse - Blue glow` (app shell background
glow — 1000px blur, never estimate it), `Portal - Big menu hover`.

## Components

All 30 non-icon components carry a description. Read it before use.

| Component | ID | Variants | Notes |
|---|---|---|---|
| Accordion | `606:4676` | 4 | Slot exists only while Open |
| Alert | `602:3240` | 5 | |
| Button | `240:695` | 45 | Two icon swaps, each with a Show toggle |
| CV score | `282:2491` | 8 | Single-purpose, see below |
| CV-List item | `282:4809` | 9 | Contains a CV score |
| Card | `755:16012` | 1 | Slot |
| Checkbox | `707:15752` | 16 | |
| Divider | `282:5919` | 1 | Fixed width; set the instance to Fill |
| Icon button | `282:3211` | 45 | `Icon` swap |
| Input | `578:2410` | 28 | 7 Type states incl. Readonly |
| List item | `498:1344` | 3 | Generic row, slot for trailing content |
| Menu | `687:15037` | 1 | Slot; the open list for Select |
| Menu-item | `687:14985` | 1 | Row inside Menu — **not** Navigation link |
| Modal | `755:16051` | 1 | |
| Navigation link | `240:1644` | 4 | App shell sidebar link |
| Portal big navigation | `130:1744` | 3 | Fixed 280×128 by design |
| Pulse - App shell | `974:20406` | 1 | Sidebar + content slot |
| Radio button | `768:16104` | 16 | |
| Select | `687:14876` | 10 | Trigger only; pair with Menu |
| Tab | `768:17298` | 3 | |
| **Table** | `1031:2` | 1 | Wrapper, two slots |
| **Table cells** | `886:19537` | 3 | Cell primitive |
| **Table row** | `1044:317` | 15 | `Columns` 2–6 × `Type` |
| Tabs | `768:17319` | 5 | `Amount` 2–6 |
| Tags | `84:844` | 8 | |
| Text link | `240:1612` | 3 | |
| Thinking anim | `193:1237` | 4 | |
| Toggle switch | `768:17009` | 16 | |
| Tooltip | `355:1499` | 1 | |
| User | `240:1546` | 2 | |

Legacy, superseded by the table components but still present and still used by the
old Kravmatrise screens: `Column header` `605:4199`, `Column` `605:4265`. The old
`Cell` `605:4204` no longer exists.

## Icons

Page 🖼️ Ikoner `692:15086`, three sections:

| Section | ID | Size | Notes |
|---|---|---|---|
| Material Symbols 20px | `692:15087` | 20 | **The one you want.** This file mostly uses 20px icons |
| Material Symbols 48px | `692:15090` | 48 | Static, one icon |
| Ikoner Pulse | `692:15091` | 36 | Bespoke Pulse icons, not Material Symbols |

Icons are Google Material Symbols, style `outlined`, weight 400, grade 0, no
fill. Each is a 20×20 component containing one `VECTOR` named `Vector` with its
fill bound to `Icon/Primary`.

When a design needs an icon the file does not have, import it — recipe in
`build-patterns.md` under *The icon you need is not in the icon set*. There is
also a bulk-import plugin at `figma-plugins/material-symbols-importer` in the
`Awesheim-portfolio` repo, but a single icon is faster to script.

## Page chrome anatomy

Now a component — `Pulse - App shell` `974:20406`, 2560×1440. Instance it rather
than rebuilding:

- background fill `Surface/Inverse`
- glow ellipse using the `Pulse - Blue glow` effect style
- sidebar 196 wide at `x: 24`
- content area exposed as `Slot`
- the sidebar menu sits at ~30% of viewport height, achieved with three
  equal-Fill spacer rectangles (one above the menu, two below). They are named
  `Spacer 1fr … do not export` — implement as `margin-top: 30vh`, not as elements.

## Known asymmetries that are intentional

Do not report these as bugs; the owner has confirmed each.

| Component | Behaviour |
|---|---|
| `Input` | `Show error message` is wired only on the 4 Error variants — the error message only exists there |
| `Table cells` | The `Header` variant deliberately has no slot; it uses `Heading` + `Show icon` instead |
| `Table cells` | Every cell draws a right-edge divider. On the last column it sits under the table border — that is how the trailing divider is hidden |
| `Portal big navigation` | Fixed 280×128 on both axes, to force short app names |
| `Modal` | No `Closable` toggle — the close button is always present |

## Still open

| Component | Problem |
|---|---|
| `CV score` `282:2491` | `Label`, `Kunde`, `Prosjekt`, `Show label` are wired in only 4 of 8 variants — the Large/Small split. Accepted: it is a single-use component for the CV-quality list. Do not reuse it elsewhere without fixing this first. |

## Fixed since the previous version of this file

Do not re-report any of these.

- **Accordion** — `Title`, `Qualifier`, `Show qualifier`, `Show tag` now bound in
  all four variants; values persist across every state change.
- **Input** — the dead `Text#768:329` property (default `-14`) removed; `Påkrevd`
  now wired on the Disabled variants; a **Readonly** Type added (28 variants).
  Readonly means submitted-and-validated-but-not-editable; Disabled means not
  applicable and not submitted.
- **Icon button** — the dead `Label` / `Show left icon` / `Show right icon` /
  `Right icon` properties are gone; a working `Icon` instance-swap added.
- **Toggle switch** — no longer in an error state; `componentPropertyDefinitions`
  reads normally. `Label` text property added.
- **Checkbox**, **Radio button** — `Label` text property added.
- **Tabs** — the redundant `Property 1` axis removed; `Amount` is the only axis.
- **Tooltip** — `Text` property added; it no longer carries one hardcoded sentence.
- **Modal** — `Heading`, `Qualifier` and `Show Qualifier` added.
- **User** — the unnamed `Variant2` renamed; the axis is now `Type: Default | Hover`.
- **Menu item → Navigation link**, **Portal big menu → Portal big navigation** —
  the `Menu item` / `Menu-item` name collision is resolved.
- **Consultant list → List item** — generalised: `Text`, `Qualifier`,
  `Show qualifier`, and a `Slot` for trailing content that hugs its width.
  `Type: Default | Hover | Selected`. Use on `Surface/Primary` only.
- **Table**, **Table row**, **Table cells** — the table primitive the retro asked
  for now exists. Row heights equalise themselves.
- **Pulse - App shell** — the page shell is a component; the glow is an effect
  style rather than a number living in one instance.
- Every non-icon component now has a description.

## Defaults that are load-bearing

The principle stands: **a default that looks wrong in isolation may be
load-bearing in context.** Check instance usage before touching one, and say what
you found.

The specific example previously recorded here — `Tags` defaulting to `-14` as
real score data across CV-kvalitet — no longer matches the file: the `Tags` text
default now reads `Lorem ipsum`. Instances that override the text are unaffected,
but if you are working in CV-kvalitet, verify those scores still read correctly
before assuming nothing regressed.

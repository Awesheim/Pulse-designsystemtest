# Pulse / CV-Bot file inventory

File-specific facts observed in `CV-Bot – Designfiler`, file key
`hrX50tyQXpSb7phWJ5y5Dp`. Node IDs are stable but everything else should be
re-verified — this is a starting point, not a source of truth.

## Pages

| Page | ID |
|---|---|
| 🖋️ Design | `240:1682` |
| 🧱 Tokens | — |
| 🧩 Komponenter | `11:130` |
| 🖼️ Ikoner | `692:15086` |
| ✏️ Skisser | — |

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

**There is no token for 10.** Any `itemSpacing: 10` you find is legacy, baked
into a component. All padding in the library is already token-bound.

## Colour variables in regular use

`Surface/Inverse`, `Surface/Primary`, `Surface/Primary-Light`,
`Surface/Secondary`, `Border/Weak`, `Border/Subtle`, `Text/Primary`,
`Text/Subtle`.

## Text styles

Enumerate rather than trusting these — they are recorded only because they were
used successfully:

- `S:2a72250a84df36da21a2b0ba8e224e0c90e53bbc,` — body copy (Paragraph-M)
- `S:0bdfcc666fdfe43ab091418e44def4295cb5c740,` — small subtle text (dates, meta)

```js
return (await figma.getLocalTextStylesAsync()).map(s => ({ name: s.name, id: s.id }));
```

## Component IDs

| Component | ID | Notes |
|---|---|---|
| Accordion | `606:4676` | closed `606:4674`, open `606:4686` |
| Cell | `605:4204` | 4 variants |
| Card | `755:16012` | slot fixed to auto-layout |
| Modal | `755:16051` | slot fixed to auto-layout |
| Column header | `605:4199` | standalone |
| Column | `605:4265` | standalone |
| Tags | `84:844` | 8 variants |
| Button | `240:695` | 45 variants |
| Icon button | `282:3211` | 45 variants |
| Search | `554:2063` | |
| Select | `687:14876` | |
| Input | `578:2410` | 24 variants |
| Alert | `602:3240` | |
| Menu item | `240:1644` | |
| Text link | `240:1612` | |
| User | `240:1546` | |
| CV score | `282:2491` | |
| Toggle switch | `768:17009` | **in an error state** |

## Page chrome anatomy

Taken from the CV-Kvalitet screen, at 2560×1440:

- background fill `Surface/Inverse`
- glow ellipse — copy the effect array verbatim; the blur is **1000px**
- sidebar 196 wide at `x: 24`
- content card 2292 wide at `x: 244`, fill `Surface/Primary`
- sidebar menu sits one third down, achieved with 1:2 spacer frames

## Known-broken components

Verified by auditing every property against every variant. These are **not**
fixed — treat them as landmines.

| Component | Problem |
|---|---|
| **Toggle switch** `768:17009` | 16 variants covering only 8 unique combinations — every one duplicated. `componentPropertyDefinitions` throws; unreadable via API until fixed in the editor. |
| **Accordion** `606:4676` | `Title`, `Qualifier`, `Show qualifier` are wired only in the *Closed* variant. The header frame is `Title+Tag` in Closed and `Title+icon` in Open, which broke the bindings. Set the layers directly on Open. |
| **Cell** `605:4204` | Three separate slot properties (`Slot#617:99`, `Slot#617:104`, `Slot 2#617:110`) because the slot layer is named `Slot` in two variants and `Slot 2` in the others. Switching variant drops the content. |
| **Icon button** `282:3211` | `Label`, `Show left icon`, `Show right icon`, `Right icon` are wired in none of the 45 variants — dead properties copied from `Button`. Only `Left icon` works. |
| **Input** `578:2410` | `Text#768:329` — the actual field text — is unwired in all 24 variants; you cannot set an input's content by property. `Show error message` works only on the 4 Error variants. `Påkrevd` is missing on Disabled. |
| **CV score** `282:2491` | Large and Small have divergent internals: Large lacks `Kunde`, `Prosjekt`, `Show label`; Small lacks `Label`. |
| **Column header** `605:4199` | Declares a `SLOT` property but contains no slot node. Set the text layer directly. |

Components that passed: `Tags`, `Button`, `Search`, `Select`, `Alert`,
`Text link`, `Menu item`, `Portal big menu`, `Tab`.

## Defaults that are load-bearing

`Tags` uses `-14` as default text on all eight variants, and 20 instances across
CV-kvalitet and the Pulse sketches rely on it as **real score data**. The
`Accordion`'s nested tag defaults to `Danger` for the same reason — a negative
score should read red.

Both look like mistakes in isolation. Neither is. Check instance usage before
touching a default.

## Already fixed (do not re-report)

- `Card` and `Modal` slots given auto-layout; roots changed from fixed to hug.
  A card now grows 251 → 404px with 300px of content.
- `Accordion`'s two closed variants' slots given auto-layout.
- 180 `itemSpacing` values bound to tokens across 18 components. The `10 → S (8)`
  changes were all on nodes with fewer than two visible children, so nothing
  moved in the default state.
- Placeholder text replaced in `Input`, `Search`, `Select`, `Tab`, `Menu-item`
  and `Cell`. Body-copy lorem in `Alert` and `Portal big menu` was left alone —
  it demonstrates text wrapping and is doing a job.

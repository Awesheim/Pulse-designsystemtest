---
name: pulse-figma-design
description: Build, update, or audit screens in Figma using the Pulse / CV-Bot design system through the Figma MCP (use_figma). Use this whenever the user asks to design, mock up, rebuild, or extend a screen, component, table, or layout in Figma with an existing design system — including when they only point at a sketch, name a screen, share a figma.com URL, or say "build this properly with the components". Also use it when auditing a Figma file for token compliance, or when a component behaves in a way the user did not expect (properties that do nothing, content that will not fit, variants that disagree with each other).
---

# Designing in Figma with the Pulse design system

## The one thing to internalise

A component library is a set of **claims**, not guarantees. Components in this
file will accept a property and silently ignore it, expose a slot that cannot
grow, and define the same property three times under three different IDs. The
plugin API reports success for several of these.

So the loop that works is: **probe, build, measure, look**. Never assume a
component does what its property panel advertises, and never trust an edit you
have not seen rendered.

Screenshots are not a formality here. Every real defect found in this file was
invisible in the API response and obvious in the picture.

## Workflow

### 1. Find the reference screen first

Before opening a single component, find the most similar **finished** screen in
the file and read its structure. It answers questions the design system cannot:
page chrome anatomy, layout rhythm, which spacing step is used at which nesting
level.

This step is not optional and it is the one most easily skipped. Building from
the components alone produces a screen that is internally consistent and still
wrong, because a component's internal gap is not the same decision as the gap
between elements on a page.

**Token-bound is not the same as correct.** An audit that proves every value
comes from a variable will happily pass a screen that uses `XXL` where every
comparable screen uses `L`. The reference screen is the only thing that catches
that. Read `references/build-patterns.md` for how to extract a chrome recipe.

### 2. Preflight the components you plan to use

Run the probes in `references/preflight.md` on every component before you design
around it. It takes one tool call and tells you:

- which slots can actually grow (`layoutMode: NONE` means they cannot)
- which properties are wired up in which variants
- what the real fills, strokes, radii, padding and effects are

If a component fails its probe, you have two options: rebuild it by hand from its
own recipe (see `references/build-patterns.md`), or use a different one. Decide
before you build, not halfway through.

### 3. Build

Prefer live instances. Hand-build only what the probe proved unusable, and when
you do, replicate the component's real recipe rather than eyeballing it — read
the fills, strokes, radius, padding, gap and effects off the component and bind
every one of them to the same variable it uses.

Patterns for page shells, cards, tables and content slots are in
`references/build-patterns.md`. API traps that will cost you a call each are in
`references/api-gotchas.md`.

### 4. Audit

Run the audit script in `references/audit.md`. It reports unbound fills and
strokes, text without a text style, and untokenised spacing — scoped to nodes you
authored, skipping component internals you do not own.

Treat a clean audit as necessary, not sufficient. Compare your spacing against
the reference screen from step 1 by hand.

### 5. Look at it

Screenshot every screen you touched, at a size where you can read the labels.
Then go back and fix what you see. Expect to find something — in this file the
picture caught a wrong-variant tag, an unapplied title property, a placeholder
that read as real content, and a glow effect that was four times too weak.

## The five failure modes in this file

| Symptom | Cause | What to do |
|---|---|---|
| Content will not make its container grow | Slot has `layoutMode: NONE` | Give the slot auto-layout, or hand-build the container |
| `setProperties` succeeds but nothing changes | Property is not wired in that variant | Set the layer directly; report the component as broken |
| Same property appears two or three times | Layers named differently across variants | Rename layers to match, rebind; do not paper over it |
| A default reads as real data | Placeholder text like `-14` or `Lorem ipsum` | Check instance usage before changing it — see below |
| Screen looks right, matches no other screen | Built from components, not from a reference | Mirror the nearest finished screen's rhythm |

## Before changing anything shared

Components and their defaults propagate. Before editing one, count who depends
on it:

```js
// how many instances rely on the default vs. override it?
const variantIds = new Set(componentSet.children.map(c => c.id));
for (const p of figma.root.children) {
  await p.loadAsync();
  for (const inst of p.findAllWithCriteria({ types: ['INSTANCE'] })) {
    const mc = await inst.getMainComponentAsync();
    if (mc && variantIds.has(mc.id)) { /* inspect inst */ }
  }
}
```

If finished designs rely on the default, leave it and report it. In this file a
tag default that looked like a mistake (`Danger` type, text `-14`) turned out to
be a score penalty in the CV-Kvalitet screens, where red was correct. Changing it
degraded finished work.

The rule: **a default that looks wrong in isolation may be load-bearing in
context.** Check usage, then decide, and say what you found.

## Scope discipline

Sections of this file contain finished, approved work. Do not read from or write
to them unless asked. When told to build something new, build it beside the
existing work, not on top of it.

## Reference files

- `references/preflight.md` — probes to run before designing around a component
- `references/build-patterns.md` — page shell, card, table and slot recipes
- `references/api-gotchas.md` — Figma plugin API traps that cost a call each
- `references/audit.md` — verification scripts
- `references/pulse-inventory.md` — Pulse tokens, component IDs, known-broken list

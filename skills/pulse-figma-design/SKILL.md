---
name: pulse-figma-design
description: Build, update, or audit screens in Figma using the Pulse / CV-Bot design system through the Figma MCP (use_figma). Use this whenever the user asks to design, mock up, rebuild, or extend a screen, component, table, or layout in Figma with an existing design system — including when they only point at a sketch, name a screen, share a figma.com URL, or say "build this properly with the components". Also use it when auditing a Figma file for token compliance, or when a component behaves in a way the user did not expect (properties that do nothing, content that will not fit, variants that disagree with each other).
---

# Designing in Figma with the Pulse design system

## The one thing to internalise

A component library is a set of **claims**, not guarantees. Components will accept
a property and silently ignore it, expose a slot that cannot grow, and define the
same property under IDs that do not match across variants. The plugin API reports
success for several of these.

So the loop that works is: **read the description, probe, build, measure, look.**
Never assume a component does what its property panel advertises, and never trust
an edit you have not seen rendered.

Screenshots are not a formality here. Every real defect found in this file was
invisible in the API response and obvious in the picture. The MCP screenshot tool
can return the image inline with `enableBase64Response: true` — use that when the
sandbox cannot fetch the returned URL.

## Workflow

### 1. Read the component description — always, first

**Every non-icon component in this file has a description, and they are the
primary documentation.** They are written to be read by an agent before use, and
they record the things that cost the most time to rediscover: required steps,
sizing contracts, hard limits, and which behaviours are intentional rather than
bugs. Reading one costs a fraction of a probe.

```js
const ids = ['1031:2', '1044:317', '886:19537'];        // the components you plan to use
const out = [];
for (const id of ids) {
  const n = await figma.getNodeByIdAsync(id);
  out.push({ name: n.name, description: n.description });
}
return out;
```

To sweep everything at once:

```js
const page = await figma.getNodeByIdAsync('11:130');    // 🧩 Komponenter
await page.loadAsync();
return page.children
  .filter(n => n.type === 'COMPONENT_SET' || n.type === 'COMPONENT')
  .map(n => ({ id: n.id, name: n.name, description: n.description }));
```

Take the descriptions literally. When one says a step is required, it is required
and cannot be designed around — the reasoning is usually recorded alongside it.
When one says a behaviour is intentional, do not "fix" it and do not report it as
a finding.

If a description contradicts what you observe, trust the observation, then say so
and update the description. A stale description is worse than none.

### 2. Find the reference screen

Before building, find the most similar **finished** screen in the file and read
its structure. It answers questions the design system cannot: page chrome
anatomy, layout rhythm, which spacing step is used at which nesting level.

This step is the one most easily skipped. Building from the components alone
produces a screen that is internally consistent and still wrong, because a
component's internal gap is not the same decision as the gap between elements on
a page.

**Token-bound is not the same as correct.** An audit that proves every value comes
from a variable will happily pass a screen that uses `XXL` where every comparable
screen uses `L`. The reference screen is the only thing that catches that.

### 3. Preflight anything the description did not settle

Run the probes in `references/preflight.md`. They tell you which slots can grow,
which properties are wired in which variants, and what the real fills, strokes,
radii, padding and effects are.

If a component fails its probe, decide before you build, not halfway through:
rebuild it by hand from its own recipe (`references/build-patterns.md`), or use a
different one.

### 4. Build

Prefer live instances. Hand-build only what the probe proved unusable, and when
you do, replicate the component's real recipe rather than eyeballing it — read
the fills, strokes, radius, padding, gap and effects off the component and bind
every one of them to the same variable it uses.

Patterns for page shells, cards, tables and content slots are in
`references/build-patterns.md`. API traps that will cost you a call each are in
`references/api-gotchas.md` — that file is long because each entry was paid for.

### 5. Audit

Run the audit script in `references/audit.md`. It reports unbound fills and
strokes, text without a text style, and untokenised spacing — scoped to nodes you
authored, skipping component internals you do not own.

Treat a clean audit as necessary, not sufficient. Compare your spacing against the
reference screen by hand.

### 6. Look at it

Screenshot every screen you touched, at a size where you can read the labels.
Then go back and fix what you see. Expect to find something — in this file the
picture caught a wrong-variant tag, an unapplied title property, a placeholder
that read as real content, and a glow effect that was four times too weak.

## Failure modes worth recognising

| Symptom | Cause | What to do |
|---|---|---|
| `setProperties` succeeds but nothing changes | Property is not wired in that variant | Set the layer directly; report the component as broken |
| Content will not make its container grow | Slot has `layoutMode: NONE`, or the frame was resized after being set to hug | Give the slot auto-layout; re-apply hug **after** any `resize()` |
| A size you set is silently ignored | You are resizing a sublayer of an instance | Not possible — Figma refuses it. Restructure or hand-build |
| Row or cell heights do not equalise | Something in the chain is fixed rather than hug/fill | Every cell Fill height, every row Hug height |
| Switching variant drops slot content | The slot layer resolves to different property IDs per variant | Rename the layers to match and rebind |
| A default reads as real data | Placeholder like `-14` or `Lorem ipsum` | Check instance usage before changing it — see below |
| The icon a design needs is not in the file | Only imported icons exist as components | Import it from Material Symbols into the 20px section — recipe in `build-patterns.md` |
| Screen looks right, matches no other screen | Built from components, not from a reference | Mirror the nearest finished screen's rhythm |

## Before changing anything shared

Components and their defaults propagate. Before editing one, count who depends on
it:

```js
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

## Keep the descriptions current

If you change a component's behaviour, update its description in the same session.
The descriptions are the contract every later agent reads first, and they are only
worth reading while they are true.

Writing them has one trap of its own: the `description` setter HTML-escapes ASCII
quotes and apostrophes, so `"` is stored as `&quot;`. Use typographic quotes
(`“ ” ’`) or no quotes at all. See `references/api-gotchas.md`.

## Scope discipline

Sections of this file contain finished, approved work. Do not read from or write
to them unless asked. When told to build something new, build it beside the
existing work, not on top of it.

## Reference files

- `references/preflight.md` — probes to run before designing around a component
- `references/build-patterns.md` — page shell, card, table and slot recipes
- `references/api-gotchas.md` — Figma plugin API traps that cost a call each
- `references/audit.md` — verification scripts
- `references/pulse-inventory.md` — Pulse tokens, component IDs, current state

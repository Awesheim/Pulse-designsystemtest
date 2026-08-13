# Preflight: probe components before designing around them

Run these before you commit to a component. Each is one `use_figma` call and
saves you from discovering the problem after the layout is built.

## 0. Read the description first

Every non-icon component in this file has one, and it is cheaper than any probe.
The descriptions record required steps, sizing contracts, hard limits, and which
odd-looking behaviours are deliberate — most of what the probes below would tell
you, already written down.

```js
const page = await figma.getNodeByIdAsync('11:130');   // 🧩 Komponenter
await page.loadAsync();
return page.children
  .filter(n => n.type === 'COMPONENT_SET' || n.type === 'COMPONENT')
  .filter(n => n.description)
  .map(n => ({ id: n.id, name: n.name, description: n.description }));
```

Probe **in addition** when the description does not settle the question, when you
are about to hand-build a replacement, or when what you see disagrees with what it
says. If it disagrees, the file is the truth — fix the description afterwards.

## 1. Which slots can actually grow

A slot with `layoutMode: NONE` will not resize to its content. `slot.resize()`
returns success and does nothing. `layoutSizingVertical = 'HUG'` on the parent
also does nothing. This is the single most expensive trap in this file.

```js
const page = await figma.getNodeByIdAsync('11:130'); // Komponenter
await page.loadAsync();

const slots = [];
const roots = [
  ...page.findAllWithCriteria({ types: ['COMPONENT_SET'] }),
  ...page.findAllWithCriteria({ types: ['COMPONENT'] })
       .filter(c => !c.parent || c.parent.type !== 'COMPONENT_SET')
];
for (const root of roots) {
  for (const s of root.findAll(n => n.type === 'SLOT')) {
    slots.push({ owner: root.name, slotId: s.id, name: s.name,
      layoutMode: s.layoutMode,            // 'NONE' => rigid
      sizV: s.layoutSizingVertical, sizH: s.layoutSizingHorizontal,
      h: Math.round(s.height) });
  }
}
return slots;
```

### Prove it rather than trusting the flag

Instance the component off-canvas, push oversized content into its slot, measure,
then delete the probe. This is the only way to be sure.

```js
const probe = figma.createFrame();
probe.name = 'TEMP probe'; probe.x = -4000; probe.y = -4000;
probe.resize(600, 600);
page.appendChild(probe);

const inst = (await figma.getNodeByIdAsync('755:16012')).createInstance();
probe.appendChild(inst);
const slot = inst.findOne(n => n.type === 'SLOT');
const before = { slotH: Math.round(slot.height), instH: Math.round(inst.height) };

const filler = figma.createFrame();
filler.resize(200, 300);
slot.appendChild(filler);
filler.layoutSizingHorizontal = 'FILL';

const after = { slotH: Math.round(slot.height), instH: Math.round(inst.height) };
probe.remove();                       // always clean up
return { before, after, grew: after.instH > before.instH };
```

### Fixing a rigid slot

```js
slot.layoutMode = 'VERTICAL';
slot.layoutSizingHorizontal = 'FILL';
slot.layoutSizingVertical = 'HUG';
owner.layoutSizingVertical = 'HUG';   // the container must hug too
```

Set the slot's sizing before the owner's, otherwise you briefly have a FILL child
inside a HUG parent and Figma resolves it in a way you did not intend.

## 2. Which properties are actually wired, per variant

A property is defined on the **set**, but each **variant** must contain a layer
that references it. When a layer is renamed or a variant is rebuilt by hand, the
reference is lost — and `setProperties` will still report success.

```js
const collectRefs = (root) => {
  const refd = new Set();
  for (const n of [root, ...root.findAll(() => true)]) {
    const r = n.componentPropertyReferences;
    if (r) for (const k of Object.keys(r)) if (r[k]) refd.add(r[k]);
  }
  return refd;
};

const gaps = [];
for (const set of page.findAllWithCriteria({ types: ['COMPONENT_SET'] })) {
  let defs;
  try { defs = set.componentPropertyDefinitions || {}; }
  catch (e) { gaps.push({ set: set.name, error: 'set has existing errors' }); continue; }

  const props = Object.keys(defs).filter(k => defs[k].type !== 'VARIANT');
  if (!props.length) continue;

  for (const v of set.children) {
    const refd = collectRefs(v);
    const missing = props.filter(p => !refd.has(p));
    if (missing.length) gaps.push({ set: set.name, variant: v.name, missing });
  }
}
return gaps;
```

Read the result like this:

- **missing in every variant** — a dead property, usually copied from a sibling
  component. Nothing you do in the property panel will ever affect the design.
- **missing in some variants** — the trap. It works while you develop on one
  variant, then silently stops when you switch.
- **the same property name appearing under several IDs** — the layer is named
  differently across variants, so Figma could not merge them. Swapping variant
  drops the content.
- **the set throws on `componentPropertyDefinitions`** — duplicated variant
  combinations. The set cannot be read by the API at all until a human fixes it
  in the editor.

Partial wiring is not automatically a defect. Several components in this file are
deliberately asymmetric — a property that only makes sense in one state is bound
only there. Check `pulse-inventory.md` and the component's own description before
reporting one; the intentional cases are listed.

Note that a missing binding is not always a missing *reference* — sometimes the
layer itself was deleted from that variant, and the fix is to restore the layer
before rebinding. `Input`'s required tag was missing from the Disabled variants
for exactly this reason.

## 3. Read a component's real recipe

Needed whenever you must hand-build a replacement. Never eyeball these values.

```js
const n = await figma.getNodeByIdAsync('<component id>');
return {
  fills: n.fills, strokes: n.strokes, strokeWeight: n.strokeWeight,
  strokeAlign: n.strokeAlign, radius: n.cornerRadius, effects: n.effects,
  layoutMode: n.layoutMode, itemSpacing: n.itemSpacing,
  padding: [n.paddingTop, n.paddingRight, n.paddingBottom, n.paddingLeft],
  bound: n.boundVariables          // tells you WHICH token each value uses
};
```

`boundVariables` is the important field — it names the variable behind each
value, which is what you need in order to rebuild with the same tokens rather
than the same numbers.

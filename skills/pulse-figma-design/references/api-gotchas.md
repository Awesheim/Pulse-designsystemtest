# Figma plugin API traps

Each of these cost a wasted `use_figma` call during the Kravmatrise build. They
are cheap to avoid once you know them.

## `getNodeByIdAsync` needs the node's page loaded

It resolves nodes on the **current** page. For anything on another page, load
that page first or you get `null` and a confusing downstream `TypeError`.

```js
const comps = await figma.getNodeByIdAsync('11:130');
await comps.loadAsync();                     // now components resolve
const card = await figma.getNodeByIdAsync('755:16012');
```

Loading two pages in one call is fine and usually what you want when comparing a
design against its components.

## Not every node type has every property

Walking an ancestor chain will eventually hit a `SECTION`, which has no
`layoutSizingHorizontal`. Reading it throws and kills the whole call. Guard reads
when you do not control the node type:

```js
const g = (n, k) => { try { return n[k]; } catch (e) { return undefined; } };
```

## `componentPropertyDefinitions` throws on a broken set

A component set with duplicated variant combinations is in an error state, and
reading its property definitions raises `Component set has existing errors`.
Wrap it — otherwise one bad component aborts an audit of the whole library:

```js
let defs;
try { defs = set.componentPropertyDefinitions || {}; }
catch (e) { /* record and continue */ }
```

## Property IDs cannot be guessed

Properties are keyed `Name#123:456`. The numeric suffix is unique per component
and there is no way to derive it. Read the definitions first, then match by
prefix:

```js
const defs = set.componentPropertyDefinitions;
const titleKey = Object.keys(defs).find(k => k.startsWith('Title#'));
inst.setProperties({ [titleKey]: 'My title' });
```

Variant properties are the exception — those are plain names (`Type`, `Size`,
`State`) and can be set directly.

## `setProperties` reports success when nothing happened

There is no error when a property is not wired in the current variant. Read the
value back, or screenshot. This is the highest-value habit in the whole file:

```js
inst.setProperties({ [titleKey]: 'My title' });
const t = inst.findOne(n => n.type === 'TEXT' && n.name === 'Item title');
if (t.characters !== 'My title') { /* fall back to setting the layer directly */ }
```

## Fonts must be loaded per style segment

A text node with mixed styling needs each segment's font loaded, not just the
node's primary font:

```js
for (const seg of t.getStyledTextSegments(['fontName'])) {
  await figma.loadFontAsync(seg.fontName);
}
t.characters = 'New text';
```

## `itemSpacing` is ignored under `SPACE_BETWEEN`

A frame using `primaryAxisAlignItems: 'SPACE_BETWEEN'` keeps whatever stale
`itemSpacing` it had. Values like `242` or `174` are leftovers, not decisions —
do not "fix" them by binding them to a token, and do not treat them as findings.
Check the alignment before judging a gap:

```js
const inert = n.primaryAxisAlignItems === 'SPACE_BETWEEN';
```

## Binding a component's spacing propagates into its instances

After binding `itemSpacing` on a component, instances of it inherit the binding —
including instances nested inside other components. An audit run before and after
will show more fixed than you explicitly touched. That is correct behaviour, not
a miscount.

## `INSTANCE_SWAP` defaults are node IDs

The default value of an instance-swap property is a raw node ID like `"234:487"`,
not a name. Resolve it before reporting it to a human:

```js
const icon = await figma.getNodeByIdAsync(defs[key].defaultValue);
return icon ? icon.name : defs[key].defaultValue;
```

## Text styles are applied asynchronously by ID

`setTextStyleIdAsync` takes the raw style ID string, including its trailing
comma. Enumerate the available styles rather than hardcoding IDs you saw once:

```js
const styles = await figma.getLocalTextStylesAsync();
return styles.map(s => ({ name: s.name, id: s.id }));
```

## Clean up probe nodes

Anything you create to measure must be removed in the same call, including on the
failure path. A stray `TEMP probe` frame left at `-4000,-4000` is invisible in
screenshots and shows up later as a mystery node.

## Deleting children while iterating

`slot.children` is live. Copy before removing:

```js
for (const c of [...slot.children]) c.remove();
```

That works for content **you** put there. It does not work for a slot's default
content — see below.

## Slot default content can only be removed one child per call

Removing one child of a slot's *default* content invalidates every remaining
sibling for the rest of that execution. The children array still reports them,
but each handle raises `Node with id … not found`. Re-fetching the slot from the
instance root does not help, and removing from the end instead of the front does
not help either.

The handles are valid again in the **next** `use_figma` call. So clearing a slot
with three default children takes three calls. Removals in *different* slots can
be done in the same call — the invalidation is per slot.

```js
// one pass; call again for the next child
const kids = slot.children;
if (kids.length) kids[kids.length - 1].remove();
```

This is a scripting constraint only. In the editor each action is its own
transaction, so a human clearing a slot sees none of it.

Related: a hidden slot or a boolean-hidden layer is **removed from the instance
tree entirely**, not merely set invisible. Test for the node's existence rather
than reading `.visible`:

```js
const shown = name => { const n = inst.findOne(x => x.name === name); return !!(n && n.visible); };
```

## Sublayers of an instance cannot be resized

Any size change to a node inside an instance is refused, and the three routes fail
differently — which makes it easy to believe one worked:

| Call | Result |
|---|---|
| `node.resize(w, h)` | returns normally, size unchanged |
| `node.resizeWithoutConstraints(w, h)` | returns normally, size unchanged |
| `node.minWidth = w` | throws `This property cannot be overridden in an instance` |

The same node as a free-standing instance resizes fine, so the boundary is the
parent instance, not the node. If a layout needs per-child sizing inside a
component, the component has to expose it another way or be hand-built.

## `clone()` drops component property references

Cloning a variant to make a new one loses every `componentPropertyReferences`
binding on it and its descendants. The clone looks correct and every property is
silently unwired. Copy the references across explicitly by walking both trees in
parallel — the structures are identical:

```js
const pair = (a, b) => {
  const aRefs = a.componentPropertyReferences || {};
  if (Object.keys(aRefs).length) b.componentPropertyReferences = { ...aRefs };
  if (a.children && b.children && a.children.length === b.children.length)
    a.children.forEach((c, i) => pair(c, b.children[i]));
};
pair(sourceVariant, clonedVariant);
```

## There is no `figma.createSlot`

Slots cannot be created from scratch. Clone one out of a component that already
has it, clear its children, then bind it to a new property:

```js
const donor = (await figma.getNodeByIdAsync('687:15037')).findOne(n => n.type === 'SLOT');
const slot = donor.clone();
container.appendChild(slot);
for (const c of [...slot.children]) c.remove();

const key = comp.addComponentProperty('Rows', 'SLOT', '');   // '' is required
slot.componentPropertyReferences = { slotContentId: key };
```

`addComponentProperty` with a `SLOT` type rejects `null` and rejects being called
with no third argument — it validates `defaultValue` against the boolean/string
union regardless of type. Pass an empty string.

A cloned slot brings the donor's padding, corner radius and fills with it. Clear
them or you inherit a stray 4px radius and a background you did not ask for.

## Accessing an unknown property on the `figma` global throws

`typeof figma.createSlot` does not return `'undefined'` — it raises
`no such property 'createSlot' on the figma global object`. Feature-detect inside
a `try`, or the probe itself kills the call.

The same applies to node types: `findAll` does not exist on a `TEXT` node, so a
recursive walk that calls it unguarded dies partway.

## `counterAxisAlignItems` has no `STRETCH`

It accepts only `MIN | MAX | CENTER | BASELINE`. To stretch children across the
counter axis, set each child's own `layoutSizingVertical` (or `Horizontal`) to
`FILL`. There is no parent-level stretch.

## Component descriptions are HTML-escaped on write

The `description` setter escapes ASCII quotes and apostrophes: `"` is stored as
`&quot;` and `'` as `&#39;`, and they render that way in Dev Mode. Reading the
value back and rewriting it escapes the ampersand again, so a description edited
twice ends up as `&amp;amp;quot;`.

Use typographic characters, which pass through untouched:

```js
const LQ = String.fromCharCode(8220), RQ = String.fromCharCode(8221);
const AP = String.fromCharCode(8217);
```

To repair an already-mangled description, unwind the `&amp;` layers first, then
convert the remaining entities to typographic characters.

## `preferredValues` works on SLOT properties

A slot can suggest a component in the fill-slot UI. Local unpublished components
have real keys, so this works before publishing:

```js
comp.editComponentProperty(rowsKey, {
  preferredValues: [{ type: 'COMPONENT_SET', key: rowSet.key }]
});
```

`editComponentProperty` is also how you rename a property. Renaming updates every
reference automatically, including in other components that consume it — verified
across 45 variants and two consumer components with no stale references left.

## Read node values before you delete their ancestor

Object literals evaluate after the statements above them. This returns an error,
not a measurement:

```js
probe.remove();
return { height: inst.height };   // inst is gone
```

Capture what you need into plain values first, then clean up.


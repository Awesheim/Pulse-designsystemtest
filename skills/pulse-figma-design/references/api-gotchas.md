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

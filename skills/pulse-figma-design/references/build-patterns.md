# Build patterns

Recipes that worked, with the reasoning behind them.

## Setup: variables, colours, text styles

Every build starts here. Bind colours through variables rather than setting
literal fills — a literal fill passes visual inspection and fails the audit.

```js
const cols = await figma.variables.getLocalVariableCollectionsAsync();
const V = {};
for (const c of cols) {
  for (const id of c.variableIds) {
    const v = await figma.variables.getVariableByIdAsync(id);
    if (v) V[v.name] = v;
  }
}

// bind a colour variable into a paint
const paint = name => figma.variables.setBoundVariableForPaint(
  { type: 'SOLID', color: { r: 0, g: 0, b: 0 } }, 'color', V[name]
);

frame.fills = [paint('Surface/Primary-Light')];
frame.strokes = [paint('Border/Weak')];
frame.setBoundVariable('itemSpacing', V['XL']);
```

Text needs its font loaded before `characters` can be set, and the text style
applied after the node exists:

```js
await figma.loadFontAsync({ family: 'Mona Sans', style: 'Regular' });
const t = figma.createText();
t.fontName = { family: 'Mona Sans', style: 'Regular' };
t.characters = 'Some label';
parent.appendChild(t);
await t.setTextStyleIdAsync(styleId);   // async, and after appending
t.fills = [paint('Text/Primary')];      // style may carry its own colour — set after
t.textAutoResize = 'HEIGHT';
t.layoutSizingHorizontal = 'FILL';      // only valid once inside an auto-layout parent
```

## Extracting a chrome recipe from a reference screen

Do this before building any new screen. Read the finished screen's structure and
copy it exactly, including effects.

```js
const ref = await figma.getNodeByIdAsync('<reference screen id>');
const rows = [];
const walk = (n, d) => {
  rows.push({ d, name: n.name, type: n.type,
    x: Math.round(n.x), y: Math.round(n.y),
    w: Math.round(n.width), h: Math.round(n.height),
    lm: n.layoutMode, gap: n.itemSpacing,
    bound: n.boundVariables, effects: n.effects, fills: n.fills });
  if (d < 3 && 'children' in n) n.children.forEach(c => walk(c, d + 1));
};
walk(ref, 0);
return rows;
```

**Copy effects verbatim — never estimate them.** A drop shadow or glow guessed by
eye will be close enough to look plausible and wrong enough to be off-brand. The
Pulse background glow is a 1000px blur; a 400px guess looked reasonable in
isolation and was obviously wrong beside the real screen.

The cheapest reliable method is to clone the effect array straight across:

```js
target.effects = JSON.parse(JSON.stringify(sourceEllipse.effects));
```

## Hand-building a component you cannot instance

Only when preflight proved the component unusable. Read its recipe first (see
`preflight.md`), then reproduce every property through the same variables.

A card in this system is:

```js
const card = figma.createAutoLayout('VERTICAL', { name: 'Card/…' });
card.fills = [paint('Surface/Primary-Light')];
card.strokes = [paint('Border/Weak')];
card.strokeWeight = 1;
card.strokeAlign = 'INSIDE';
card.setBoundVariable('topLeftRadius', V['S']);      // and the other three corners
card.setBoundVariable('paddingTop', V['XL']);        // and the other three sides
card.setBoundVariable('itemSpacing', V['XXL']);
card.effects = JSON.parse(JSON.stringify(cardComponent.effects));  // shadow
```

Note the gap: the component's own `itemSpacing` separates its **header from its
slot** — two blocks. If your hand-built version has more children than that, that
value is probably not the right gap between all of them. Check the reference
screen for what the real card uses.

## Tables

There is no table primitive, so assemble rows and cells and handle two things the
components do not.

### Cell borders without doubling

Setting all four sides on every cell produces 2px seams where cells meet. Set
sides individually:

```js
cell.strokes = [paint('Border/Subtle')];
cell.strokeWeight = 1;
cell.strokeAlign = 'INSIDE';
cell.strokeTopWeight = 0;
cell.strokeBottomWeight = 1;                       // always
cell.strokeLeftWeight = 0;
cell.strokeRightWeight = isLastColumn ? 0 : 1;     // except last column
```

### Equalising cell heights within a row

Cells hug their content, so the three cells in a row end up different heights.
Let the tallest hug and make the others fill. Re-run this after every content
change, resetting to HUG first so the measurement is honest:

```js
for (const row of rows) {
  row.children.forEach(c => { c.layoutSizingVertical = 'HUG'; });   // reset
  const heights = row.children.map(c => c.height);
  const tallest = heights.indexOf(Math.max(...heights));
  row.children.forEach((c, i) => {
    if (i !== tallest) c.layoutSizingVertical = 'FILL';
  });
}
```

This keeps columns flush at any row height without hardcoding a single value.

## Filling a slot on an instance

When the slot has auto-layout, appending works normally:

```js
const slot = instance.findOne(n => n.type === 'SLOT');
for (const c of [...slot.children]) c.remove();     // drop the placeholder
const content = figma.createAutoLayout('VERTICAL', { name: 'Content' });
slot.appendChild(content);
content.fills = [];
content.setBoundVariable('itemSpacing', V['S']);
content.resize(slot.width, content.height);
```

Measure afterwards. If the slot's height did not change, it was rigid and the
preflight probe would have told you.

## Order of operations that avoids fights with auto-layout

1. Create the node
2. Append it to its parent
3. `resize()` to a sensible starting size
4. Then set `layoutSizingHorizontal` / `layoutSizingVertical`

Setting sizing before the node is inside an auto-layout parent throws. Resizing
after setting FILL is silently overridden.

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

**There is a table primitive now — use it.** Three components, read their
descriptions before building:

| Component | ID | What it is |
|---|---|---|
| `Table` | `1031:2` | The wrapper. Shadow, corner radii, border and clipping baked in. Two slots. |
| `Table row` | `1044:317` | One row. `Columns` 2–6 × `Type` Main fill / Alternate fill / Header. |
| `Table cells` | `886:19537` | The cell primitive. `Main fill`, `Alternate fill`, `Header`. |

The old manual recipes — per-cell border sides, and measuring rows to equalise
heights by hand — are obsolete. The cells carry their own right-edge divider, and
row heights equalise themselves. Do not reintroduce either.

### Building a table

```js
const table = (await figma.getNodeByIdAsync('1031:2')).createInstance();
parent.appendChild(table);
table.layoutSizingHorizontal = 'FILL';   // or leave fixed and set a width

// header slot takes CELLS directly — it is already a horizontal row
const hSlot = table.children[0].children[0];
// rows slot takes one child per body row
const rSlot = table.children[1].children[0];

const row = (await figma.getNodeByIdAsync('1044:317')).children
  .find(v => v.name === 'Columns=3, Type=Main fill').createInstance();
rSlot.appendChild(row);
row.layoutSizingHorizontal = 'FILL';     // REQUIRED — see below
```

### The two rules that matter

**Every row instance must be set to Fill width after placing.** Instances arrive
at a fixed 768px. Figma stores sizing on the instance, not the component, so a
component cannot default to Fill — this cannot be designed away. A row you forget
stays 768 while the table grows around it.

**Row heights equalise on their own.** Every cell is Fill width and Fill height,
every row is Hug height. The tallest cell's content sets the row height and the
others stretch to match. Never set a row or cell height by hand; doing so breaks
the mechanism that makes multi-line content work.

### Equal columns only

Cell widths **cannot** be overridden inside a `Table row` instance. `resize()` and
`resizeWithoutConstraints()` are silently ignored and `minWidth` throws
`This property cannot be overridden in an instance`. All columns in a row are
therefore the same width.

For a table that genuinely needs uneven columns, skip `Table row` and build the
row by hand — a free-standing cell instance *can* be resized:

```js
const row = figma.createAutoLayout('HORIZONTAL', { name: 'Row' });
rSlot.appendChild(row);
row.fills = []; row.itemSpacing = 0; row.clipsContent = false;
row.layoutSizingHorizontal = 'FILL';
row.layoutSizingVertical = 'HUG';

cells.forEach((c, i) => {
  row.appendChild(c);
  c.layoutSizingVertical = 'FILL';                 // always — this equalises heights
  if (i === 0) { c.layoutSizingHorizontal = 'FIXED'; c.resize(368, c.height); }
  else c.layoutSizingHorizontal = 'FILL';          // the rest split the remainder
});
```

Fixing only the narrow column and leaving the others on Fill is the trick: three
columns in a 1716px row come out 368 / 674 / 674 without computing anything.

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
4. **Then** set `layoutSizingHorizontal` / `layoutSizingVertical`

Setting sizing before the node is inside an auto-layout parent throws. Resizing
after setting FILL is silently overridden.

**`resize()` sets both axes to fixed.** This is the expensive half of the rule.
Calling `resize()` on an auto-layout frame flips `primaryAxisSizingMode` and
`counterAxisSizingMode` to `FIXED`, so any hug you configured beforehand is
silently discarded:

```js
comp.layoutMode = 'HORIZONTAL';
comp.counterAxisSizingMode = 'AUTO';   // hug height
comp.resize(768, 44);                  // ← just undid it; height is now fixed
```

The symptom is a container that refuses to grow with its content, which reads as
a broken component rather than a sizing mistake. It cost a full round of
misdiagnosis on `Table row`, where every variant was pinned at 44px and
multi-line text overflowed instead of pushing the row taller. Set the sizing modes
**after** the resize:

```js
comp.resize(768, 44);
comp.primaryAxisSizingMode = 'FIXED';  // fixed width
comp.counterAxisSizingMode = 'AUTO';   // hug height
```

Changing `layoutMode` has a related effect: the primary and counter axes swap, so
a frame that was hug-height as a vertical stack becomes hug-**width** when you
turn it horizontal. Re-assert both axes after any `layoutMode` change.

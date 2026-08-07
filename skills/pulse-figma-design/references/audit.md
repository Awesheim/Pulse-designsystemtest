# Auditing your own work

Run this on anything you built before calling it done. It scopes itself to nodes
you authored and skips component internals, which belong to the library rather
than to you.

## The audit

```js
const roots = ['<screen id>', '<screen id>', '<screen id>'];
const out = {};

// anything inside an INSTANCE is the component's business, not yours
const inInstance = n => {
  let p = n.parent;
  while (p) { if (p.type === 'INSTANCE') return true; p = p.parent; }
  return false;
};

for (const id of roots) {
  const root = await figma.getNodeByIdAsync(id);
  const unboundFills = [], noTextStyle = [], rawSpacing = [];

  for (const n of root.findAll(() => true)) {
    if (inInstance(n)) continue;

    if (Array.isArray(n.fills)) {
      for (const f of n.fills) {
        if (f.type === 'SOLID' && f.visible !== false &&
            !(f.boundVariables && f.boundVariables.color)) {
          unboundFills.push(n.name + ' [' + n.type + ']');
        }
      }
    }
    if (Array.isArray(n.strokes)) {
      for (const s of n.strokes) {
        if (s.type === 'SOLID' && s.visible !== false &&
            !(s.boundVariables && s.boundVariables.color)) {
          unboundFills.push('stroke:' + n.name);
        }
      }
    }
    if (n.type === 'TEXT' && !n.textStyleId) {
      noTextStyle.push(n.name + ' "' + n.characters.slice(0, 24) + '"');
    }
    if (n.layoutMode && n.layoutMode !== 'NONE') {
      const bv = n.boundVariables || {};
      for (const k of ['itemSpacing','paddingTop','paddingRight','paddingBottom','paddingLeft']) {
        if (n[k] > 0 && !bv[k]) rawSpacing.push(n.name + '.' + k + '=' + n[k]);
      }
    }
  }
  out[root.name] = { unboundFills, noTextStyle,
                     rawSpacing: rawSpacing.slice(0, 20), rawSpacingCount: rawSpacing.length };
}
return out;
```

## Reading the result

Expect **zero** unbound fills, zero unbound strokes and zero text without a style
in your own work. Those are unambiguous defects.

Raw spacing needs interpretation. Values baked into library components
(`itemSpacing=10` on `Text link`, `User`, `Menu item`, `Tags`) will surface on the
instance root even though you did not set them. Those belong to the component —
report them, do not silently rewrite them inside someone else's screen.

## What this audit cannot catch

This is the important part. A clean audit proves every value came from a
variable. It says nothing about whether it was the **right** variable.

A screen can pass with `XXL` where every comparable screen in the file uses `L`,
because both are tokens. The audit has no notion of the file's layout rhythm.

So pair it with a manual comparison against the nearest finished screen:

```js
// print the spacing scale down the ancestor chain of a known element,
// for both your screen and the reference screen, then compare by eye
let n = someElement, chain = [];
while (n && n.type !== 'PAGE') {
  const bv = (n.boundVariables || {}).itemSpacing;
  chain.unshift(n.name + (n.layoutMode !== 'NONE'
    ? ' gap=' + n.itemSpacing + (bv ? '/' + bv.id : ' UNBOUND') : ''));
  n = n.parent;
}
return chain;
```

If your nesting levels read `32 / 32 / 16 / 32` and the reference reads
`64 / 24 / 24 / 16`, you have a problem the audit will never report.

## Before you finish

- Screenshot every screen you touched and actually look at it
- Confirm you did not write into finished sections you were not asked to touch
- If you changed anything shared, report who else depends on it

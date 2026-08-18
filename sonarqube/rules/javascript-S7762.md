# `javascript:S7762` / `javascript:S7768` — `ChildNode.remove()` / `replaceWith()`

Sonar asks for `child.remove()` instead of `parent.removeChild(child)` (S7762) and
`old.replaceWith(new)` instead of `parent.replaceChild(new, old)` (S7768). Both live in the legacy
`xwiki-platform-web-war` scripts.

## The only thing that decides the fix

**Is the receiver provably `child.parentNode`?** The new form always acts on the node's *actual*
parent; the old form acts on whatever object you called it on.

- **Safe, mechanical** — the call already reads `x.parentNode.removeChild(x)` /
  `x.parentNode.replaceChild(y, x)`, or the receiver is a local assigned from `x.parentNode`
  (`panelWizard.js` has `const realParent = el.parentNode;` 30 lines above the flagged call — read the
  declaration, do not assume from the name). Also safe when the receiver is the node the child was
  just appended to in the same statement sequence.
- **Judgement call** — the receiver is a variable holding "the container we think it is in"
  (`prevcolumn.removeChild(dragel)`, `window.parentNode.replaceChild(el, dragel)` in
  `panelWizard.js`'s drag code, where the parent is tracked in globals across `onDrag` calls). The
  invariant holds in practice but the code never states it, and the two forms differ exactly there:
  **`removeChild()`/`replaceChild()` throw `NotFoundError` when the node is not a child; `remove()`
  and `replaceWith()` silently do nothing.** Ship these in the sibling judgement PR.

## Notes

- Prototype.js also defines `Element#remove()` with the same semantics, so an extended element in a
  Prototype-era script behaves identically either way — this is not a reason to drop.
- `wrapper.removeChild(wrapper.firstChild)` → `wrapper.firstChild.remove()` is safe by construction
  (`firstChild` is by definition a child of `wrapper`).
- There is no equivalence to *prove* with `node`: state the receiver/`parentNode` identity per site in
  the PR body instead, quoting the line where the receiver was assigned.

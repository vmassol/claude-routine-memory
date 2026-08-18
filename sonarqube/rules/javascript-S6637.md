# `javascript:S6637` — "The function binding is unnecessary"

`function (…) { … }.bind(this)` where the body never reads `this`. The fix is deleting `.bind(this)`.

## Doing it correctly

- **Verify the body yourself; do not trust the rule alone.** The flagged line is the *closing*
  `}.bind(this)`, so walk back to the matching `function` and confirm no `this` appears in between.
  In the Prototype-era `Ajax.Request` option blocks the same file mixes both kinds — `comments.js`
  has five removable binds and, twenty lines away, a `callback` whose body does
  `this.editing = item;` and must keep its bind. Sonar flags only the former, but a scripted batch
  keyed to `}.bind(this)` would hit both.
- **The flagged text is never unique in the file** (`            }.bind(this),` occurs many times).
  Anchor each edit on the two or three preceding lines, and assert the anchor occurs exactly once.
- Watch the three trailing shapes — `}.bind(this),` (object property), `}.bind(this)` (last
  property, no comma) and `}.bind(this))` / `}.bind(this));` (an argument, so the call's closing
  paren must survive).
- Nothing to prove at runtime: a body that never reads `this` is identical with or without the bind.
  One `node` assertion (`body.bind({})(x) === body(x)`) is enough for the PR body.

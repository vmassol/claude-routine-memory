# `java:S5993` — "Constructors of an `abstract` class should not be declared `public`"

OKF-denylisted as *"reduce a constructor to `protected` — **reduces visibility** → Revapi
`java.method.visibilityReduced`"*. **That is true of the non-`internal` half of the pool and false of
the rest** — the eighth denylist entry found to be a partial-pool claim, and the split axis is the
one that keeps paying: *which subset is the reason actually true of*. Here it is not visibility of the
*member* but visibility of the *package*.

## The split is free — it is in the file PATH

```
'internal' if '/internal/' in component_path else 'public'
```

XWiki's Revapi configuration excludes internal packages from the API check outright, in
`xwiki-commons-tools/xwiki-commons-tool-verification-resources/src/main/resources/revapi.json`:

```
"exclude": [ { "matcher": "java", "match": "type ^org.xwiki.**.internal.** {}" },
             { "matcher": "java", "match": "type ^org.xwiki.**.test.** {}" },
             { "matcher": "java", "match": "type ^com.xpn.xwiki.**.internal.** {}" } ]
```

so an `internal` class is not published API and `revapi:check` has nothing to say about it. **`.**.`
in that pattern matches ZERO segments**: `com.xpn.xwiki.internal.event.AbstractXObjectEvent` (no
segment between `xwiki` and `internal`) is excluded too — 15 oldcore sites of that exact shape passed
`revapi:check`. Do not write those off on the pattern alone; the build settles it.

Sonar's own `message` is uniform ("Change the visibility of this constructor to `protected`.") so it
does not split the pool — the path does.

## Why the internal subset breaks nothing at all (not just nothing Revapi checks)

For an **abstract** class there are exactly two ways to reach a constructor, and
[JLS §6.6.2.2](https://docs.oracle.com/javase/specs/jls/se21/html/jls-6.html#jls-6.6.2.2) permits
both on a `protected` constructor **from any package**: a `super(...)` call in a subclass, and an
anonymous class instance creation `new AbstractX(...){…}`. The one form `protected` would forbid,
`new AbstractX(...)`, is already illegal — the class is abstract. So no *compilable* caller can exist
that the change breaks, in any package, in any module.

That is a proof, not an argument, and it is worth the 60 seconds it takes to run: two packages, one
`javac`, quoted in the PR body. It is what turns "this reduces visibility" from a reviewer objection
into a settled point.

## Mechanics

- The flagged line is uniformly `    public <ClassName>(`; the fix is `public ` → `protected `, one
  token, nothing else on the line.
- **The line grows by 3 characters**, so re-check 120 columns — 2 of 81 sites needed the parameter
  list re-split onto a different continuation line (the list itself unchanged).
- Assert the class really is `abstract` (`abstract\s+class\s+<Name>`) and the package really is
  internal before writing, per file.
- No import, no body, no signature and no `@since` — the constructor's descriptor is unchanged, so
  there is nothing to document.

## Outcome

The commons half (20 sites, 16 classes) **merged uncommented** within hours of opening
(`xwiki/xwiki-commons#1926`) — the eighth denylist rescue to land that way. Platform (59) and
rendering (2) were still open at that point. So the JLS-proof framing works on a reviewer: state the
two independent reasons (the `revapi.json` exclusion *and* the language argument) and the visibility
reduction stops being the question.

## Drop condition

**Non-`internal` packages — permanent.** There the entry's original reason stands: it is a real
published-API visibility reduction and Revapi rejects it. That is the majority of the pool and it
should stay open rather than be accepted.

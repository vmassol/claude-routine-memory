# `java:S4973` — "Strings and Boxed types should be compared using `equals()`"

Small, permanently-regenerating rule (platform 6 at the time of writing, commons/rendering 0). The
transform looks like the safest thing in the catalogue — `a == CONST` → `CONST.equals(a)` — and it has
exactly one trap, which is invisible unless you open the **constant's declaration**.

## The trap: the constant's value can be `null`

`DocumentSolrMetadataExtractor` reads

```java
if ((type != TypedValue.TEXT && type != TypedValue.STRING) || …)
    … StringUtils.capitalize(type == TypedValue.TEXT ? TypedValue.STRING : type)
```

and `TypedValue` declares

```java
public static final String STRING = "string";
public static final String TEXT = null;      // <- the sentinel meaning "analyzed / no explicit type"
```

So `type != TypedValue.TEXT` is a **null check written through a named constant**, not a String
comparison, and the obvious fix (`TypedValue.TEXT.equals(type)`) throws `NullPointerException` on
every call. Both sites are permanent drops until someone reworks the sentinel.

**Classifier, one line read per site and no snippet reads beyond it:** find the constant's
declaration and look at its initializer. A `null` (or an expression that can be `null`) ⇒ drop;
a literal ⇒ the fix is free. Do this *before* writing the edit — the mistake compiles, passes
Checkstyle and only fails at runtime on a path the module's own tests may not cover.

## The free shapes

* **Two boxed operands, both non-null by construction** — `Objects.equals(a, b)` / `!Objects.equals(a, b)`
  is the safe form: value-based, null-safe, and identical to `==` for every value inside the `Integer`
  cache, which is why the old code worked. Used for `CharacterDiffService`'s four
  `Difference.getDeletedStart()/getAddedStart()/getDeletedEnd()/getAddedEnd()` vs `Difference.NONE`
  comparisons. It compiles whether or not the accessor is actually boxed, so you do not need the
  third-party jar on hand to decide (a cold `~/.m2` will not have it).
* **Two `Boolean` operands where the file already shows the house form** — `XWikiDocument#getMetaDataDiff`
  compared `fromDoc.isHidden() != toDoc.isHidden()` while the four neighbouring comparisons in the
  same method used `ObjectUtils.notEqual(…)`. Copy the neighbour rather than introducing
  `Objects.equals`: it is the same semantics and it keeps the method reading one way.

## Risk level

The `Objects.equals` conversions are **judgement-PR** material, not mechanical: they widen the set of
values that compare equal (anything outside the boxing cache starts comparing equal where it did not
before), which is the defect being fixed but is still a behaviour change. The `ObjectUtils.notEqual`
one, where the surrounding method already does it that way, is mechanical.

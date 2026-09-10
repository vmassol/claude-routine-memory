# `java:S2097` — "Add a type test to this method" (on `equals`)

Fires on an `equals(Object)` that casts its argument without an `instanceof`/`getClass()` check.
Platform 5 at the time of writing; commons/rendering 0. Splits cleanly into **real defects** and
**false positives**, and the split is decidable from the flagged method's first ten lines.

## The free classifier: how the method establishes the type

* **No check at all** ⇒ real. `XWikiDocument#equals` did `XWikiDocument doc = (XWikiDocument) object;`
  straight after `if (this == object) return true;`, so `equals(aString)` threw `ClassCastException`
  and `equals(null)` threw `NullPointerException` — both contract violations. `NumberProperty#equals`
  is the same shape one level down: `EqualsBuilder.appendSuper(super.equals(obj)).append(getValue(),
  ((NumberProperty) obj).getValue())` evaluates the cast as an *argument*, so `super.equals` returning
  `false` does not short-circuit it.
* **Only a `null` check** ⇒ real, and the fix is a one-token replacement:
  `if (o == null)` → `if (!(o instanceof X))`. The `null` case is covered for free
  (`null instanceof X` is `false`), so the guard count does not change.
* **`getClass().isAssignableFrom(object.getClass())`** ⇒ **false positive**. It *is* a type test —
  it guarantees the argument's class is this class or a subclass, hence an instance of whatever
  interface the body then casts to — Sonar simply does not model that form. Two platform event base
  classes (`AbstractActionExecutionEvent`, `AbstractResourceReferenceHandlerEvent`) are this shape.
  Resolve them in the code with `@SuppressWarnings("java:S2097")` plus the `//` reason, per the OKF
  convention, not by *Accepting* the issue in SonarCloud.

## Mechanics

Use a **pattern variable** and let flow scoping delete the separate cast line:

```java
-        XWikiDocument doc = (XWikiDocument) object;
+        if (!(object instanceof XWikiDocument doc)) {
+            return false;
+        }
```

XWiki is on Java 21, so this is idiomatic. Where a large comment block sits between the guard and the
cast (`AbstractNotificationPreference`), leave the cast alone and only swap the `null` guard — moving
the comment is a bigger diff than the fix.

## Risk level and what to say in the PR

**Judgement PR, not mechanical.** Every real fix converts a *thrown exception* into `false`. That is
precisely what `Object#equals` requires, but a caller that relies on today's `ClassCastException`
would see the change, and these are `equals` methods on central oldcore types (`XWikiDocument`,
`BaseProperty` subclasses). State that trade-off explicitly rather than presenting the change as
behaviour-preserving.

Do **not** confuse this rule with its neighbour `java:S1206` ("also override `hashCode()`"), which
fires on the same classes and is a genuine drop: adding a `hashCode` to a legacy oldcore type changes
how existing instances hash, which is a product decision.

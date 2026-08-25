# `java:S6213` — "Restricted Identifiers should not be used as Identifiers"

OKF-denylisted as *"a rename of a public method or field is an API change, and the XWiki pool sits
on `record(…)` methods of the `*QuestionRecorder` classes"*. **That reason is true of a minority of
the pool** — the seventh denylist entry found to be a partial-pool claim, and the split axis is the
same one that rescued `S1172`: *what is being declared*.

## The split is free — it is in the `message`

- `"Rename this variable to not match a restricted identifier"` → a **variable or parameter**.
  Renaming it changes no signature and breaks no caller; the compiler is the whole verification.
  16 of commons' 18, 4 of platform's 6.
- `"Rename this method to not match a restricted identifier"` → a **method**. Drop: published API
  (`QuestionRecorder#record(T)`), and a test method's name is a naming-convention question rather
  than a cleanup.

One `issues/search` grouped by `message` settles it with no source read.

## The pool

Entirely `record` (never `yield`/`var`/`sealed`), and entirely one feature: commons'
`org.xwiki.extension.job.history` (18) plus platform's `ExtensionHistoryScriptService` and the two
`*QuestionRecorder`s (6). Rendering 0. `ExtensionJobHistoryRecord record` → `historyRecord` reads
better, which is the honest argument for it — the rule's own "it looks like the keyword" is weaker,
since `record` is legal as an identifier forever.

**Ship it as a judgement PR, not in the mechanical batch**: parameter names are visible in Javadoc
and IDE completion, so it is a taste call, and it is cheap for a reviewer to close.

## Mechanics

- Substitute `(?<![\w$.])record(?![\w$])(?!\s*\()`: the `.` in the look-behind spares
  `questionRecorder.record(question)`, and the trailing `(?!\s*\()` spares the method declarations
  and calls you are deliberately NOT renaming.
- Run it **per line, skipping comment lines** — a blanket file-wide regex mangles English prose
  ("Adds a new record to the history"). The one comment form to rewrite is the `@param record` TAG,
  which must follow the parameter: `PARAM = re.compile(r'(@param )record(?![\w$])')`.
- Assert the new name is absent from each file beforehand, and re-check 120 columns (the name grows
  by 7 characters).

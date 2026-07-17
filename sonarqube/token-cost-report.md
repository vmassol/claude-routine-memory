# Token-cost report (load only when asked to report token cost)

- Parse the transcript jsonl at `/root/.claude/projects/<id>.jsonl`. Dedupe assistant turns by
  `message.id` keeping the LAST record per id (streamed partials accumulate; the tool_use block often
  appears only in a later partial), then sort by timestamp. Find phase boundaries by tool_use NAME
  (not string-match on content). Buckets: (1) find = up to first fix edit; (2) fix = edit + build +
  commit + push; (3) post = PR + label + accept + report. Cache-read dwarfs everything. Add the report
  to the PR **body**, not a comment.

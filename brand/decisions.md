# Brand decisions

Append-only log of visual-identity decisions: what was chosen, what was rejected, and why.
Entries are marked superseded when reversed, never edited away.

## 2026-08-02 — Stale `--warn` rendering scoped to telemetry; snapshots never judged stale

**Chosen**: The provenance pattern's stale `--warn` caption applies only to values carrying an
expected-fresh window — telemetry reads. Capture-time provenance (snapshots, uploads) always
shows its capture time and is never rendered as stale.

**Rejected**: Keeping the original "Stale snapshot" `--warn` prescription, with the system built
to judge a snapshot stale ("no longer depicts the current print").

**Why**: Judging a snapshot stale requires the bound printer's live print state in the UI, and no
module seam delivers it — the UI is SSE-driven and never polls. Honouring the original wording
would have added a printer → server → ui print-state event family, with new tests in three
modules, in service of one caption. Scoping the rule keeps a snapshot's age fully visible (the
capture time is always in the caption) while dropping only the judgment, which telemetry — where
an expected-fresh window actually exists — retains.

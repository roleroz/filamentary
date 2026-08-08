# Harvest: per-seam invariants

Transient input to the interface-contracts phase — delete once the contracts are approved.
These are the WHAT-level obligations distilled from twenty review rounds over the v1 module
docs: each is something the review showed can genuinely break, stated as an observable
guarantee. Deliberately absent: every mechanism the v1 docs invented to satisfy these
(sequence schemes, epochs, stamps, markers, state machines) — the contracts derive their own
means fresh, and none of the old machinery is a constraint on them.

## Live sync and events (sessions/printer → server → ui)

- Every connected device converges on the committed state without manual refresh: a change a
  device missed is eventually applied, or the device knows it must refetch — a view is never
  silently stale (R1).
- A committed transcript entry renders exactly once on every device — no duplicate, no loss —
  regardless of how connects, disconnects, fetches, and restarts interleave with the change.
- Delivery may duplicate but must never lose; consumers must be able to apply any event twice
  with the effect of once.
- Partial model output is never rendered misleadingly: a device renders a stream live only if
  it observed the stream's beginning; otherwise it waits for the committed entry — never a
  mid-sentence fragment presented as a message.
- A running turn is visibly in progress on every connected device — including devices that
  joined or reconnected mid-turn — and is never shown running after it ended, with any window
  of wrongness bounded by the turn's next observable activity.
- A dead connection becomes visibly disconnected within bounded time even when nothing is
  being sent: idle and dead must be distinguishable.
- Stale never renders as live: values carrying an expected-fresh window are judged at view
  time against their capture time, and the judgment changes on screen without user
  interaction; capture-time-only provenance (snapshots, uploads) always shows its time and is
  never judged (settled brand decision).
- Cross-restart identity confusion is impossible: a client resuming with pre-restart state can
  never be mis-served different content under reused identifiers — it either resumes correctly
  or knows to refetch.

## Message lifecycle (ui/server/sessions ↔ agent ↔ backend)

- A message the user sent is never silently lost: after any single crash or restart at any
  point, it is either delivered, or visibly failed with a working retry — never invisible and
  unretryable.
- A message is never silently duplicated to the backend: any path that can double-send (retry
  after a lost response, recovery after multiple faults) surfaces as an explicit user action
  or a visible note, never an automatic silent duplicate.
- The transcript never shows a reply or failure answering a message it does not show, and
  never shows a message as sent that the backend path did not accept; a synchronous rejection
  leaves the transcript untouched and the composer's content intact.
- Destructive recovery acts only on definite knowledge: no session, message, or file is ever
  destroyed on an inference or an unanswered question — uncertainty always fails toward
  preservation, and accepted degraded outcomes are visible and bounded, never silent.
- One turn at a time per session; a concurrent send is rejected visibly and harms nothing.
- Usage-limit exhaustion shows the reset time when one exists and resumes automatically at it
  with no user action; an automatic resume never fires into a running turn and never
  re-sends a message that already succeeded; user action supersedes a pending resume.
- Credential failures name the source actually in play and its correct remedy; once
  credentials are fixed, the next attempt succeeds without a service restart — except
  replacing the environment token, which R2.2 defines as requiring one. A fresh install's
  first message fails visibly and recoverably, with the session created and its message kept.
- Backend unavailability (crash, hang, restart-in-progress) always resolves to a classified,
  user-visible outcome in bounded time — nothing blocks indefinitely and nothing wedges,
  interpretation included; recovering one session's backend trouble never silently destroys
  another session's in-flight work beyond a visible, retryable failure.
- A deleted session can never resurrect in any form — rows, pending messages, backend
  mappings, tool access — after any crash/retry interleaving, and deletion is never
  half-visible.

## Binding (sessions ↔ printer ↔ mcp/server/ui)

- Every uploaded `.3mf`'s printer name is eventually evaluated against an interpreted list,
  exactly once, in upload order: transient unavailability of the list (none yet, error)
  defers evaluation — it never permanently misfiles a session — and when a list arrives,
  deferred evaluations complete without user action.
- A standing binding is never revoked or degraded by transient uncertainty; a manual binding
  is never overwritten automatically; a later unique match re-binds an auto-bound session.
- "Missing from the document" is only ever concluded from a list that actually answered.
- Binding state is a closed set every consumer maps completely: tools, routes, and UI controls
  each have defined behaviour for every state, and no consumer can meet an unmapped one.
- A printer name outside the currently known list — reachable at any time, since the list
  changes asynchronously — always yields a definite, distinct error, never undefined
  behaviour.
- The user can bind manually at any time, in every state.

## Evidence (ui/server/sessions/fileindex/mcp)

- Upload identity is stable, unique across kinds within a session, and survives restarts;
  identical filenames across re-slices never collide or confuse; identifiers exposed to the
  agent and the user are the same ones.
- Intake is all-or-nothing: a rejected or failed upload leaves no observable residue — no row,
  no file, no index entry, no event — and a rejection names the file and the reason.
- A whole `.3mf` or gcode file never enters a model prompt; project files are reachable only
  through bounded, continuable views, and a truncated view can always be continued or the
  continuation is refused explicitly — never silently wrong lines.
- Models and browsers are only ever handed formats they can consume; oversized images are
  served as bounded derivatives with the original still retrievable explicitly.
- Every capture carries true provenance (which camera, when) from the point of capture,
  through storage, to display.
- Cross-session access is impossible by construction: no identifier or reference from one
  session can address another's evidence, and a deleted session's evidence access is fully
  revoked.
- Agent snapshots are print-gated; user snapshots are not; both attach as ordinary evidence.

## Cross-cutting

- After any single crash at any point, observable state is either the prior state or the
  completed state (plus visible recovery artifacts) — never a half-visible intermediate.
- Every failure the user can encounter has a defined, cause-naming surface (R10); nothing
  hangs, spins, or waits silently (brand Progress rule).

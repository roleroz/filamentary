# Contract — session store

## Decisions needed from you

This section holds only open items — a topic that is absent is settled and recorded in `engineering/decisions.md`, not forgotten.

**No open decisions.**

Changes since last approval: none — this document was approved on 2026-08-08 with the module-design batch.

## 1. Scope

The session-store operations on the `server → sessions` edge (listing, search, transcript and state fetch, deletion, rename, archive, camera choice) and the `mcp → sessions` lookups edge (`resolve`, the camera-choice read). Owner: `sessions`. Consumers: `server` (routes) and `mcp` (per-call pipeline). The cross-seam conventions (`conventions.md`) apply to every operation here; the events contract owns the delivery of the durable events these operations emit. The deletion cascade `delete` drives is split by owner: the tombstone-first observability boundary is the conventions' deletion rule, the `agent.teardown` step is the message-lifecycle contract's, the evidence-file and index-removal steps are the evidence contract's revocation rule, and the internal step order beyond tombstone-first is implementation freedom.

## 2. Operations

- `list(continuation?, include_archived?) -> Result<{sessions, continuation?}, Error>` — the session listing behind R4's "sessions are listed": most recently created first (this contract's presentation order — R4 requires the listing, not an order), excluding archived sessions unless the selector asks for them; bounded and paged. Each returned entry is the session's *list row* — id, name, archived flag, created time — plus the turn-in-flight flag; the same list row the events contract's session-created payload carries.
- `search(query, continuation?, include_archived?) -> Result<{sessions, continuation?}, Error>` — finds sessions by name and by transcript text, returning the same entry shape as `list`; the archive filter and selector behave as in `list`; bounded and paged. Search is a query over committed state: a concurrently created matching session is absorbed on the fetch side (a page fetched after its commit includes it), never required to render from the live event alone — content-match membership is not decidable from the session-created payload, so the client-side event absorption the events contract's paginated rule offers applies to `search` only where membership is decidable from the event (archive changes and name matches of already-listed rows).
- `transcript(session, continuation?) -> Result<{entries, continuation?}, Error>` — the session's committed transcript entries in commit order with their structured payloads verbatim (never flattened, never re-worded); bounded and paged; this is the fetch half the events contract's gapless composition composes with for the session scope (R4's "reopenable with their full history").
- Paging, once for the three operations above: the `continuation` values are opaque tokens — absent means the first page — honoured exactly or refused explicitly, with one refusal trigger: a service-run boundary (a continuation minted by a prior run). Within a run a continuation is always honoured; concurrent mutation never invalidates one — a concurrently created or archived session, or a concurrently committed entry, lands in a page, the replay, or both, never neither (the events contract's paginated composition).
- `session_state(session) -> Result<State, Error>` — the session's current served state: its list row (name, archived), the turn-in-flight flag, and any standing session-failure condition with its key-set and cause — exactly what the events contract requires fetchable so a joining device converges.
- `delete(session) -> Result<(), Error>` — the confirmed deletion driving the message-lifecycle contract's cascade; unknown-session for a never-created or already-deleted session — user-initiated deletion is not cleanup, and the cascade's own steps carry the cleanup exemptions, not this operation.
- `rename(session, name) -> Result<(), Error>` — an empty-after-trim name is rejected synchronously (the same rule as the binding contract's `bind`); a committed rename is reflected in search.
- `archive(session)` / `unarchive(session) -> Result<(), Error>` — idempotent flag writes, orthogonal to every standing condition; archiving hides the session from the default list and search (R4), the selector reveals it in both, and an archived session stays reopenable.
- `resolve(session) -> Result<Live | Absent, Error>` — the conventions' *existence query*, declared here as that class's sanctioned instance on this seam: it answers absence definitely rather than erroring about it, the error arm reserved for store faults; consumers map Absent to their own unknown-session surfaces (`mcp`'s tool error).
- `camera_choice_set(session, camera, scope) -> Result<(), Error>` / `camera_choice_read(session) -> Result<Choice | None, Error>` (a session with no stored choice answers `None` definitely, never an error) — the printer-state contract's stored choice: the set records the camera *with the scope the offering discovery supplied* (the offering printer's canonical name and telemetry URL — born at the discovery, carried through by the caller, never fabricated en route); the read serves the stored choice in any binding state. Capture-time semantics are the printer-state contract's; the change rides the camera-choice-changed durable event.

Every *session-addressed* operation above except `resolve` (the declared existence query) draws the conventions' unknown-session error for an unknown or deleted session; `list` and `search` address no session. Every committed mutation emits its durable event per the events contract's catalog (session deleted, name changed, archive changed, camera choice changed); the events contract owns their delivery guarantees and payload rules.

## 3. Test obligations (integration)

| Operation | Injected fault | Stall | Race | Restart | Unknown session | Error distinctness |
|---|---|---|---|---|---|---|
| `list` / `search` | store fault → structured error | within bound | `list`: a session created or archived concurrently appears in a page, the replay, or both — never neither (the events contract's paginated composition, injected race); `search`: a concurrently created match appears in a post-commit page (fetch-side absorption, injected race), archive changes of listed rows composing event-side | a prior-run continuation refused explicitly | n/a — not session-addressed | refusal ≠ empty page ≠ fault |
| `transcript` | store fault → structured error, never a partial page served as whole | within bound | an entry committed mid-pagination lands in a later page or the replay, never lost or doubled (injected race) | a prior-run continuation refused explicitly | unknown-session | refusal ≠ end-of-transcript ≠ fault |
| `session_state` | store fault → structured error | within bound | a state change racing the fetch serves pre- or post-state, never a blend (per-session serialization) | serves committed state | unknown-session | condition-bearing state ≠ fault |
| `delete` | a tombstone-commit write failure is a structured fault to the caller with nothing committed (the deletion never became observable); steps after the commit re-drive per their owning contracts (§1's split) | within bound | racing operations land wholly before the tombstone or draw unknown-session (the conventions' deletion rule) | cascade re-driven at startup | unknown-session — asserted distinct from the cascade steps' idempotent completion | fault ≠ unknown-session, asserted |
| `rename` / `archive` / `unarchive` | store fault → structured error, nothing committed | within bound | concurrent writes serialize per session, each landing whole | committed state survives | unknown-session | empty-after-trim ≠ fault (rename) |
| `resolve` | store fault → the error arm — never read as absence | within bound | a resolve racing a deletion answers Live or Absent, never a blend | answers from committed state | n/a — its domain *is* existence: Live and Absent answered definitely | Absent ≠ Error, asserted |
| `camera_choice_set` / `read` | store fault → structured error, nothing committed | within bound | set racing a binding change lands whole; the read serves pre- or post-value | choice survives restart | unknown-session | fault ≠ unknown-session, asserted |

Scenarios:

1. Archive filtering end-to-end: an archived session leaves the default list and search results, the include-archived selector reveals it in both, and it reopens with full history (R4 — all asserted).
2. Search finds by name and by transcript content; a rename is reflected in subsequent searches (asserted).
3. Transcript fidelity: a fetched page's entries are byte-identical in payload to the committed entries, in commit order, across entry kinds (asserted against the emitter fixtures).
4. The camera-choice scope round-trip: the stored scope equals what the offering discovery supplied through the consumer's pass-through, and the read serves it in every binding state (asserted; capture-time validation is printer-state's obligation, tested there).
5. Listing shape: `list` orders most-recently-created first (asserted against out-of-order creation), and every `list` and `search` entry carries the full list row with the turn-in-flight flag (asserted on content, mid-turn and idle).

This seam touches no live external dependency, so it carries no release items.

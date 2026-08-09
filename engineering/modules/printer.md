# Module design — `printer`

## Decisions needed from you

This section holds only open items — a topic that is absent is settled and recorded in `engineering/decisions.md`, not forgotten.

**No open decisions.**

Changes since last approval: none — this version was approved on 2026-08-08.

## 1. Purpose

Printer knowledge and access, per the architecture: document interpretation with retained results, the read-only Moonraker client, log/config views, webcams and snapshots, reachability. The seam behaviour is the printer-state contract's; this document covers the state machines, the clients, and the parsers behind it. The document location is supplied at startup wiring and updatable live through a module-private `set_document_location` entry point (`server`'s config set route hands the new value in).

## 2. Internals

- **Document loader** — reads the configured location (the option `server` surfaces); the retained store keeps the last successfully interpreted *content hash* alongside the list, which is what makes edit-then-revert free and lets an interpretation commit only if its input still matches the document (the mid-flight-edit rule). `set_document_location(path)` — the printer-state contract's operation — is the eager entry point behind `server`'s config set route: it records the new location and starts an interpretation attempt immediately — for the unset→set transition that is the contract's location-unset retrigger (there is no document to hash-compare), and for a set→different-set change the same eagerness follows from the freshness definition itself (Fresh means matching the *configured* document's current content, so a new path makes the retained list Pending until its interpretation lands). Arriving while an interpretation is already in flight, it queues exactly one follow-up attempt for when single-flight frees — the superseded in-flight result never commits as Fresh, and the new location is never left waiting for an unrelated `printers()` call to notice it.
- **Interpretation driver** — single-flight; hands the document text to `agent.interpret_document` and receives the typed `Printers` result (the prompt template and reply parsing are `agent`'s — a parse failure arrives as the malformed-output classification, routed here as a backend-class retry cause). Commit is transactional: list + hash + Fresh, or nothing. Retrigger routing is a table over the six cause classes.
- **Retained store** — the module's tables in the shared SQLite: the list, content hash, freshness state and cause. Reachability and the retained last failure are deliberately *not* persisted — an in-memory per-printer map, giving the per-run semantics (restart → Unknown) for free; per-printer serialization is a lock per map entry, and the discard rule drops observations whose printer's URL changed or vanished between issue and landing (compared against the list generation captured at issue).
- **Moonraker client** — HTTP with per-call timeouts (the bounded-resolution bound); an explicit endpoint allowlist containing only reads — printer object queries (status, temperatures, position, print state, configfile), the webcam list, and the klippy.log file reads — so read-only holds by construction. Every response carries capture provenance (source, timestamp, fresh window for live values).
- **Log views** — klippy.log served through Moonraker's file API as bounded byte-range reads; the restart boundary is found by scanning backwards for the restart-marker line pattern (a release item — the pattern rides in data); rotation is detected by comparing a remembered head-prefix and length, growth by length alone; continuations carry (offset, head-prefix) and are refused on rotation, honoured across growth.
- **Config views** — the configfile printer object provides section names (the listing operation) and per-section content; continuations carry a section-content hash and are refused when it changed between pages.
- **Webcams and snapshots** — discovery via the webcam list, its outcome feeding the reachability map like every Moonraker call (a webcam-only-queried printer gets its verdict from discovery alone), the successful answer carrying the offering scope (the queried printer's canonical name and telemetry URL) per the contract's discovery-born-scope rule; snapshot URL resolution applies the relative-URL rule (resolved against the printer host's web frontend base — a release item); capture validates a stored choice's scope (canonical + URL) then the camera name against discovery — a bare per-call identifier (the contract's `Agent`-request form) skips the scope check, having no recorded scope, and is validated against discovery alone — fetches with a bounded timeout, and returns bytes + provenance to the caller (`sessions` attaches; this module stores nothing).
- **Print-state gate** — the agent-requester check reads print state in the same capture call path; idle and undeterminable produce the two distinct refusals, undeterminable carrying the underlying cause.
- **Event emission** — freshness (list-level) and reachability (per-printer) transitions emit through the module's multi-subscriber channel per the events contract.

## 3. Interface

Public: the printer-state contract's operations, `set_document_location` included (the contract's `server → printer` location supply, at wiring and live alike). Module-private: the retrigger table, the marker pattern and URL-resolution rule (data), and the clients behind injectable traits.

## 4. Error handling and failure visibility

Moonraker and camera-host failures map to the contract's cause categories with the failing endpoint logged; interpretation failures log the classification and, for document-class causes, the path and problem that will ride the freshness cause; store faults surface per the conventions and are logged with the operation; the discard rule logs discarded stale observations. Freshness causes are the user-visible surface for everything document- and interpretation-shaped, so nothing here depends on logs for user visibility — logs add the endpoint/frame detail R10's rendering does not carry.

## 5. Test plan

Contract tests (`test_contract_*`), unit level, against scripted Moonraker/camera fakes, a scripted `agent` fake, and stateful store fakes:

1. Freshness machine: every transition of the six-cause taxonomy, each retrigger class's routing, single-flight under concurrent loads, edit-then-revert zero calls, commit-if-current under a mid-flight edit, duplicate-name document problem retaining the prior list; `set_document_location` is eager — the unset→set call starts an interpretation and its freshness event lands with zero intervening `printers()` calls (asserted on call counts), a set→different-set call flips to Pending and re-interprets the same way (asserted), and one arriving mid-flight queues exactly one follow-up attempt that runs when single-flight frees (asserted).
2. Interpretation parsing: the recorded reply shape parses to entries; malformed replies classify malformed-output and leave the retained state untouched.
3. Reachability: observation-driven transitions per cause category, Unknown neutrality, retained-last-failure exposure and clearing rules, per-run reset, the URL-change/removal/reappear reset-and-discard rules under an injected race.
4. Log views: window bounds, backward marker scan (found, absent → boundary-unknown), growth-honoured and rotation-refused continuations against crafted logs.
5. Config views: section listing, per-section bounds and continuation, changed-content refusal, unknown-section error.
6. Webcams and snapshots: absent-vs-unknowable discovery outcomes, a discovery failure registering as the printer's reachability observation with no telemetry call ever made (asserted on the `reachability()` read), the full refusal partition (no-camera, unknown-camera — stored choice and bare per-call identifier alike — out-of-scope on both mismatch axes and asserted inapplicable to the bare form, idle, undeterminable-with-cause, unreachable) each distinct, URL resolution cases (absolute, relative), a successful discovery answer carrying the offering scope — the queried printer's canonical name and telemetry URL, asserted on content per the contract's discovery-born-scope rule — provenance on success, nothing stored.
7. Read-only: the allowlist contains no state-changing endpoint (asserted over the client's surface); zero traffic with no operations issued; zero traffic ever for evidence-only printers, each operation drawing no-telemetry.
8. Provenance: fields present per value class — fresh windows on live telemetry only.
9. Unknown printer: every per-printer operation given a name outside the current list draws the distinct unknown-printer error with zero Moonraker traffic (asserted per operation); `reachability()` on an evidence-only printer answers the NoTelemetry verdict itself (asserted).
10. Serialization: a `reachability()` read concurrent with an observation returns the pre- or post-observation verdict, never a blend; two concurrent observations land serialized per printer, one coherent final verdict, events in commit order (injected races).
11. Restart and crash: a fresh instance over the same store, backend down, serves the retained list with correct freshness and cause immediately (R3's restart criterion); a crash mid-interpretation costs at most one redundant model call after restart (asserted by call count).

Failure-mode tests (`test_failure_<dependency>_<mode>`):

- moonraker HTTP: `connection`, `timeout` (injected stall resolves within bound), `error_status`, `malformed_body` — each the right cause, each flipping reachability as that observation.
- camera host: `connection`, `timeout`, `protocol` — structured, retryable, reachability untouched.
- agent: each availability classification plus `malformed_output` — freshness cause per class, no recycle-adjacent behaviour triggered from here.
- retention store: `unavailable` (the `printers()` Error arm, distinct from every freshness state), `write_failure` (freshness cause, prior list in force), `corrupt_row` (nothing valid to serve — the Error arm, like `unavailable`, never a freshness cause over garbage), `stall` (bounded).
- document filesystem: `missing`, `unreadable` (causes naming the path), `location_unset` (cause naming the option).

Provisional mocks — unverified: the Moonraker response fixtures for every allowlisted endpoint, the klippy.log restart-marker and rotation fixtures, real webcam snapshot bodies, and the relative-URL resolution rule; plus the project-authored interpretation conversation. Each is a release item carried by the printer-state (13–16) and message-lifecycle (live interpretation) contracts; recordings replace them at first live run.

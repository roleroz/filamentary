# Engineering decisions

Append-only. Each entry states what was chosen, what was rejected, and why. Entries are never
edited away; a reversed decision gets a new entry and the old one is marked superseded.

## 2026-08-02 — Agent boundary is the Agent Client Protocol, via the claude-agent-acp sidecar

**Chosen**: the `agent` module is an ACP client (official Rust SDK) managing a Node sidecar
process running `agentclientprotocol/claude-agent-acp`, which wraps the official Claude Agent
SDK in subscription mode. Future backends are any ACP-speaking agent.
**Rejected**: driving `claude -p --output-format stream-json` directly.
**Why**: ACP makes sessions, streaming, MCP configuration, and permissions protocol
primitives instead of re-derived semantics over a weaker interface, and it keeps subscription
auth as a pass-through to Anthropic's own SDK. Accepted cost: a Node runtime and a
community-governed dependency.

## 2026-08-02 — UI framework is React

**Chosen**: React for the TypeScript browser app.
**Rejected**: Svelte.
**Why**: best-supported framework under Bazel's rules_ts/rules_js toolchain — the build
system is fixed — and the largest ecosystem for the session/chat UI. Weight accepted.

## 2026-08-02 — Persistence is SQLite

**Chosen**: a single SQLite database for sessions, transcripts, evidence metadata, bindings,
and the retained printer-document interpretation, with full-text session search via FTS;
uploaded evidence files live on disk beside it.
**Rejected**: plain JSON/files.
**Why**: search, atomic writes, and migrations come built in, and backup is copying one file;
the bundled C dependency is Bazel-friendly.

## 2026-08-02 — Backend in Rust

**Chosen**: the system is built in Rust; Bazel remains the build system, and the browser UI
remains TypeScript. The spec stays implementation-neutral — the stack is recorded here and
will be stated as current state in the architecture doc.
**Rejected**: Python (the original choice, made only because the Claude Agent SDK ships in
Python) and Go (the smoothest Bazel story, but not chosen).
**Why**: research showed the agent abstraction can be a language-neutral protocol boundary —
the Agent Client Protocol with its official Claude adapter — so the SDK's language no longer
dictates the backend's. With that constraint gone, the user chose a compiled language.
Consequence for architecture: the agent backend integrates through a subprocess protocol
boundary (ACP-style) rather than in-process SDK calls. Supersedes the "Backend in Python,
built with Bazel" entry below.

## 2026-08-02 — User snapshots at any time; only agent captures are print-gated

**Chosen**: the user can take a webcam snapshot whenever a webcam is discovered, regardless of
print state; the agent may capture only while Moonraker reports an active print (printing or
paused), asking the user for a picture otherwise.
**Rejected**: gating both requesters on an active print (the 2026-08-01 stance below).
**Why**: the print-state gate exists to stop the *agent* from autonomously collecting useless
idle images; the user deciding to look at their own printer needs no such guard. Refines the
webcam-capture entry below — the agent-side gate stands, the user-side gate is dropped.

## 2026-08-02 — The `.3mf` is optional evidence, prompted for when settings matter

**Chosen**: an investigation can proceed on any evidence; the agent asks for the `.3mf`
whenever slicer settings become relevant, and a recommendation that would need the print's
actual values says so until one is uploaded.
**Rejected**: requiring a `.3mf` before any findings — the earlier stance, taken from the
original brief's "start by providing a slicer project".
**Why**: the `.3mf` is the settings ground truth, but many print problems — mechanical
noises, Klipper errors, adhesion visible on camera — need none of it, and blocking those
investigations was pure friction. Supersedes the `.3mf`-mandatory wording implied by the
no-calibration-only-sessions entry below.

## 2026-07-31 — Calibration is OrcaSlicer-guided, not Filamentary-generated

**Chosen**: Filamentary instructs the user which OrcaSlicer built-in calibration test to run and
with which settings; the user slices and feeds the result back. Klipper-native tuning is issued
as exact console commands for the user to run.
**Rejected**: Filamentary generating calibration gcode itself (alone, or alongside the guided
flow).
**Why**: gcode generation would have to be correct per printer/filament combination — a large,
risky scope for v1 — while OrcaSlicer already ships maintained calibration tests. Guided flow
also matches the read-only printer stance below.

## 2026-07-31 — Webcam snapshots are in scope when a camera is configured

**Chosen**: when a printer has a webcam (Moonraker/crowsnest), Filamentary fetches snapshots on
demand as session evidence; with no webcam the feature is absent and everything else works.
**Rejected**: phone photos only in v1, webcam support deferred.
**Why**: snapshots let the user observe a print mid-run without walking to the printer, the
Moonraker path is cheap to add, and graceful absence keeps the cost near zero for camera-less
printers.

## 2026-07-31 — Printer access is read-only in v1

**Chosen**: Filamentary only reads printer state through Moonraker; every recommended printer
action is executed by the user.
**Rejected**: printer writes gated by per-action confirmation with an always-visible emergency
stop (the model `brand/design-language.md` describes).
**Why**: the user chose the smaller, safer v1 over the brand doc's write-gated flow. Writes move
to future work; the brand doc's confirmation-gate and emergency-stop patterns stay defined but
dormant until then.

## 2026-07-31 — Photo capture: direct camera trigger and file upload, both over plain HTTP

**Chosen**: the UI offers both taking a photo — the phone's camera triggered directly from the
UI — and uploading an existing image file. Both are achievable over plain HTTP (a file input
with the camera-capture attribute opens the camera directly), so no TLS in v1.
**Rejected**: an in-page live viewfinder in v1. It requires a secure context, meaning TLS with
a certificate installed on every phone.
**Why**: the user asked for both capture paths; neither requires the certificate-provisioning
cost of a live viewfinder, which moves to future work. Interpretation on record: "trigger the
camera directly from the UI" is read as opening the phone camera from a UI control, not as an
embedded live preview — if an embedded viewfinder was meant, this decision needs revisiting.

## 2026-07-31 — Profiles stay slicer-owned; Filamentary reads them from the `.3mf`

**Chosen**: Filamentary owns no filament or process profiles. The profile a session works on is
the one embedded in the uploaded `.3mf`, and every profile change is delivered as instructions
the user applies in OrcaSlicer by hand.
**Rejected**: Filamentary keeping its own profile records (with manual application), and
Filamentary writing OrcaSlicer's profile files directly.
**Why**: the `.3mf` already carries the exact settings the print was made with, so it is the
truthful source for what a session is debugging; a parallel profile store could drift from what
the slicer actually uses. Future work: Filamentary owning profiles with the slicer consuming
them.

## 2026-08-01 — Webcams discovered through Moonraker, captured only during active prints

**Chosen**: webcams are not described in the printer document; Filamentary discovers them
through the bound printer's Moonraker. Snapshots are taken only while Moonraker reports an
active print — when the printer is idle, or the print state is unknowable because Moonraker is
unreachable, no image is taken or stored, and the user is asked for a phone photo instead.
**Rejected**: a webcam field in the printer document, and webcam capture independent of
telemetry (which an earlier spec revision briefly stated).
**Why**: Moonraker already knows its webcams, so the document field was redundant; and webcam
images are only useful while a print is running, which only Moonraker can confirm — tying
capture to an active print also answers how to reach a webcam when Moonraker is down: don't.
This supersedes the webcam-address aspect of the printer-document decision below and refines
the webcam-capture decision below.

## 2026-08-01 — Self-signed TLS approved as fallback if camera capture needs a secure context

**Chosen**: v1 serves plain HTTP — triggering the phone camera via the file-input capture
mechanism needs no secure context. If implementation reveals a secure context is unavoidable
for a required capture path, a self-signed certificate is the pre-approved v1 fallback.
**Rejected**: provisioning TLS up front.
**Why**: no known v1 requirement needs it; the contingency is recorded so hitting the
constraint mid-implementation does not need a new approval round.

## 2026-07-31 — No calibration-only sessions in v1

**Chosen**: v1 sessions are debugging investigations started from a `.3mf`; sessions dedicated
to calibration move to future work. Calibration guidance still occurs inside debugging
sessions, governed by the OrcaSlicer-guided decision above, when a finding calls for
recalibration.
**Rejected**: a dedicated calibration session type in v1, whether with an optional or a
required `.3mf`.
**Why**: it removes an entire input flow — printer-first session creation, filament identity
without a project file — from v1 while keeping calibration reachable through the debugging
path.

## 2026-07-31 — Printer descriptions live in a user-owned document, not in Filamentary

**Chosen**: printers are described in a free-form document the user owns; Filamentary reads it,
never writes it, and a Filamentary configuration option names its location.
**Rejected**: in-app printer registration with Filamentary owning the printer records.
**Why**: the user already keeps printer descriptions as their own documents; pointing
Filamentary at that file keeps a single source of truth outside the tool, consistent with
profiles staying slicer-owned.

## 2026-07-31 — Backend in Python, built with Bazel [SUPERSEDED 2026-08-02 by "Backend in
Rust" above]

**Chosen**: Python for the system's code, Bazel as the build system.
**Rejected**: TypeScript, the other language the Claude Agent SDK ships for.
**Why**: the user's call — the Claude Agent SDK's availability decides the language, and Bazel
is the user's standard build system. The UI language is a separate decision.

## 2026-07-31 — Browser UI in TypeScript

**Chosen**: TypeScript for the browser UI, built under Bazel with Aspect's rules_ts and a pnpm
lockfile so every dependency stays pinned. The framework choice is deferred to the
architecture doc.
**Rejected**: Python-server-rendered HTML with htmx.
**Why**: the UI is genuinely stateful — streaming session transcripts, camera capture, upload
progress, live telemetry — so real client-side code exists either way; TypeScript keeps it
typed to the same standard the project demands of Python, where the htmx route would leave the
interactive parts as untyped hand-written JavaScript.

## 2026-07-31 — Model access sits behind an agent-backend abstraction

**Chosen**: all model interaction goes through an agent-backend abstraction, with the Claude
Agent SDK (subscription mode) as the only v1 backend; no component outside the boundary depends
on any vendor's SDK.
**Rejected**: coupling the system directly to the Claude Agent SDK.
**Why**: nothing in Filamentary's job is Anthropic-specific, and other vendors ship comparable
agent SDKs — a clean boundary makes adding them future work instead of a rewrite. Additional
backends are recorded in the spec's future work.

## 2026-07-31 — LAN-only access with no authentication

**Chosen**: the service binds to the LAN so the phone can reach it, the home network is trusted,
and there is no login. The UI shows the `LOCAL`/`EXPOSED` badge.
**Rejected**: a shared token/password (e.g. QR-scanned from the phone).
**Why**: single user on a trusted home network; authentication is complexity v1 does not need.
It moves to future work alongside any exposure beyond the LAN.

## 2026-08-02 — Sidecar vendored and pinned into the build

**Chosen**: the `claude-agent-acp` adapter and its npm dependency tree are pinned by lockfile
and vendored into the build via Bazel rules_js; hosts need only a Node runtime, verified at
startup with a clear failure message.
**Rejected**: fetching the adapter via `npx` at run time.
**Why**: the app ships the exact adapter it was tested with — hermetic, offline-installable,
and consistent with the pin-everything rule; runtime fetching is unpinned by nature.

## 2026-08-02 — XDG locations with a TOML config file

**Chosen**: configuration at `~/.config/filamentary/config.toml` with flag > env > file >
default precedence; the SQLite database and evidence files under
`~/.local/share/filamentary/`; the service log at `~/.local/state/filamentary/`.
**Rejected**: CLI flags only with data beside the binary.
**Why**: platform convention, one directory to back up, and data decoupled from the install
location.

## 2026-08-02 — Mesh geometry exposed as rendered views plus measurements

**Chosen**: `fileindex` keeps the `.3mf` mesh and exposes it as numeric measurements and
on-demand rendered images from named viewpoints, so the agent can compare intended geometry
against photos of the print.
**Rejected**: measurements only.
**Why**: visual comparison of intended-versus-printed shape is central to R9's photo-driven
debugging; a software-rendering dependency is an accepted cost.

## 2026-08-08 — Newest named upload supersedes an older deferred evaluation

**Chosen**: at most one binding evaluation is ever outstanding — the newest named `.3mf`
upload's; a newer named upload supersedes an older upload's uncompleted evaluation, which
completes as a no-effect "superseded" (no state change, no note).
**Rejected**: strict upload-order completion, where every deferred evaluation completes with
its full outcome in sequence.
**Why**: a superseded evaluation's only possible effect is overwriting newer truth with
staler — and strict ordering would make a corrected re-slice wait behind a stale deferred
evaluation while the printer list is unavailable, delaying the bind the user is waiting for.
Supersession also dissolves the contradiction between "positive outcomes complete on any
serving list" and "no upload skips past an earlier deferred one" that the review loop
plateaued on.

## 2026-08-08 — Camera choice scoped to the printer it was chosen on

**Chosen**: the stored camera choice records, at set time, its printer's canonical name and
telemetry URL; a capture whose target printer differs in either is refused with a structured
out-of-scope error naming the choice's printer — never a capture from any camera. No
binding-side trigger clears the choice.
**Rejected**: the binding contract reporting resolved-printer changes to a printer-state
clearing trigger (the prior design).
**Why**: printer identity beyond the canonical name does not exist in the data, so a
change-detection trigger cannot distinguish a rename from a reassignment and misses a URL
re-pointed at a different machine — the exact case where a stale choice silently captures
another printer's camera. Capture-time scope validation is stateless, closes the URL hole,
and deletes a cross-contract seam; the cost is one re-pick after a rename, chosen as safety
over convenience.

## 2026-08-08 — The unbound session's evaluation stands, re-deriving on every Fresh list

**Chosen**: the newest named upload's evaluation stands while the session is unbound: each
Fresh list re-derives its outcome, so a document edit that adds the missing printer binds the
session automatically with no re-upload, and stale candidates or notes refresh with the
document. It ends when an outcome binds the session, a manual bind lands, or a newer named
upload replaces it; on a bound session an evaluation completes exactly once.
**Rejected**: one-shot evaluations, where a completed no-match or ambiguous outcome stays
frozen until the next upload.
**Why**: frozen outcomes strand stale annotations — after the user fixes the document, the
session still shows candidates the document may no longer carry, contradicting "a note never
outlives the situation it describes" — and make the user re-upload a file the system already
holds to get the bind they already asked for. The standing rule is also symmetric with the
judgment machinery, which already re-derives a bound session's state on every Fresh list.

## 2026-08-08 — Cross-seam conventions extracted into one document; completeness sweeps replace further sampling rounds

**Chosen**: the rules universal to every seam (unknown-session outcomes, bounded resolution,
structured error arms, fault-class versus validity errors, crash-atomicity and re-drive,
per-entity serialization, emission discipline, and standard test conventions) move to
`engineering/contracts/conventions.md`, cited by every contract and approved with the batch;
and the plateaued review loops are unblocked by one deterministic completeness sweep per
document (operations × failure axes, prose claims mapped to obligations) plus a cross-document
vocabulary coherence pass, before any further review rounds.
**Rejected**: keeping the universal rules restated per contract, and continuing stochastic
single-reviewer rounds until the coverage matrix exhausts by accident.
**Why**: the finding stream was dominated by per-operation rediscovery of the same universal
rules and by unsampled cells of each contract's coverage matrix — per-doc restatement made
every operation a chance to miss a carry, and sampling rounds converge only stochastically.
One normative home plus one deterministic sweep closes both classes at the root. Also adopted
as a standing development-process rule in the global config.

## 2026-08-08 — Standing evaluation extended: it stands until a manual bind

**Chosen**: the newest named upload's evaluation stands while the session is unbound *or
automatically bound*, re-deriving on every Fresh list — a document edit can clear a stale
mismatch note or re-bind an auto-bound session to a newly unique match. Only a manual bind
(or a newer named upload) ends it; a manually bound session's evaluation completes exactly
once. Extends the same-day "unbound session's evaluation stands" decision, which stopped at
the first bind.
**Rejected**: ending the standing evaluation at any bind, automatic included.
**Why**: stopping at an automatic bind left the auto-bound session's mismatch note frozen —
it could describe a name the edited document now matches, permanently, contradicting "a note
never outlives the situation it describes". Automatic binding follows the document by
definition, so its annotations and re-binds should track the document too; manual binds are
the user's explicit override and stay untouched.

## 2026-08-08 — Binding state becomes a pure derivation

**Chosen**: binding.md is rewritten around derived state: the reported state, judgment,
annotations, and pending causes are computed from four inputs — the stored name + manual flag
(the only binding write, by bind or auto-bind), the newest named upload postdating that bind,
and printer-state's retained list with its freshness. Events emit when the derived output
changes; there are no stored judgments, no evaluation-completion machinery, no deferral queue.
**Rejected**: keeping the event-sourced model (stored judgments written on Fresh arrivals,
evaluations as completing acts, deferral triggers) and patching its round-11 findings.
**Why**: eleven Opus rounds plateaued at 6–10 findings through two structural passes, with the
last round dominated by pairwise interactions between the model's own sub-machines. A pure
derivation deletes the interaction surface: one write path, one emit rule, causes named by
whichever input is missing, and the approved standing-evaluation behaviour falls out by
construction instead of by trigger rules.

## 2026-08-08 — Review convergence by blocking bar; evidence fixed blockers-only with a session-blind fileindex

**Chosen**: review findings block acceptance only when an observable outcome is contradicted,
unsatisfiable, or undefined; everything else is fixed silently when trivial or rejected into
the ledger, and a round with no blocking findings converges the document. For the evidence
contract, only round 12's genuine blockers were fixed — chiefly declaring `fileindex`
session-blind, with session existence resolved by its callers before it is consulted — and
the earlier-proposed full dedup rewrite was dropped as unnecessary under the bar.
**Rejected**: continuing accept-everything review rounds (roughly forty rounds produced an
empty rejected-findings ledger and no termination), and the full evidence restructure.
**Why**: the loop's termination condition was reviewer-judged, and adversarial reviewers of a
sizeable document always return something defensible; without triage the loop cannot
terminate and every fix grows the surface. The bar restores the triage step the process
always specified. Also adopted as a standing rule in the global config.

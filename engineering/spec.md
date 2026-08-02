# Filamentary — Specification

## Decisions needed from you

This section holds only open items — a topic that is absent is settled and recorded in
`engineering/decisions.md`, not forgotten.

**No open decisions.**

Changes since last approval: none — this version was approved on 2026-08-02.

## 1. What Filamentary is

Filamentary is a locally-run assistant that improves the output quality of a user's 3D printers.
It uses an AI agent — in v1 Claude, through the Claude Agent SDK in subscription mode — to
debug printing problems: a session investigates a specific print problem from evidence —
photos, the slicer project, the gcode, and the printer's own state — and recommends exact
setting changes. When a problem calls for recalibration, the recommendation is guided
calibration: the exact OrcaSlicer built-in test or Klipper console command to run. Sessions
dedicated to calibration alone are future work.

It serves a single person, the printers' owner, through a web browser — including the browser on
a phone standing next to the printer, which is also how photos of prints are captured.

## 2. Scope and non-goals

### In scope (v1)

- Linux as the host platform, started locally by the user.
- Klipper-based printers, accessed read-only through their Moonraker API.
- OrcaSlicer as the sole supported slicer; its `.3mf` projects and gcode as session inputs.
- Multiple printers, each with its own configuration. Filament and process profiles are not
  owned by Filamentary: the profile a session works on is read from that session's `.3mf`.
- Multiple persistent, named, resumable sessions; each session addresses one problem.
- MCP tools giving the model targeted access to `.3mf` and gcode content, printer telemetry,
  and webcam capture.
- Photo capture from the phone browser; webcam snapshots discovered through Moonraker —
  user-requested at any time, agent-requested only during an active print.

### Non-goals (v1)

- Slicing, gcode generation, or mesh editing — OrcaSlicer does all slicing.
- Writing to the printer: no commands, no file uploads, no print starts, no emergency stop.
  The confirmation-gate and emergency-stop UI patterns in `brand/design-language.md` are
  defined for the future printer-write capability and do not appear in v1.
- Multi-user support, accounts, or authentication.
- Cloud or hosted operation; the system runs on the user's own machine.
- Anthropic API-key access — subscription mode only, as a hard constraint, not a default.

## 3. Requirements

Each requirement carries acceptance criteria; end-to-end tests trace back to them. The canonical
end-to-end fixtures are `testdata/rack_cable_manager_2u.3mf` and
`testdata/rack_cable_manager_2u_PETG_19m11s.gcode` (an OrcaSlicer 2.4.2 project and its sliced
gcode for a Voron v2 350 with a 0.6 mm nozzle, Polymaker PolyLite PETG); new fixtures are added
under `testdata/`.

### R1. Local runtime, browser access from the phone

The system runs as a locally-started service on Linux and serves its UI over HTTP, listening on
all of the machine's network addresses.
On startup it prints to the shell every URL at which the UI is reachable — one per network
address it listens on — each accompanied by a QR code rendered in the terminal, so the phone
reaches the freshly started service by scanning rather than typing an IP. When the service
cannot start serving — the port already occupied, an address it cannot bind — the start
command exits non-zero with a message naming the cause: startup failures are the one class
that reports to the shell, because they precede the UI.
The UI is fully usable from a phone browser and offers two ways to attach a picture: taking a
photo — the camera is triggered directly from the UI — and uploading an existing image file.
Both work over plain HTTP. Image uploads accept the formats phones actually produce — JPEG,
PNG, and HEIC at minimum; an unreadable or unsupported image is rejected with a message naming
the file and the reason. Several devices can use the UI at the same time against the same
session — the phone at the printer and the laptop doing the slicing: anything sent from one
device (text, file, image) and every agent response appears on all connected devices
automatically, without a manual refresh. Each device shows its own connection state: when the
live connection drops — screen lock, a wifi blip — the UI marks itself disconnected rather
than silently going stale, reconnects automatically, and shows what arrived in the interim.
The UI displays whether the server is reachable
only from localhost or exposed to the network, using the `LOCAL` / `EXPOSED` badge from
`brand/design-language.md`; because v1 always listens on every address, the badge reads
`EXPOSED` — the `LOCAL` state becomes reachable only if a future bind option arrives.

Acceptance criteria:
- A single documented command starts the service; it accepts connections on every network
  address of the machine, and the UI loads from another device on the LAN via any of them.
- With the port already occupied, the start command exits non-zero and names the port and the
  cause.
- Startup output lists one URL per address the service listens on, each with a terminal QR code
  that decodes to exactly that URL.
- With the same session open on two devices, a text message, a photo, and a file sent from one
  each appear on the other, and the agent's response appears on both, with no user action on
  the receiving device.
- A JPEG, a PNG, and a HEIC image each attach successfully; an unreadable image, and a
  readable image in an unsupported format, are each rejected with a message naming the file
  and the reason.
- With a device's connection interrupted and restored, the UI shows the disconnected state
  while down and, after reconnecting, shows the content sent in the interim.
- From the phone, both taking a new photo triggered from the UI and uploading an existing image
  attach the picture to the open session, where it is visible.
- The badge reads `EXPOSED`.

### R2. Agent access

#### R2.1. Agent-backend abstraction

All model interaction goes through an agent-backend abstraction: nothing about the model's job
here is vendor-specific, so no component outside that boundary may depend on any particular
vendor's SDK. The abstraction owns what every backend must provide — running a session's
conversation, streaming responses, calling Filamentary's MCP tools — the file queries of
R5, the telemetry reads of R6, and the snapshot requests of R7 — and interpreting the printer
document of R3. It also owns mid-session
failure behaviour: a failed model call — network failure, backend outage, usage limits —
surfaces in the UI with its cause, never loses the session or its history, and the failed
message can be retried. Requirements specific to a backend live with that backend (R2.2 for
the only v1 backend), so future backends add their own section without touching this one.

Acceptance criteria:
- No code outside the agent-backend boundary references any vendor's SDK — mechanically
  checkable once the architecture doc names the boundary.
- With the backend failing mid-session (mocked), the failure appears with its cause, the
  session's transcript stays intact, and retrying the message succeeds once the backend
  recovers.

#### R2.2. Claude Agent SDK backend (v1)

The sole v1 backend is the Claude Agent SDK in subscription mode. Both of its subscription
authentication methods must work: the credentials under `~/.claude` left by logging in with the
`claude` CLI, and a long-lived token generated with `claude setup-token` and provided through
the `CLAUDE_CODE_OAUTH_TOKEN` environment variable. When both are present, the environment
token is used; a rejected credential is surfaced naming the source that was tried, never
silently falling back to the other. No Anthropic API key is read or accepted. Credential
failures never prevent session creation — creating and persisting sessions is local, so the
session and its messages survive and the model call fails per R2.1. When no credential source
is available, the failure message shows both recovery paths: logging in with the CLI, and
generating a token with `claude setup-token` and storing it in the environment. Credentials
that are present but rejected — an expired token, a stale login — surface the backend's
failure naming the rejected source, with the recovery path that actually applies to it: a
rejected environment token calls for a new token and a restart, or removing the variable so
the `~/.claude` credentials take over; rejected `~/.claude` credentials call for logging in
again or providing a token. Credentials are read at each model call, so logging
in with the CLI takes effect on the next attempt with no service restart; the environment
token, by contrast, is fixed when the process starts — updating `CLAUDE_CODE_OAUTH_TOKEN`
means restarting the service with the new value, and the recovery message says so.
Subscription usage-limit exhaustion is surfaced as exactly that, distinguishable from other
failures: the message shows when the limit refreshes, and the interrupted work resumes
automatically once it does, without user action.

Acceptance criteria:
- The backend works with `~/.claude` credentials and no token in the environment, and with
  `CLAUDE_CODE_OAUTH_TOKEN` set and no `~/.claude` credentials.
- The system operates with no `ANTHROPIC_API_KEY` in the environment; with the variable
  present, behaviour is unchanged — subscription credentials are used and the key is ignored.
- With no credential source available, the session is still created with its derived name and
  keeps its first message; the model call fails with a user-visible message naming the cause
  and showing both
  recovery paths — not a silent failure or a stack trace — and retrying after credentials are
  fixed continues the session.
- With both credential sources present, the environment token is the one used; when it is
  rejected, the `~/.claude` credentials are not silently tried — the failure names the token
  as the source.
- With a present-but-rejected credential (mocked backend rejection), the failure names the
  rejected source and shows the recovery path that applies to it.
- With the backend reporting usage-limit exhaustion (mocked), the message identifies the
  usage limit specifically — not a generic failure — and shows when the limit refreshes; once
  it refreshes (mocked), the session continues automatically with no user action.
- With no environment token set, logging in with the CLI while the service is running makes
  the next model attempt succeed, with no service restart.

### R3. Printers

Printers are described in a document the user owns: Filamentary reads it and never writes it.
A Filamentary configuration option names the document's location. The document is free-form
text — Filamentary interprets it rather than imposing a format — and per printer it must
convey exactly one name, any number of aliases, and the Moonraker URL when telemetry is
wanted; printer names must be unique, and an interpretation yielding two printers with the
same name is surfaced as a document problem like any other document failure. A canonical
printer-document fixture under `testdata/` pins what interpretation must find.
Interpreting the document uses the agent backend, and the most recent successful
interpretation is retained, surviving service restarts and later document failures alike:
whether the backend is unavailable or a previously read document has gone missing or
unreadable, the printer list and existing bindings from the last successful read keep
working while the problem is surfaced — a pending re-interpretation names the backend as the
cause, a document failure names the path and problem. Before any
successful interpretation exists — a fresh install without working credentials — the printer
list is empty, with a message naming the backend as what is needed to read the document.
Webcams are not described in the document — they are discovered through Moonraker (R7).
A printer conveyed without a Moonraker URL is an evidence-only printer: its sessions run on
uploaded evidence, the UI shows a permanent no-telemetry notice — distinct from the
degraded-mode bar, which is reserved for an unreachable Moonraker — and no telemetry or webcam
capability exists for it. A missing, unreadable, or printer-less document never prevents the
service from starting: it runs, prints its URLs, and surfaces the problem in the UI so the
document can be fixed and re-read without a restart. A session keeps its binding by printer
name: if a document edit removes that name, the session stays open with its evidence,
telemetry and webcam become unavailable with the UI naming the missing printer as the cause,
and the user can re-bind to any documented printer. A session is bound to at
most one printer at any time, and must be
bound before telemetry or webcam capture is available: the binding happens automatically
when the session's `.3mf` is uploaded, by matching the project's printer name against the
names and aliases in the document — a match is case-insensitive, whitespace-trimmed string
equality, never fuzzy — and the user can bind or re-bind manually at any time, including
before any `.3mf` has been uploaded. A name matching more than one documented printer (colliding
aliases) never auto-binds: the session stays unbound and the UI presents the candidates for
the user to pick. Each `.3mf` upload re-runs the automatic match, with two provisos: a manual
binding sticks until the user changes it, and on an already-bound session a later upload that
matches nothing, or matches ambiguously, leaves the existing binding unchanged with a note
that the new project's printer name did not match it. Wherever the bound printer appears, it
is shown under its one
name — never under the alias that happened to match. Filamentary does not own filament or
process profiles either: the profile a session works on is the one embedded in its uploaded
`.3mf`, and every profile change is handed to the user as instructions to apply in OrcaSlicer
by hand.

Acceptance criteria:
- The canonical document fixture yields exactly its expected printers: names, aliases, and
  telemetry capability.
- With a document describing two printers, both appear; a session bound to one never reads the
  other's telemetry.
- Uploading a `.3mf` whose printer name matches a documented printer binds the session to that
  printer automatically; a manual re-bind afterwards sticks, including across later `.3mf`
  uploads. A `.3mf` matching no documented
  printer, by neither name nor alias, leaves the session unbound until the user picks a
  printer; an unbound session offers no telemetry and no webcam capture.
- A `.3mf` whose printer name matches more than one documented printer leaves the session
  unbound and presents the matching candidates to pick from.
- On an already-bound session, a later `.3mf` matching nothing or matching ambiguously leaves
  the binding unchanged and shows a note that the new project's printer name did not match.
- A session with no `.3mf` can be bound manually, making telemetry and webcam capture
  available before any upload.
- Uploading a `.3mf` whose printer name matches an alias binds the session to that printer, and
  the session displays the printer's name, not the matched alias.
- A session on an evidence-only printer runs on uploaded evidence, shows the no-telemetry
  notice, and causes zero calls to any Moonraker endpoint.
- After an edit to the document, the next time any view showing the printer list is opened or
  refreshed it reflects the edit, with no service restart.
- A missing or unreadable document at the configured location produces a user-visible message
  naming the path and the problem, with the service still running and serving the UI — and,
  when a successful interpretation existed before the failure, the printer list from it still
  working; a readable document in which no printer can be identified produces a message saying
  exactly that; an unset location option likewise leaves the service running, with the UI
  naming the unset option.
- Editing the document to remove a bound session's printer leaves the session usable on its
  evidence, shows why telemetry is unavailable, and allows re-binding to a documented printer.
- With the backend unavailable (mocked), the previously interpreted printer list still works —
  including after a service restart — and an edited document is marked as awaiting
  re-interpretation with the backend named as the cause.
- On a fresh install with no working credentials, the printer list is empty with a message
  naming the backend as the blocker; once credentials work, the document is interpreted
  without a restart.

### R4. Sessions

A session addresses a single problem — one debugging investigation. A session is created by
sending its first message — usually the problem description, though it may also be just an
attachment, like a photo of the failure. The system derives the session's name from that
message without requiring the model — from its text when present, otherwise from the
attachment kind and time — so a name exists even when credentials are missing; the user is
never asked for a name, and can rename the session at any time.
Sessions can be deleted — deletion is permanent, so it requires an explicit confirmation
first, and a declined confirmation changes nothing — and can be archived: an archived session
is hidden from session lists and search results unless the show-archived selector is enabled,
and remains fully intact and reopenable; it can be unarchived at any time, returning it to the
default list and to search results. Sessions are searchable by name and by transcript
content. Sessions are listed, reopenable with their full history, and persist across service
restarts.

Acceptance criteria:
- Sending a new session's first message creates it with an automatically derived name, with no
  name requested from the user; renaming it persists. A photo-only first message also creates
  the session, with a name derived from the attachment kind and time.
- Deleting a session asks for confirmation; declining leaves the session untouched, and
  confirming removes it and its attached evidence — it appears in no list and no search result
  afterwards.
- Archiving a session hides it from the default list and from search results; the
  show-archived selector reveals it in both, and reopening it works unchanged.
- Unarchiving an archived session returns it to the default list and to search results.
- Searching for a term that occurs in a session's name, and one that occurs only in its
  transcript, each returns that session.
- After a service restart, the session list, each session's transcript, and its attached
  evidence are intact, and a reopened session can be continued.

### R5. Session inputs through MCP

The central inputs are an uploaded OrcaSlicer `.3mf` project and, when the investigation
needs one, a gcode file. The `.3mf` is optional but prompted for: it is the only source of
the actual settings the print used — the profile lives in it — so the agent asks for it
whenever slicer settings become relevant, and a recommendation that would need the real
values says so until one is uploaded. Uploads of any kind — photos, gcode included — are
accepted in any order from the session's first message (R4) onwards. As the investigation
iterates — apply a change, re-slice, print again — the session accepts further `.3mf` and
gcode uploads. Each upload receives a stable
identifier within the
session — its upload sequence and time, shown in the UI — because re-slices routinely reuse
the same filename. MCP queries address the newest upload of each kind unless an earlier one is
named by its identifier, the most recent `.3mf` is the one the session's current profile is
read from, and earlier uploads stay in the session as queryable evidence. Both file
types are too large for the model's context, so the model
never receives either file whole: an MCP server exposes targeted queries (settings, metadata,
excerpts, per-region gcode) and the model pulls only what it needs. The `.3mf` embeds the full
printer, process, and filament settings in use, so it is also where the session reads the
current profile from.

Acceptance criteria:
- With the canonical fixtures uploaded, the model's findings cite values that exist only inside
  the files (for example the project's 0.3 mm layer height or the gcode's first-layer bed
  temperature), demonstrating access through MCP queries.
- No model prompt at any point contains the full content of either uploaded file; the session
  transcript shows the MCP calls and their responses.
- An upload that is not a readable OrcaSlicer `.3mf` (or not readable gcode) is rejected with a
  user-visible message naming the file and the reason.
- In a session with no `.3mf`, a settings-relevant question makes the agent ask for the
  project, and a recommendation that would need the print's actual values is marked as
  awaiting them.
- After a second `.3mf` upload, findings cite the newest project's settings, and the earlier
  upload remains accessible in the session as evidence, answering MCP queries that name its
  upload identifier — even when both uploads share a filename.

### R6. Printer telemetry, read-only

The system reads printer state through Moonraker: print status, temperatures, position, the
Klipper configuration, and the Klipper log back to Klipper's last restart — all usable as
session evidence, shown in the UI and readable by the agent through MCP tools. The Klipper
log gets the same treatment as the R5 files: the model reads it through bounded, targeted
queries, and no prompt ever contains it whole. It never
issues a state-changing call; every recommended printer action (a console command, a macro) is
displayed for the user to execute themselves. When Moonraker is unreachable the session
continues on the remaining evidence and the UI shows the degraded-mode bar from
`brand/design-language.md`, naming what is unavailable.

Acceptance criteria:
- During a full session against a mocked Moonraker, the mock records only read/query calls —
  zero state-changing calls, asserted exactly.
- With Moonraker unreachable, a session still runs on uploaded evidence and the degraded-mode
  bar is visible, naming the printer connection as the missing capability.
- Telemetry shown as evidence carries provenance: source and age, per the brand doc.
- Print status, temperatures, position, and the Klipper configuration are each retrievable as
  session evidence through the mocked Moonraker.
- Klipper log entries back to Klipper's last restart are retrievable as session evidence; no
  model prompt contains the log whole, and the transcript shows the bounded queries used.

### R7. Webcam snapshots

Webcams are discovered through the bound printer's Moonraker — the printer document does not
configure them. A printer may have several webcams: the user chooses the camera in the UI, the
agent can request a specific camera or the default — the session's current camera choice,
shared across devices like all session state, or the first camera Moonraker lists when no
choice has been made — and every snapshot's provenance names the camera it came from.
The user can take a snapshot at any point while a webcam is discovered and its Moonraker is
reachable; with Moonraker unreachable no webcam is reachable either, so the snapshot control
is disabled with the reason shown and pictures come from the phone. The agent may capture
only while Moonraker reports an active print on that printer — printing or paused, since
pausing to inspect a defect is part of debugging: when no print is active, the agent takes no
image and instead asks the user for a picture — a snapshot or a phone photo; when the print
state cannot be determined because Moonraker is unreachable, it asks for a phone photo.
Without a discovered webcam the feature is simply absent — no errors, no dead controls. A
snapshot fetch that fails is a
transient failure: it surfaces with its cause, the session continues, and the user can retry;
it does not trigger the degraded-mode bar. Snapshots attach to the session as evidence with
source and capture-time provenance.

Acceptance criteria:
- With a mocked Moonraker reporting an active print and a discovered (mocked) webcam, a
  snapshot requested by the user and one requested by the agent each appear in the session
  with their provenance captions.
- With two discovered webcams, a snapshot can be taken from each, and each snapshot's
  provenance names its camera.
- With the print paused, agent snapshots remain available.
- With the printer idle, the agent captures no image and asks the user for a picture instead,
  while a user-requested snapshot still works.
- With Moonraker unreachable, the snapshot control is disabled with the reason shown, and the
  agent asks for a phone photo.
- With no webcam discovered, the UI offers no snapshot action and the session works normally.
- With a webcam whose fetch fails (mocked), the failure appears with its cause, the session
  continues, and a retry is offered.

### R8. Calibration guidance inside debugging sessions

There are no calibration-only sessions in v1 (future work). When a debugging session concludes
that a value needs recalibrating, the guidance is OrcaSlicer-guided: the exact OrcaSlicer
built-in calibration test to run (temperature, flow rate, pressure advance, retraction, or
maximum volumetric speed) and with which settings, using OrcaSlicer's exact setting names —
or, for tuning Klipper performs natively (pressure-advance tuning tower, input-shaper
calibration), the exact console command for the user to run. Results come back as photos or
user-reported readings within the same session; it concludes with the calibrated value and
exact instructions for updating the OrcaSlicer profile by hand, naming the filament and the
printer the value was established on.

Acceptance criteria:
- A session whose finding calls for calibration produces instructions containing the exact
  OrcaSlicer test name and settings, or the exact Klipper console command — precise enough to
  follow without external lookup.
- When a calibration result is fed back, the session states the concluded value and the exact
  OrcaSlicer profile settings to change, naming the filament and printer it applies to; the
  conclusion stays readable in the persisted session.

### R9. Debugging investigations

A debugging session takes a problem description and evidence — photos, the `.3mf`, gcode,
telemetry, webcam snapshots — and produces findings. Every finding follows the brand doc's
finding → evidence → change structure: what was observed, which evidence supports it, and the
exact OrcaSlicer setting change or user-executed printer action recommended. Findings are
labelled with the brand doc's certainty registers (established / hypothesis / external), and
every claim cites its evidence. The external register is fed by the agent consulting external
reference material — documentation for the printer itself, Klipper, OrcaSlicer, or a
filament — through its backend's own lookup capabilities; such claims name their source.
When a piece of evidence cannot support a conclusion — a photo too blurry, taken from the
wrong angle, missing the failure region — the agent says which conclusion it cannot support
and describes what a usable shot would look like, per the brand doc's evidence-quality
pattern; never a generic "image unclear".

Acceptance criteria:
- Given the canonical fixtures and a defect description, the session output contains at least
  one finding with all three parts: observation, cited evidence, exact recommended change.
- Findings display their certainty register in the UI.
- A finding resting on external reference material is labelled external and names its source.
- Given a photo that cannot support the asked conclusion (mocked replay), the response names
  that conclusion and describes the shot needed — not a generic complaint about image quality.

### R10. Diagnosability

The user can always tell what the system did and why something failed. Every model interaction,
including MCP tool calls and their results, is recorded in the session transcript and viewable
in the UI. The service writes a log to a documented location. Every failure the user can
encounter — missing or rejected subscription credentials, unreachable Moonraker, invalid
upload, failed webcam fetch, a missing or unreadable printer document, a document naming no
identifiable printer, an unset document location, a document re-read blocked by an unavailable
backend, a printer removed from under a bound session, a failed or usage-limited model call, a
device's lost live connection — surfaces in the UI with its cause; the one exception is a startup
failure, which precedes the UI and reports to the shell (R1).

Acceptance criteria:
- Each failure mode listed above has a user-visible message naming the cause; none dies silently
  or as a raw stack trace.
- The session transcript view shows, for any finding, the MCP calls and telemetry reads behind
  it.
- The service log exists at the documented path after a run and contains the requests served.

## 4. Constraints

- Host platform: Linux only. The service and its storage live on the user's machine.
- Model access: through the agent-backend abstraction; the only v1 backend is the Claude Agent
  SDK, subscription mode only.
- Printer interface: Moonraker HTTP API, read-only, Klipper printers only.
- Slicer: OrcaSlicer only — its `.3mf` project layout and gcode dialect.
- Single user; LAN access without authentication; the network is trusted.
- The UI follows `brand/design-language.md` (dark-first, its palette, type, and UI patterns),
  minus the confirmation-gate and emergency-stop patterns, which wait for printer writes
  (future work).

## 5. Future work

Recorded here so v1 non-goals are deliberate, not forgotten:

- Calibration-only sessions: dedicated guided-calibration flows that start from a printer and
  a filament rather than from a problem investigation.
- macOS and Windows as host platforms; Kubernetes as a deployment target.
- Non-Klipper printers (for example Marlin/OctoPrint, Bambu Lab, Prusa Connect).
- Slicers beyond OrcaSlicer (for example PrusaSlicer, Bambu Studio, Cura).
- Printer writes — running calibration commands and starting prints from Filamentary —
  activating the confirmation-gate and emergency-stop patterns in `brand/design-language.md`.
- Agent backends beyond the Claude Agent SDK — other vendors' agent SDKs plugged in behind the
  same abstraction.
- Authentication and safe exposure beyond the trusted LAN.
- Filamentary owning filament and process profiles, with the slicer consuming them — replacing
  v1's hand-applied profile changes.
- An in-page live camera viewfinder (requires TLS, so it arrives together with certificate
  provisioning).

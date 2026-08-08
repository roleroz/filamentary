# Harvest: release-suite verification list

Transient input to the interface-contracts phase — delete once the contracts carrying these
obligations are approved. Everything here is an assumption about *external reality* that no
mocked suite can verify, unified from the v1 module docs' "Provisional mocks — unverified"
sections. Entries state the external fact to verify, never a mechanism; how the system reacts
to each fact is for the contracts and module docs to decide fresh.

## The `claude-agent-acp` adapter and Claude Agent SDK

1. **Error shapes**: the real auth-failure, usage-limit, and outage error shapes match what the
   recorded ACP transcripts assume — and the usage-limit error actually carries a parseable
   reset time (the auto-resume experience depends on it; a degraded no-reset-time path must
   exist either way).
2. **Credential precedence**: with both `CLAUDE_CODE_OAUTH_TOKEN` and `~/.claude` present, the
   SDK uses the token and does *not* silently fall back to `~/.claude` when the token is
   rejected (R2.2's no-silent-fallback criterion rests on this).
3. **Credential caching**: whether the sidecar process caches `~/.claude` credentials — or
   their absence — for its lifetime, and whether that caching happens at spawn. Decides
   whether/when a process recycle is needed for no-restart login pickup (R2.2, R3).
4. **`~/.claude` layout**: which files' presence means "credentials exist" — a wrong probe
   assumption swaps which R2.2 recovery message the user sees.
5. **Cross-process `session/load`**: a freshly spawned sidecar can load a session created by an
   earlier process (ACP session state persists outside the sidecar's lifetime). Every
   recycle-then-continue and restart-then-continue path rides on this.
6. **HTTP MCP servers**: the adapter accepts a streamable-HTTP MCP server in `mcpServers` (ACP
   gates this behind a capability). A stdio-only adapter kills the entire toolset in production
   under a green mocked suite.
7. **Image content blocks**: an MCP tool result's image content actually reaches the model as
   vision input — the scripted agent consumes our server directly and can never prove what the
   real model sees. The photo-evidence loop (R7/R9) rests on this.
8. **Structured tool-result relay**: `session/update` notifications carry the MCP result
   payload intact rather than flattened to text (the UI's provenance rendering reads fields
   from the stored payload).
9. **Concurrent prompting**: two `session/prompt`s on different sessions in one sidecar process
   can be in flight at once (interpretation during an active turn, and two devices on two
   sessions, both assume it).
10. **Non-enumerated update kinds**: what the adapter actually emits beyond text/tool/turn
    events (thought chunks, plan updates) — so the relay policy is built against reality.
11. **Live interpretation**: a real model, given the interpretation prompt and a real printer
    document, returns parseable structured output (the recorded interpretation conversation is
    project-authored until then).
12. **Markup emission**: a real model, given the instruction assets, actually emits the
    finding/evidence-request markup in a parseable form — the plain-text fallback makes
    non-emission silent, so a live conversation must produce at least one parsed typed entry
    before first release (R9's register rendering).

## Moonraker and webcams

13. **Response shapes**: the Moonraker HTTP fixtures (written from the documented API) match a
    real instance's responses for every endpoint used.
14. **Relative snapshot URLs**: how real crowsnest/Mainsail installs report `snapshot_url`
    (commonly a relative path served by the printer host's web frontend, not the Moonraker
    port) and what base resolves it — a wrong rule 404s on the most common real setup (R7).
15. **klippy.log format**: the restart-marker line pattern the last-restart boundary keys on,
    and the log's rotation behaviour — fixture logs are project-authored, so the boundary
    detection is unverified until run against a real klippy.log (R6).
16. **Webcam snapshot bodies**: what a real camera endpoint returns (fixtures are
    project-authored images).

## Phone media

17. **Real iPhone HEIC**: a genuine phone-captured HEIC (HEVC-encoded HEIF) decodes through the
    chosen decoder stack — fixture HEICs are produced by the same stack that validates them, so
    a green suite is compatible with production rejecting every real phone photo (R1's
    mainstream first-message case).

## fileindex

None — no external service dependency; its canonical inputs are real OrcaSlicer files already
in `testdata/`.

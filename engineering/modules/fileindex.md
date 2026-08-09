# Module design — `fileindex`

## Decisions needed from you

This section holds only open items — a topic that is absent is settled and recorded in `engineering/decisions.md`, not forgotten.

**No open decisions.**

Changes since last approval: none — this version was approved on 2026-08-08. (The 2026-08-08 revision pruned the document to cite the evidence contract now that the contracts phase exists — the view bounds/continuation semantics, error vocabulary (no-upload answer, kind error, read-failure, unknown-ref), and identity rules are the contract's, no longer restated here; the module is session-blind per that contract (it never answers unknown-session); `printer_name` is the contract-defined post-commit read (never a validity input); unqualified resolution pins at view start with continuations refused only at service-run boundaries; `remove_session` is idempotent. The test plan gains cases for pinning, run-boundary refusal, `printer_name`'s three outcomes, and `remove_session` idempotence, and renames error outcomes to the contract vocabulary.)

## 1. Purpose

Per the architecture: validate and index every `.3mf` and gcode upload under its session-assigned identifier; serve bounded, targeted views to `mcp` and parsing services to `sessions` (printer-name extraction, upload validation); resolve unqualified queries to the newest upload of a kind; drop a session's indexed data on deletion. Never discard data for being large — large data gets a bounded view.

## 2. Internals

- **Upload registry** — keyed by the session-assigned identifier (session, sequence, kind, original filename, upload time) and storing the file's on-disk path, since lazy re-parsing after a restart or cache miss needs it and filenames are not unique (re-slices reuse them). Newest-upload resolution is a lookup of the highest sequence per (session, kind). The registry rows live in the SQLite database; the uploaded files on disk are owned by `sessions`, and `fileindex` reads them at the recorded path.
- **`.3mf` reader** — the file is a zip: `Metadata/project_settings.config` (JSON — the full printer/process/filament profile). The printer name for binding is the `printer_model` field ("Voron v2 350mm3" in the fixture) — not `printer_settings_id`, which names the settings profile and carries the nozzle suffix; a missing `printer_model` yields a summary without a printer name and the session simply stays unbound. The zip also holds `Metadata/plate_1.json` — the source of per-plate metadata: the first-layer time and plate bounding boxes (`slice_info.config` carries only a version header) — `Metadata/model_settings.config`, plate thumbnails, and the 3MF model. Geometry uses the 3MF Production extension: the root `3D/3dmodel.model` typically contains no mesh — only component references to object model files under `3D/Objects/` plus the build-item transforms (the canonical fixture is exactly this shape). The reader follows those references for the vertices and applies the build transforms for measurements. Parsing is lazy and cached per upload; the cache is rebuildable from the file, so nothing parsed needs separate persistence.
- **gcode reader** — parses the OrcaSlicer dialect: `HEADER_BLOCK`, `THUMBNAIL_BLOCK`s (base64 PNGs), the unmarked stats trailer between `EXECUTABLE_BLOCK_END` and `CONFIG_BLOCK_START` (comment lines such as `; estimated printing time (normal mode) = 19m 11s` — the only source of the total estimate), the trailing `CONFIG_BLOCK` (flattened settings, ~639 keys in the fixture), and an index over the executable body built from layer-change and feature-type markers (`;LAYER_CHANGE`, `;TYPE:`, `M73`) mapping layers and features to line ranges.
- **Views** (bounds, truncation signalling, continuation semantics, and pinning per the evidence contract — restated nowhere here):
  - settings: single keys, grouped sections, or the full profile of a `.3mf`/gcode config block; a diff view between two uploads for the iterate loop.
  - metadata: a per-upload summary — for a `.3mf`: printer name, filament, plate metadata including the first-layer time; for gcode: filament, layer count, and the total print estimate from its stats trailer — plus thumbnails. Summaries are per upload; there is no cross-upload merge view.
  - gcode: line windows, per-layer extraction, per-feature extraction, bounded pattern search.
  - mesh: numeric measurements (per-object bounding boxes, dimensions, footprint) and on-demand rendered images from named viewpoints (top, front, side, isometric), via a software renderer.

## 3. Interface

Rust API consumed by `sessions` and `mcp` (names indicative, signatures for the implementation task):

- `validate_and_index(upload_id, kind, original_filename, path) -> Result<UploadSummary, ValidationError>` — `sessions` passes the user-visible original filename separately from the on-disk path (whose basename it does not promise to preserve); the registry stores it, and rejection messages name it. `UploadSummary` carries headline metadata; the printer name is never a validity input and is read post-commit via `printer_name` (the evidence contract's operation, served from the parsed cache).
- `resolve(session, selector) -> Result<UploadId, QueryError>` — selector is `Newest(kind)` or an explicit identifier; unqualified resolution happens once, at view start, and continuations stay pinned to that upload per the contract.
- Query calls mirroring the views above, each returning a bounded, serializable result for MCP tool responses; continuation handles embed the service-run id, which is what makes cross-run refusal explicit.
- `remove_session(session)` — drops registry rows and caches for the session's uploads; idempotent, including for a session that never had an indexed upload.

## 4. Error handling and failure visibility

- `ValidationError` is structured and user-renderable (R5/R1 rejection messages name the file and reason): not a zip, required `.3mf` entries missing, settings unparseable, gcode unrecognizable.
- Query outcomes use the evidence contract's vocabulary: unknown-ref (an unresolvable identifier), the no-upload answer (an answer, not a failure — the normal no-`.3mf` state of R5), the kind error (a view addressed at the wrong kind), read-failure (an unreadable index or file, naming path and operation), plus out-of-range window. The module is session-blind: it never answers unknown-session — session existence is its callers' business, per the contract.
- Database errors (registry unavailable, locked, write failure, corrupt row) are structured and name the operation; a failed indexing write leaves no partial registry row behind.
- Session removal is transactional: a store failure during `remove_session` leaves either the prior state or full removal — never a half-removed session — is reported to `sessions` as a structured error, and succeeds on retry.
- Render failures (a degenerate or unrenderable mesh, a renderer fault) are structured errors naming the upload and viewpoint — never a panic, and never a poisoned cache.
- Filesystem failures at read time (file vanished, unreadable, truncated mid-read) are distinct from validation failures and name the path and operation.
- Every failure is logged through the service log with the upload identifier; nothing is swallowed.

## 5. Test plan

Contract tests (`test_contract_*`), enumerated over the interface promises:

1. The canonical `.3mf` fixture validates and its summary carries the printer name from `printer_model` — exactly "Voron v2 350mm3", not the `printer_settings_id` profile string — plus filament PETG and plate metadata.
2. The canonical gcode fixture validates; header, thumbnails, and config block are indexed.
3. Settings lookup returns exact fixture values from both kinds: the `.3mf` profile (layer height 0.3, nozzle 0.6) and the gcode config block (first-layer bed 80).
4. Grouped-section and full-profile settings views work against both kinds; each full profile is capped, flags truncation, and its continuation resumes until exhaustion — 653 keys for the `.3mf` fixture, ~639 for the gcode config block.
5. Settings diff between two uploads of the same kind — tested for a `.3mf` pair and a gcode pair — reports exactly the changed keys; a diff with more changes than the cap truncates at the cap with a working continuation.
6. Newest-upload resolution: with two `.3mf` uploads sharing a filename, `Newest` resolves to the higher sequence, and the explicit identifier still reaches the older one.
7. `Newest(kind)` with no upload of that kind returns the distinct no-upload answer, distinguishable from every failure.
8. Per-layer gcode extraction returns exactly the lines between layer markers; layer count matches the fixture header's total; a layer's output respects the cap and continues.
9. Per-feature extraction returns exactly the `;TYPE:`-delimited ranges for a named feature, capped with working continuation.
10. Line windows: an in-range window returns exactly the requested lines; a window wider than the cap is truncated at the cap with a working continuation; out of range returns `QueryError`, not empty success.
11. Bounded search: a pattern with many matches returns at most the cap, flags truncation, and its continuation resumes where the previous response stopped.
12. Thumbnails: the gcode fixture's decode to its declared 48x48 and 300x300; the `.3mf` fixture's plate PNGs decode to valid images whose dimensions are asserted from the decoded files.
13. Mesh measurements match the fixture's bounding box after following the Production-extension references and applying build transforms; rendered views return images for each named viewpoint.
14. `remove_session` makes every subsequent query for those uploads answer unknown-ref; a second invocation, and one for a session that never had an indexed upload, complete cleanly.
15. Validation rejections: not-a-zip, zip missing `.3mf` entries, unparseable settings, unrecognizable gcode — each yields its specific `ValidationError` carrying the passed original filename (not the on-disk basename) and leaves no registry row: subsequent queries for the rejected upload answer unknown-ref. A missing `printer_model` is not among these: it is valid input.
16. A view addressed at the wrong kind (a mesh query on a gcode upload) returns the kind error, decided on the identifier's discriminant before resolution.
17. The metadata-summary query (the view `mcp` serves) is per upload: the `.3mf` fixture's summary returns printer name, filament, plate metadata, and the 167.3 s first-layer time; the gcode fixture's summary returns the 19m11s total estimate from its stats trailer.
18. A crafted degenerate-mesh `.3mf` (empty or zero-area mesh, added to `testdata/`) yields a structured render error naming upload and viewpoint, measurements still answer where defined, and later queries on the upload succeed (no poisoned cache, no panic).
19. A crafted `.3mf` without `printer_model` (added to `testdata/`) validates successfully; `printer_name` answers the explicit no-name — absence is valid input, not a rejection — while the canonical fixture's `printer_name` answers "Voron v2 350mm3" by explicit ref, newest or not; an injected index-read fault yields read-failure, distinct from no-name.
19a. An unqualified view started before a newer upload commits continues over its original upload — pinned, never mixed; a continuation handle from a previous service run is refused explicitly.
20. Cold start: a fresh instance constructed over the same store and files — no in-memory state from indexing time — answers the same queries with identical results, exercising lazy re-parse from the recorded paths (the module's share of R4's restart criterion).

Failure-mode tests (`test_failure_<dependency>_<mode>`), enumerated over the declared dependencies — the filesystem (injected reader trait), the SQLite registry store (injected store trait), and the software renderer (injected renderer trait), all constructor parameters:

1. `filesystem_missing` — file missing at parse time.
2. `filesystem_permission` — file unreadable.
3. `filesystem_truncated` — torn write: parse fails cleanly, no partial index left behind (cleanup asserted on observable registry state).
4. `filesystem_fault_then_retry` — read error partway through a cached parse: the cache entry is not poisoned; a retry after the fault clears succeeds.
5. `store_unavailable` — registry unreachable at index time: structured error, no partial row, and re-indexing succeeds after recovery.
6. `store_busy` — locked/busy on query: structured error; a retry succeeds.
7. `store_write_failure` — write fails mid-index: cleanup asserted — no partial registry row remains, observable through the store fake's state.
8. `store_corrupt_row` — a corrupt registry row yields a structured error naming the upload, not a panic.
9. `renderer_failure` — a render fault yields a structured error naming the upload and viewpoint; the parse cache is not poisoned, and measurements still answer.
10. `store_removal_failure` — a store fault mid-`remove_session`: structured error returned, and observable state is all-or-nothing — every row still present or every row gone, never a half-removed session; the retry after the fault clears completes the removal.

Provisional mocks — unverified: none. The module has no external service dependency; the filesystem and store fakes are stateful trait implementations, not recorded mocks.

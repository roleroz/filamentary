# Filamentary — Design Language

Reference for anyone (human or agent) writing UI or documentation for Filamentary, the 3D printer
calibration and debugging assistant. Scope: brand, visual system, UI patterns, and voice.

The product is a diagnostic instrument that can send G-code to a hot machine. Everything below
serves two ideas: **evidence over opinion**, and **nothing happens without a deliberate yes**.

---

## 1. Brand

**Name:** Filamentary — *filament* + *elementary*. Detective register, never cute-detective.
Write it as one word, capital F, no tagline in product chrome. The descriptor `Print diagnosis`
may appear under the wordmark in marketing lockups only.

**Mark:** a magnifier whose handle is a strand of filament; the lens shows three layer lines.
Ring and handle take the surrounding text colour; the evidence lines are the accent.

```svg
<svg viewBox="0 0 120 120" fill="none" xmlns="http://www.w3.org/2000/svg">
  <circle cx="50" cy="46" r="30" stroke="currentColor" stroke-width="8"/>
  <path d="M34 38h32M34 48h20M34 58h28" stroke="#FF6A3D" stroke-width="5" stroke-linecap="round"/>
  <path d="M71 67c12 12-8 17 2 27 6 6 16 4 20-3" stroke="currentColor" stroke-width="8" stroke-linecap="round"/>
</svg>
```

Rules:

- **Ring and handle share one stroke weight** (8 at 120px; evidence lines 5). Round caps
  everywhere — the handle must read as extruded plastic.
- **The handle is anchored on the ring's centreline** — its first point is at 45° on the
  circle's own radius (30), never offset. Because the two strokes match in width, the handle's
  inner and outer edges land flush with the ring's, and nothing intrudes into the glass. When
  the ring thickens at small sizes, the handle thickens with it; the anchor never moves.
- **Clear space** on all sides equals the lens radius.
- **Size ladder** — detail drops as it shrinks, strokes thicken to compensate:
  | Size | Evidence lines | Ring + handle stroke |
  | --- | --- | --- |
  | ≥64px | 3 | 8 |
  | 32–63px | 2 | 9 |
  | 24–31px | 1 | 11 |
  | 16px | 0, handle truncated after the kink | 14 |
- **Icon tiles** (favicon, app icon) are white, with the mark in `--dark` and the evidence
  lines in `--accent` — the only spot of colour. No orange tile; it reads heavy under a mask.
- **One-colour cutdowns** (engraving, printed parts, monochrome bars): everything in one colour,
  evidence lines at 45% opacity so they never out-shout the ring.
- Ready-made files live in `brand/` — `mark/` (on-dark, on-light, mono, currentColor, and
  size-optimised 32/24/16), `lockup/`, `favicon.svg`, `app-icon.svg`. Use the size-optimised
  files rather than scaling the primary.
- Never: rotate it, add a gradient, put it in a circle badge, restyle the handle as a straight
  stick, move the handle anchor off the ring, or set the wordmark in a font other than
  Bricolage Grotesque.

---

## 2. Colour

Dark-first. The product is used in a workshop and on a phone at arm's length in mixed light.

| Token | Hex | Use |
| --- | --- | --- |
| `--bg` | `#121317` | App background |
| `--surface` | `#191B21` | Cards, panels, message bubbles |
| `--surface-2` | `#0D0E12` | Recessed wells, image/artifact backgrounds |
| `--border` | `#262932` | Hairlines, card edges |
| `--border-strong` | `#333744` | Secondary button outlines, focus rings |
| `--ink` | `#F2ECE2` | Primary text, light-mode background |
| `--ink-2` | `#A7ABB5` | Body text, descriptions |
| `--ink-3` | `#8B8F9A` | Labels, metadata |
| `--ink-4` | `#5C616E` | Captions on dark wells, disabled |
| `--accent` | `#FF6A3D` | The filament. Primary action, active state, brand |
| `--ok` | `#3DDC97` | Connected, approved, complete |
| `--warn` | `#F2B705` | Stale snapshot, degraded mode, weak evidence |
| `--danger` | `#FF4D4D` | Emergency stop, dangerous action, error |

Discipline:

- **One accent.** `--accent` is filament orange and belongs to the primary action on a view.
  If two things on screen are orange, one of them is wrong.
- Status colours are for status, never for decoration. `--ok` never fills a button.
- No gradients, no coloured shadows, no tinted glass. Depth comes from `--surface` steps and
  hairline borders.
- Text on `--accent` is always `#121317`, never white.
- Light surfaces (`--ink` as a background) exist for print, PDF summaries, and the marketing
  site. In-app light mode is not a goal.

---

## 3. Type

- **Bricolage Grotesque** — UI and headings. 700/800 for headings, 500–600 for UI.
  Letter-spacing `-0.03em` at 24px and above, `0` below.
- **JetBrains Mono** — anything the machine said or the user must copy verbatim: G-code,
  config snippets, file names, layer numbers, measurements, IDs. Also small uppercase labels at
  `letter-spacing: 0.16em`.

| Role | Size / weight |
| --- | --- |
| Page title | 32–44 / 800 |
| Section title | 18–20 / 700 |
| Body | 15–17 / 400, line-height 1.5 |
| Secondary | 13–14 / 400, `--ink-2` |
| Label (mono, caps) | 10–12 / 600, `--ink-3` |
| Code block (mono) | 13 / 400 |

Never below 13px on phone. Tap targets never below 44px. `text-wrap: pretty` on prose.

---

## 4. Layout and shape

- 4px base grid; spacing steps 4 / 8 / 12 / 16 / 24 / 32 / 48 / 64.
- Radii: 8 (chips, inputs) · 12 (buttons) · 20 (cards) · 26 (sheets, phone panels) · 999 (pills).
- Sibling groups are laid out with flex/grid + `gap`, never with margins on each child.
- Phone first: single column, thumb-reachable primary action, no horizontal scroll. Desktop is
  the same column plus a session rail — not a different information architecture.
- Content max width for prose: ~72ch.

---

## 5. UI patterns

These encode requirements from the spec. Do not invent alternatives.

**Confirmation gate.** Every printer write is a distinct, unmistakable card: the exact commands
in mono, one sentence of plain-language effect, then `Approve` (accent, filled) / `Reject`
(outlined). One card per action — there is never an "approve all". A dangerous action
(unhomed motion, out-of-limit move, cold extrude, heater safety change) gets a `--danger` left
rule, a `--danger` label reading what could go wrong, and a longer press target.

**Emergency stop.** Always visible while a printer is connected, top-right of the app frame,
outside the conversation. `--danger`, mono uppercase label, never scrolls away, never disabled.

**Provenance.** Any machine value shown carries its source and age as a mono caption:
`live · read 12s ago`, `saved config · 3d ago`, `snapshot · 2026-07-14`. A stale source renders
the caption in `--warn`. The knowledge-base document is never quoted as a current value.

**Certainty.** Distinguish three registers visually and in copy: *established* (plain text,
cites the artifact), *hypothesis* (prefixed "Likely —", `--ink-2`), *external* (a bordered
citation block with the source domain, always ranked below first-hand evidence).

**Evidence quality.** When a photo cannot support a conclusion, the system says which conclusion
and what a usable shot looks like — an actionable request card, not a generic "image unclear".

**Degraded mode.** When a printer is unreachable, a persistent bar states it plainly and lists
what is unavailable. Questions about runtime state are declined, never answered from saved
values.

**Progress.** Uploads, ingestion, indexing and model responses always show motion. A multi-second
silent wait is a defect. Streaming text, determinate bars for uploads, mono byte counts.

**Mode badge.** `LOCAL` (`--ok`) or `EXPOSED` (`--accent`) pill in the header at all times.

---

## 6. Voice

Direct, technical, unhurried. The reader owns the printer and knows Klipper.

- Lead with the finding, then the evidence, then the change. Never a list of generic causes.
- Name things exactly: settings by their OrcaSlicer name, values with units, layers by number
  and Z height, files by name.
- Say what you don't know. "The photo shows the shift but not the surface finish" beats a
  confident guess.
- No exclamation marks, no emoji, no "Great question", no apologising. Errors state what
  happened, what it means, and what to do.
- Filament recommendations always name **both** the filament and the printer they were
  established on.
- Documentation: second person, present tense, sentence case headings, mono for every path,
  command, and setting name. Short paragraphs; tables over bullet lists when comparing.

---

## 7. Quick checklist

- [ ] One accent-coloured primary action per view
- [ ] Machine values carry source + age
- [ ] Every printer write has its own approval card
- [ ] Emergency stop reachable
- [ ] Mono for anything the user copies
- [ ] Works one-handed at 390px wide
- [ ] Nothing below 13px / 44px tap target
- [ ] Established vs. hypothesised vs. external is visually distinct

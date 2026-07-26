# Filamentary brand assets

Full rules: [design-language.md](design-language.md).

```
mark/    filamentary-mark-on-dark.svg      primary, dark UI
         filamentary-mark-on-light.svg     primary, light / print
         filamentary-mark-mono-*.svg       one-colour (engraving, printed parts)
         filamentary-mark-currentcolor.svg inline in HTML, inherits text colour
         filamentary-mark-32/24/16.svg     size-optimised — use these, do not scale the primary
lockup/  horizontal (header, marketing) and stacked (splash), on dark and on light
favicon.svg    32px, white tile, dark mark + orange evidence lines
app-icon.svg   512px, white tile, safe for iOS/Android masking
```

Notes:

- Colours: filament orange `#FF6A3D`, ink `#F2ECE2`, dark `#121317`.
- Clear space on every side = the lens radius.
- Ring and handle carry the same stroke weight, and the handle is anchored on the ring's
  centreline (45°, radius 30) so their edges run flush.
- Lockup wordmarks are live text in **Bricolage Grotesque 700**. Install the font, or convert
  the text to outlines before handing the file to anyone who won't have it.
- Below 64px use the size-optimised marks: strokes thicken and evidence lines drop (3 → 2 → 1 → 0).

# Sabé — Logo & Brand Assets

A logo system for the **Sabé** astromech control system, a *Bean & Bear Droid Works* build.
The mark pairs an engraved serif wordmark with a minimal sensor‑dome symbol; the acute on
the **é** is rebuilt as a burgundy "jewel" that reappears as the dome's sensor eye.

---

## What's in this package

```
svg/      Vector masters — infinitely scalable, text is OUTLINED (no font needed)
png/      High‑res raster (transparent), 2000px on the long edge
png/tiles Avatar / favicon tiles at 1024 → 32 px
```

Every mark is provided in three finishes:

| suffix     | use on…                | colors                          |
|------------|------------------------|---------------------------------|
| `-color`   | light backgrounds      | ink + burgundy eye/jewel        |
| `-reverse` | dark backgrounds       | cream + burgundy eye/jewel      |
| `-black`   | 1‑color / **silkscreen** | solid black (recolor as needed) |

### Marks
- `sabe-lockup-A` — dome + wordmark. **Primary / everyday lockup.**
- `sabe-lockup-B` — lockup over "ASTROMECH CONTROL SYSTEM". Formal / masthead.
- `sabe-lockup-C` — lockup with a hairline divider. Slightly more engineered.
- `sabe-lockup-D` — prominent dome (symbol leads). Tight / small contexts.
- `sabe-wordmark` — "Sabé" on its own.
- `sabe-dome`     — the sensor‑dome symbol on its own. **Best for PCB silkscreen & favicons.**

### Tiles (avatars)
- `sabe-tile-ink` — dark tile, cream dome, burgundy eye. **Default GitHub / Astromech.net avatar.**
- `sabe-tile-burgundy` — accent‑forward.
- `sabe-tile-light` — light‑mode, hairline keyline.

---

## Palette

| name      | hex       | role                          |
|-----------|-----------|-------------------------------|
| Ink       | `#211C17` | primary line / text           |
| Burgundy  | `#7A2230` | accent — sensor eye / é jewel |
| Cream     | `#ECE6D8` | reverse / knockout            |
| Paper     | `#F3EEE2` | light backgrounds, light tile |
| Keyline   | `#C9C0AE` | light‑tile border             |

## Type
- **Wordmark:** Cormorant Garamond, SemiBold (600).
- **Descriptor:** JetBrains Mono, tracked +0.34em, all caps.
- In the vector files the text is **converted to outlines**, so the fonts are *not*
  required to open or print the logos. Both fonts are free (OFL) if you ever want to set
  matching copy: Cormorant Garamond & JetBrains Mono on Google Fonts.

## Usage notes
- **Clear space:** keep at least the height of the dome's base clear on all sides.
- **Minimum size:** dome / tile down to ~16 px; full lockups ≥ 120 px wide for legibility.
- **PCB silkscreen:** use a `-black` SVG (or the dome). Single color, no fine fills — it
  holds at small sizes. Recolor the black to whatever silk ink the board uses.
- Don't recolor the eye/jewel to anything but the burgundy (or the mono color); don't
  stretch, rotate, or add effects to the marks.

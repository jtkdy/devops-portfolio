---
name: ppt-guide
description: Dark, restrained 16:9 slide design system set only in Pretendard, with fixed header positions and dense body layouts. Use whenever creating or editing a PPT, PPTX, slide deck, or presentation in this vault.
---

# Slide Design System

## Overview

Near-black canvas, white text, hairline borders, generous type contrast. The deck reads like a research lab announcing its work, not a marketing pitch: no gradients, no atmospheric backdrops, no decorative imagery, no shadows.

**Hard rules:**
- Output 16:9 slides only. Never produce 4:3, A4, or any other ratio.
- Use Pretendard for every piece of text. Never use another font, including for code, numbers, charts, or tables.
- Do not place a logo anywhere. Do not reserve space for one.
- Keep the chapter, title, and subtitle at the exact same position on every content slide.
- Fill the body zone. Do not leave the lower part of a slide empty.

## Canvas & Fixed Layout

All values are in points on a 960 x 540 pt canvas (13.333 x 7.5 in).

| Zone | x | y | Width | Height | Content |
|---|---|---|---|---|---|
| Chapter | 48 | 24 | 864 | 16 | Current chapter name |
| Title | 48 | 42 | 864 | 40 | Slide title, one line |
| Subtitle | 48 | 84 | 864 | 22 | One-line supporting statement |
| Divider | 48 | 114 | 864 | 1 pt line | `hairline` |
| Body | 48 | 126 | 864 | 372 | Slide content, ends at y 498 |
| Footer | 48 | 508 | 864 | 14 | Source on the left, page number on the right |

**Position rules:**
- Build these zones into the slide master or a shared helper, so every slide inherits identical coordinates.
- Never move, resize, or center the header zones to suit the content. Change the content instead.
- Top-anchor chapter, title, and subtitle text with zero internal margins.
- Keep the title and subtitle on one line. Rewrite a long one shorter. Never wrap it or shrink it.
- Every content slide has all three header elements. If a slide has no natural subtitle, write its one-line takeaway there.
- Only the cover and section divider slides may use a larger title. They keep the chapter label at its fixed position and the title left edge at x 48.

**Body grid:** 12 columns with 16 pt gutters.

| Layout | Column width |
|---|---|
| 2-up | 424 pt |
| 3-up | 277.33 pt |
| 4-up | 204 pt |

## Density

The body zone must look deliberately full, while staying orderly.

- The lowest body element must end between y 474 and y 498.
- Stretch side-by-side cards to one shared height, and end them at y 498 when they are the lowest element.
- Do not leave more than about 25% of a card empty. Enlarge its content or add supporting content.
- Keep 16 pt gaps between body blocks and 16 pt card padding. Density comes from more content, not from removing gaps.

**When a slide runs short, add real content in this order:**
1. Details from the source material that were left out: figures, conditions, reasons, steps.
2. A metric strip: 3 or 4 key numbers across the full width.
3. A supporting table, chart, or diagram of the same content.
4. A full-width takeaway card at the bottom of the body zone, stating the conclusion or its effect.
5. A larger layout: bigger chart, bigger metrics, more line spacing within the allowed range.

**Limits:**
- Never invent facts, figures, or sources to fill space. Use only the supplied material, and use step 5 when it runs out.
- Never fill space with decorative shapes, stock images, placeholder text, or repeated statements.
- When content overflows, cut it or split the slide. Never go below the minimum type sizes.

## Colors

### Surface
- **Canvas** (`#0a0a0a`): background of every slide. No light mode.
- **Canvas Card** (`#191919`): default card fill.
- **Canvas Soft** (`#1a1c20`): table header and takeaway card fill.
- **Canvas Mid** (`#363a3f`): nested surfaces and code backgrounds.
- **Hairline** (`#212327`): 1 pt dividers and card borders.

### Text
- **Ink** (`#ffffff`): titles, headings, key numbers.
- **Body** (`#dadbdf`): subtitles and body text.
- **Mute** (`#7d8187`): captions, sources, page numbers. Do not use for body text.

### Accent
Use accents rarely and only to carry meaning, such as one highlighted series or one key number. Use at most one accent per slide unless a chart needs more series.

- **Sunset Orange** (`#ff7a17`)
- **Sunset Soft** (`#ffc285`)
- **Dusk Purple** (`#7c3aed`)
- **Twilight** (`#c4b5fd`)
- **Breeze Blue** (`#a0c3ec`)
- **Midnight** (`#0d1726`): illustration and diagram backgrounds only.

## Typography

### Font Family

Pretendard is the only family. Each weight has its own typeface name:

| Weight | Typeface name | Use |
|---|---|---|
| 400 | `Pretendard` | Default for all text, including titles |
| 500 | `Pretendard Medium` | Chapter labels, chips, table headers |
| 600 | `Pretendard SemiBold` | Card headings, inline emphasis |

Thin, ExtraLight, Light, Bold, ExtraBold, and Black are installed but not part of this system. Do not use them unless the user asks.

**PPTX requirements:**
- Set the typeface on the `latin`, `ea`, and `cs` fonts of every run. Korean text uses `ea`, and it falls back to another font if `ea` is unset.
- Do not apply the bold attribute. Select the weight through the typeface name.
- Set the theme major and minor fonts to `Pretendard` as well.
- Letter spacing below is in points. In PPTX XML, `spc` is in hundredths of a point (-0.6 pt is `spc="-60"`).

### Hierarchy

| Token | Size | Typeface | Line spacing | Letter spacing | Color | Use |
|---|---|---|---|---|---|---|
| `cover-title` | 60pt | Pretendard | 105% | -1.5pt | Ink | Cover slide title |
| `divider-title` | 44pt | Pretendard | 110% | -1.1pt | Ink | Section divider title |
| `slide-title` | 30pt | Pretendard | 100% | -0.6pt | Ink | Slide title |
| `metric` | 36pt | Pretendard | 100% | -0.7pt | Ink | Key numbers |
| `subtitle` | 14pt | Pretendard | 100% | 0 | Body | Slide subtitle |
| `chapter` | 11pt | Pretendard Medium | 100% | 1.1pt | Ink | Chapter label, uppercase for Latin text |
| `card-heading` | 16pt | Pretendard SemiBold | 120% | -0.2pt | Ink | Card and block headings |
| `body` | 13pt | Pretendard | 130-150% | 0 | Body | Default body text |
| `body-sm` | 11pt | Pretendard | 130-150% | 0 | Body | Dense body and table cells |
| `caption` | 10pt | Pretendard | 120% | 0 | Mute | Chart labels and notes |
| `footer` | 9pt | Pretendard | 100% | 0 | Mute | Source and page number |

**Minimum sizes:** 11pt for body text, 10pt for captions and chart labels, 9pt for the footer only.

### Principles
- Regular weight carries the system. Size and tight tracking create display emphasis.
- Use SemiBold only for card headings and short inline emphasis, never for a slide title.
- Keep the negative tracking on display sizes. Neutral tracking loses the precise look.
- Left-align all text except numbers in table columns, which are right-aligned.

## Shapes & Depth

- **Card radius:** 6 pt.
- **Pill radius:** full height. Use for status and category chips only.
- **Full-bleed bands and tables:** 0 pt.
- **Elevation:** no shadows, glows, or gradients. A 1 pt `hairline` border carries all separation.

## Components

**`card`**: fill `#191919`, 1 pt `hairline` border, 6 pt radius, 16 pt padding. Holds a `card-heading` and `body` or `body-sm` text.

**`chip`**: transparent fill, 1 pt border in white at 25% opacity, full pill radius, `caption` text in Ink, 4 pt vertical and 10 pt horizontal padding. A white fill with `#0a0a0a` text marks the one primary chip on a slide, at most.

**`metric-strip`**: 3 or 4 equal `card`s in one row, each with a `metric` number above a `caption` label.

**`takeaway-card`**: full-width `card` with `#1a1c20` fill at the bottom of the body zone, one or two lines of `body` text.

**`table`**: header row with `#1a1c20` fill and `Pretendard Medium` 11pt text, `body-sm` cells, 1 pt `hairline` row lines, no vertical lines, no zebra stripes. Stretch row heights so the table ends in the allowed density range.

**`chart`**: transparent background, `hairline` gridlines, `caption` labels, white and gray series, one accent for the highlighted series. No 3D effects or chart shadows.

**`code-block`**: fill `#363a3f`, 6 pt radius, 12 pt padding, `body-sm` text in Pretendard.

**`diagram`**: `card`-style nodes with 1 pt straight connectors in `#7d8187`, on the canvas or a `#0d1726` panel.

## Slide Patterns

Each pattern uses the fixed header and footer, and changes only the body zone.

- **Cover:** `cover-title` and `subtitle`, left aligned, with date or author in `caption` on the footer line.
- **Section divider:** `divider-title` and a short list of the chapter's topics, left aligned.
- **Metrics + detail:** `metric-strip` on top, 2-up or 3-up `card`s below, ending at y 498.
- **Compare:** 2-up `card`s of equal height, with a `takeaway-card` below.
- **Table + insight:** `table` across 8 columns, `card` with the interpretation across 4 columns.
- **Chart + insight:** `chart` across 8 columns, 2 or 3 stacked `card`s across 4 columns.
- **Process:** 3-up or 4-up numbered `card`s, with a `takeaway-card` or `table` below.

## Do's and Don'ts

### Do
- Use `#0a0a0a` as the only slide background.
- Reuse the same master for every slide, so that flipping through the deck shows no header movement.
- Vary the body pattern to suit the content.
- Check every slide before delivery: font names, header coordinates, title on one line, no overflow, and the lowest body element within y 474-498.

### Don't
- Don't use any font other than Pretendard, or let any text fall back to a system font.
- Don't add a logo, a light mode, shadows, gradients, or decorative imagery.
- Don't bold slide titles.
- Don't center titles or body text.
- Don't invent content to reach the density target.

---
name: ppt-guide
description: Quiet, near-monochrome 16:9 slide design system set only in Pretendard, with one blue accent, fixed header positions, and dense body layouts. Use whenever creating or editing a PPT, PPTX, slide deck, or presentation in this vault.
---

# Slide Design System

## Overview

Near-invisible design that lets the content speak, like a gallery wall behind an artifact. White and off-white surfaces, near-black type, soft gray tiles for emphasis, and a single quiet blue. Typography is confident but quiet. There are no gradients, shadows, or decorative frames.

**Hard rules:**
- Output 16:9 slides only. Never produce 4:3, A4, or any other ratio.
- Use Pretendard for every piece of text. Never use another font, including for code, numbers, charts, or tables.
- Do not place a logo anywhere. Do not reserve space for one.
- Keep the chapter, title, and subtitle at the exact same position on every content slide.
- Fill the body zone. Do not leave the lower part of a slide empty.
- Use one accent color only: Blue `#0066cc`. Do not add a second accent, including per company, product, or chapter.

## Canvas & Fixed Layout

All values are in points on a 960 x 540 pt canvas (13.333 x 7.5 in).

| Zone | x | y | Width | Height | Content |
|---|---|---|---|---|---|
| Chapter | 48 | 24 | 864 | 16 | Current chapter name |
| Title | 48 | 42 | 864 | 40 | Slide title, one line |
| Subtitle | 48 | 84 | 864 | 22 | One-line supporting statement |
| Divider | 48 | 114 | 864 | 1.5 pt line | Ink `#1d1d1f`, separates the header from the body |
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
2. A stat row: 3 or 4 key numbers across the full width.
3. A supporting table, chart, or diagram of the same content.
4. A full-width takeaway card at the bottom of the body zone, stating the conclusion or its effect.
5. A larger layout: bigger chart, more line spacing within the allowed range.

**Limits:**
- Never invent facts, figures, or sources to fill space. Use only the supplied material, and use step 5 when it runs out.
- Never fill space with decorative shapes, stock images, placeholder text, or repeated statements.
- When content overflows, cut it or split the slide. Never go below the minimum type sizes.

## Colors

### Accent
- **Blue** (`#0066cc`): the only color in the system. Use it on at most one key number or one highlighted chart series per slide, and on links.

### Surface
- **Canvas** (`#ffffff`): background of every slide and of `card`.
- **Parchment** (`#f5f5f7`): `card-soft` fill, table header fill, and `tag` fill.
- **Soft Tile** (`#e8e8ed`): `tile-soft` fill.
- **Hairline** (`#e0e0e0`): 1 pt card borders and table row lines.

### Text
- **Ink** (`#1d1d1f`): titles, headings, key numbers, and the header divider.
- **Ink Muted** (`#333333`): body text, subtitles, chapter labels, captions, and table headers.
- **Fine Print** (`#7a7a7a`): page numbers and legal fine print only.

## Typography

### Font Family

Pretendard is the only family. Each weight has its own typeface name:

| Weight | Typeface name | Use |
|---|---|---|
| 400 | `Pretendard` | Body text, subtitles, chapter labels, units, captions |
| 600 | `Pretendard SemiBold` | All titles, card headings, key numbers, table headers, tags, inline emphasis |

Weight 500 is deliberately absent. Other Pretendard weights are not part of this system. Do not use them unless the user asks.

**PPTX requirements:**
- Set the typeface on the `latin`, `ea`, and `cs` fonts of every run. Korean text uses `ea`, and it falls back to another font if `ea` is unset.
- Do not apply the bold attribute. Select the weight through the typeface name.
- Set the theme major and minor fonts to `Pretendard` as well.
- Letter spacing below is in points. In PPTX XML, `spc` is in hundredths of a point (-0.3 pt is `spc="-30"`).
- The `spcPct` column is the value to write in PPTX. PowerPoint's 100% is about 119% of the font size for Pretendard, so it is not the same as the size-based line height in the `Line` column.

### Hierarchy

| Token | Size | Typeface | Line | `spcPct` | Letter spacing | Color | Use |
|---|---|---|---|---|---|---|---|
| `cover-title` | 42pt | SemiBold | 1.07 | 90% | -0.3pt | Ink | Cover slide title |
| `divider-title` | 30pt | SemiBold | 1.10 | 92% | -0.3pt | Ink | Section divider title |
| `slide-title` | 30pt | SemiBold | 1.20 | 100% | -0.3pt | Ink | Slide title |
| `card-heading` | 16pt | SemiBold | 1.20 | 100% | -0.2pt | Ink | Card and block headings |
| `metric` | 24pt | SemiBold | 1.20 | 100% | -0.3pt | Ink | Key numbers |
| `metric-unit` | 12pt | Pretendard | 1.20 | 100% | 0 | Ink Muted | Units beside a key number |
| `subtitle` | 14pt | Pretendard | 1.20 | 100% | 0 | Ink Muted | Slide subtitle |
| `chapter` | 11pt | Pretendard | 1.20 | 100% | 0 | Ink Muted | Chapter label |
| `body` | 13pt | Pretendard | 1.47 | 123% | -0.2pt | Ink Muted | Default body text |
| `body-sm` | 11pt | Pretendard | 1.47 | 123% | 0 | Ink Muted | Dense body and table cells |
| `caption` | 10pt | Pretendard | 1.43 | 120% | 0 | Ink Muted | Metric labels, chart labels, notes |
| `footer` | 9pt | Pretendard | 1.20 | 100% | 0 | Fine Print | Source and page number |

**Minimum sizes:** 11pt for body text, 10pt for captions and chart labels, 9pt for the footer only.

### Principles
- Use tight negative tracking on sizes of 13pt and up. Never use it at 11pt or below.
- Do not tighten body line height below 1.47.
- Use 600, not 700, for headlines.
- Keep key numbers small enough that they support the content. Do not raise `metric` above 24pt.
- Left-align all text except numbers in table columns, which are right-aligned.

## Shapes & Depth

| Token | Value | Use |
|---|---|---|
| `radius-lg` | 13.5 pt | `card`, `card-soft`, `tile-soft` |
| `radius-pill` | Full height | `tag` only |

- Use no other radius.
- Keep slides flat. Do not use shadows, glows, or gradients on anything.
- A change of surface is the first way to separate blocks. Add a 1 pt `hairline` border only on white cards and table rows.

## Components

**`card`**: fill `#ffffff`, 1 pt `hairline` border, `radius-lg`, 16 pt padding. Holds a `card-heading` and `body` or `body-sm` text.

**`card-soft`**: fill `#f5f5f7`, no border, `radius-lg`, 16 pt padding. Use for quieter supporting content.

**`tile-soft`**: fill `#e8e8ed`, Ink and Ink Muted text, `radius-lg`, 24 pt padding, no border. Do not use near-black fills on slides, because they feel heavy on a white canvas. Use for the one thing a slide or row is about. Do not use more than three per slide.

**`stat-row`**: 3 or 4 equal `card`s in one row, 76 pt tall, each with a `metric` number and optional `metric-unit` above a `caption` label.

**`takeaway-card`**: full-width `card-soft` or `tile-soft` at the bottom of the body zone, one or two lines of `body` text.

**`tag`**: full pill shape, fill `#f5f5f7`, `Pretendard SemiBold` 10pt Ink text. Use it to name a company, product, or topic. All tags look the same.

**`table`**: header row with `#f5f5f7` fill and `Pretendard SemiBold` 11pt Ink Muted text, `body-sm` cells, 1 pt `hairline` row lines, no vertical lines, no zebra stripes. Stretch row heights so the table ends in the allowed density range.

**`chart`**: transparent background, `hairline` gridlines, `caption` labels, black and gray series, Blue for the one highlighted series. No 3D effects or chart shadows.

**`code-block`**: fill `#f5f5f7`, `radius-lg`, 12 pt padding, `body-sm` text in Pretendard.

**`diagram`**: `card`-style nodes with 1 pt straight connectors in `#333333`, on the canvas or a `card-soft` panel.

## Slide Patterns

Each pattern uses the fixed header and footer, and changes only the body zone.

- **Cover:** `cover-title` and `subtitle`, left aligned, with date or author in `caption` on the footer line.
- **Section divider:** `divider-title`, a `tag` for the chapter, and a short list of the chapter's topics, left aligned.
- **Tile + detail:** `tile-soft` across 4 columns, 2 or 3 stacked `card`s across 8 columns.
- **Metrics + detail:** `stat-row` on top, 2-up or 3-up `card`s below, ending at y 498.
- **Compare:** 2-up `card`s of equal height, with a `takeaway-card` below.
- **Table + insight:** `table` across 8 columns, `card` with the interpretation across 4 columns.
- **Chart + insight:** `chart` across 8 columns, 2 or 3 stacked `card`s across 4 columns.
- **Process:** 3-up or 4-up numbered `card`s, with a `takeaway-card` or `table` below.

## Do's and Don'ts

### Do
- Use `#ffffff` as the background of every slide.
- Reuse the same master for every slide, so that flipping through the deck shows no header movement.
- Vary the body pattern to suit the content.
- Check every slide before delivery: font names, header coordinates, title on one line, no overflow, and the lowest body element within y 474-498.

### Don't
- Don't use any font other than Pretendard, or let any text fall back to a system font.
- Don't add a logo, a dark slide background, shadows, gradients, or decorative imagery.
- Don't use any color other than Blue as an accent, or use Blue on large surfaces.
- Don't mix radii. Cards are `radius-lg` and tags are pills.
- Don't center titles or body text.
- Don't invent content to reach the density target.

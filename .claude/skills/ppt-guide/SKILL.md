---
name: ppt-guide
description: Clean white 16:9 slide design system set only in Pretendard, with black ink, saturated identity cards, fixed header positions, and dense body layouts. Use whenever creating or editing a PPT, PPTX, slide deck, or presentation in this vault.
---

# Slide Design System

## Overview

Stark white canvas with deep-black type: confident, technical, almost editorial. Saturated identity cards break the monochrome base the way album covers would, each one declaring the personality of a single topic. Everything else stays quiet: thin borders, soft gray fills, no decoration.

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
2. A stat row: 3 or 4 key numbers across the full width.
3. A supporting table, chart, or diagram of the same content.
4. A full-width takeaway card at the bottom of the body zone, stating the conclusion or its effect.
5. A larger layout: bigger chart, more line spacing within the allowed range.

**Limits:**
- Never invent facts, figures, or sources to fill space. Use only the supplied material, and use step 5 when it runs out.
- Never fill space with decorative shapes, stock images, placeholder text, or repeated statements.
- When content overflows, cut it or split the slide. Never go below the minimum type sizes.

## Colors

### Surface
- **Canvas** (`#ffffff`): background of every content slide and of `card`.
- **Surface** (`#f7f8fa`): `card-soft` fill and table header fill.
- **Surface Soft** (`#f2f3f5`): quieter section divisions and nested blocks.
- **Hairline** (`#e5e7eb`): 1 pt dividers and card borders.
- **Hairline Soft** (`#eaecf0`): table row lines.
- **Primary** (`#0a0a0a`): `card-dark` fill and primary `chip` fill.

### Text
- **Ink** (`#0a0a0a`): titles, headings, key numbers.
- **Charcoal** (`#222222`): body text.
- **Slate** (`#45515e`): subtitles and secondary text.
- **Steel** (`#5f5f5f`): captions, table headers, sources, page numbers.
- **On Dark** (`#ffffff`): text on `identity-card` and `card-dark`.

Do not use lighter grays for text. They fall below a readable contrast on white.

### Identity
Each topic of a deck, such as a product, company, or chapter, gets one fixed identity color. Keep that assignment through the whole deck. Choose only from this list.

- **Coral** (`#ff5530`): the most attention-grabbing color. Give it to the most important topic.
- **Magenta** (`#ea5ec1`)
- **Blue** (`#1456f0`)
- **Purple** (`#a855f7`)

**Rules:**
- Use identity colors only on `identity-card`, `badge-identity`, one highlighted chart series, or one key number.
- Never use them on body text or large surfaces other than `identity-card`.
- Use at most one `identity-card` color per content slide. A slide that compares topics may use one card per topic.
- White text on Coral, Magenta, and Purple is low in contrast. Use 16pt or larger on these fills. Use a caption size only on Blue.

### Support
- **Blue Deep** (`#1d4ed8`): text on `badge-info`, and links.
- **Blue 200** (`#bfdbfe`): `badge-info` fill.
- **Success Text** (`#1ba673`) on **Success Background** (`#e8ffea`): `badge-success` only.
- **Midnight Blue 700** (`#17437d`): diagram connectors that need to stand out from gray.

## Typography

### Font Family

Pretendard is the only family. Each weight has its own typeface name:

| Weight | Typeface name | Use |
|---|---|---|
| 400 | `Pretendard` | Body text, key numbers' units |
| 500 | `Pretendard Medium` | Chapter labels, subtitles, chips, table headers |
| 600 | `Pretendard SemiBold` | All titles, card headings, key numbers, badges |
| 700 | `Pretendard Bold` | Short inline emphasis in body text only |

Other Pretendard weights are not part of this system. Do not use them unless the user asks.

**PPTX requirements:**
- Set the typeface on the `latin`, `ea`, and `cs` fonts of every run. Korean text uses `ea`, and it falls back to another font if `ea` is unset.
- Do not apply the bold attribute. Select the weight through the typeface name.
- Set the theme major and minor fonts to `Pretendard` as well.
- Letter spacing below is in points. In PPTX XML, `spc` is in hundredths of a point (-0.6 pt is `spc="-60"`).
- The `spcPct` column is the value to write in PPTX. PowerPoint's 100% is about 119% of the font size for Pretendard, so it is not the same as the size-based line height in the `Line` column.

### Hierarchy

| Token | Size | Typeface | Line | `spcPct` | Letter spacing | Color | Use |
|---|---|---|---|---|---|---|---|
| `cover-title` | 60pt | SemiBold | 1.10 | 92% | -1.5pt | Ink | Cover slide title |
| `divider-title` | 42pt | SemiBold | 1.10 | 92% | -1.1pt | Ink | Section divider title |
| `slide-title` | 30pt | SemiBold | 1.20 | 100% | -0.75pt | Ink | Slide title |
| `card-heading` | 16pt | SemiBold | 1.20 | 100% | 0 | Ink | Card and block headings |
| `metric` | 24pt | SemiBold | 1.20 | 100% | -0.5pt | Ink | Key numbers |
| `metric-unit` | 12pt | Pretendard | 1.20 | 100% | 0 | Slate | Units beside a key number |
| `subtitle` | 14pt | Medium | 1.20 | 100% | 0 | Slate | Slide subtitle |
| `chapter` | 11pt | Medium | 1.20 | 100% | 0.5pt | Steel | Chapter label |
| `body` | 13pt | Pretendard | 1.35-1.50 | 115-125% | 0 | Charcoal | Default body text |
| `body-sm` | 11pt | Pretendard | 1.35-1.50 | 115-125% | 0 | Charcoal | Dense body and table cells |
| `caption` | 10pt | Pretendard | 1.40 | 115% | 0 | Steel | Chart labels and notes |
| `footer` | 9pt | Pretendard | 1.20 | 100% | 0 | Steel | Source and page number |

**Minimum sizes:** 11pt for body text, 10pt for captions and chart labels, 9pt for the footer only.

### Principles
- Use one family with weight discipline: 400 for body, 500 for labels, 600 for headings, 700 for short emphasis.
- Keep the negative tracking and tight leading on display sizes. They create the editorial look.
- Keep key numbers small enough that they support the content. Do not raise `metric` above 24pt.
- Left-align all text except numbers in table columns, which are right-aligned.

## Shapes & Depth

| Token | Value | Use |
|---|---|---|
| `radius-sm` | 4.5 pt | Table containers and code blocks |
| `radius-md` | 12 pt | `card`, `card-soft`, `metric` cards |
| `radius-hero` | 24 pt | `identity-card` and `card-dark` |
| `radius-full` | Full height | `chip` and all badges |

- Pair `radius-hero` identity cards with `radius-md` white cards on the same slide. The radius contrast is the visual signature.
- Keep slides flat. A 1 pt `hairline` border carries separation. Do not use shadows, glows, or gradients.

## Components

**`card`**: fill `#ffffff`, 1 pt `hairline` border, `radius-md`, 16 pt padding. Holds a `card-heading` and `body` or `body-sm` text.

**`card-soft`**: same as `card` with `#f7f8fa` fill and no border. Use for quieter supporting content.

**`identity-card`**: solid identity color fill, `On Dark` text, `radius-hero`, 24 pt padding, no border. Holds a large statement or topic name in `card-heading` size or larger. Use for the one thing the slide is about.

**`card-dark`**: fill `#0a0a0a`, `On Dark` text, `radius-hero`, 24 pt padding. Use for a closing statement or takeaway.

**`stat-row`**: 3 or 4 equal `card`s in one row, 76 pt tall, each with a `metric` number and optional `metric-unit` above a `caption` label.

**`takeaway-card`**: full-width `card-soft` or `card-dark` at the bottom of the body zone, one or two lines of `body` text.

**`chip`**: full pill shape, `caption` size in `Pretendard Medium`, 4 pt vertical and 10 pt horizontal padding. Primary is `#0a0a0a` fill with white text, at most one per slide. Secondary is transparent fill with 1 pt `#0a0a0a` border and Ink text. Tertiary is white fill with 1 pt `hairline` border and Ink text.

**`badge-identity`**: identity color fill, white `Pretendard SemiBold` 10pt text, full pill shape. Use it to tag a topic on a white card.

**`badge-info`**: `#bfdbfe` fill, `#1d4ed8` `Pretendard SemiBold` 10pt text, full pill shape.

**`badge-success`**: `#e8ffea` fill, `#1ba673` `Pretendard SemiBold` 10pt text, full pill shape.

**`table`**: header row with `#f7f8fa` fill and `Pretendard SemiBold` 11pt Steel text, `body-sm` cells, 1 pt `hairline-soft` row lines, no vertical lines, no zebra stripes. Stretch row heights so the table ends in the allowed density range.

**`chart`**: transparent background, `hairline` gridlines, `caption` labels, black and gray series, one identity color for the highlighted series. No 3D effects or chart shadows.

**`code-block`**: fill `#f2f3f5`, `radius-sm`, 12 pt padding, `body-sm` text in Pretendard.

**`diagram`**: `card`-style nodes with 1 pt straight connectors in `#5f5f5f`, on the canvas or a `card-soft` panel.

## Slide Patterns

Each pattern uses the fixed header and footer, and changes only the body zone.

- **Cover:** `cover-title` and `subtitle`, left aligned, with date or author in `caption` on the footer line.
- **Section divider:** `divider-title`, a `badge-identity` for the chapter, and a short list of the chapter's topics, left aligned.
- **Identity + detail:** `identity-card` across 4 columns, 2 or 3 stacked `card`s across 8 columns.
- **Metrics + detail:** `stat-row` on top, 2-up or 3-up `card`s below, ending at y 498.
- **Matrix:** 3-up or 4-up `identity-card`s of equal height, one per topic, with a `takeaway-card` or `table` below.
- **Compare:** 2-up `card`s of equal height, with a `takeaway-card` below.
- **Table + insight:** `table` across 8 columns, `card` with the interpretation across 4 columns.
- **Chart + insight:** `chart` across 8 columns, 2 or 3 stacked `card`s across 4 columns.
- **Process:** 3-up or 4-up numbered `card`s, with a `takeaway-card` or `table` below.

## Do's and Don'ts

### Do
- Use `#ffffff` as the background of every slide.
- Reuse the same master for every slide, so that flipping through the deck shows no header movement.
- Keep each topic on its assigned identity color for the whole deck.
- Vary the body pattern to suit the content.
- Check every slide before delivery: font names, header coordinates, title on one line, no overflow, and the lowest body element within y 474-498.

### Don't
- Don't use any font other than Pretendard, or let any text fall back to a system font.
- Don't add a logo, a dark slide background, shadows, gradients, or decorative imagery.
- Don't use identity colors on body text, or use them only to decorate.
- Don't use sharp-cornered or partly rounded chips and badges. They are always full pills.
- Don't center titles or body text.
- Don't invent content to reach the density target.

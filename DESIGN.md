# Hikma design system

ヒクマ · حكمة · Hikma · *an open, auditable question bank for standardized exams.*

**This is the reference design system for Hikma implementations: a complete set of colour, type, spacing, and component conventions, published so that an implementation has a working default instead of a blank page.** The single most important point: it is a reference, not a requirement. Any implementation may replace it with a visual identity of its own, with one firm limit: that identity must not imitate the examining body's. Below: how to use or replace this system, that limit in full, then the principles, marks, colour, type, spacing, components, accessibility, licensing, and the token contract.

> **Status: reference.** `core` is a specification repository and ships no stylesheet, font, or asset files. This document is the source of the values; an implementation builds them into its own tokens.

## Contents

- [1. Reference only](#1-reference-only)
- [2. Do not adopt the examining body's visual identity](#2-do-not-adopt-the-examining-bodys-visual-identity)
- [3. Principles](#3-principles)
- [4. Brand marks](#4-brand-marks)
- [5. Colour](#5-colour)
- [6. Type](#6-type)
- [7. Space, radius, motion](#7-space-radius-motion)
- [8. Components](#8-components)
- [9. Accessibility](#9-accessibility)
- [10. Licensing and attribution](#10-licensing-and-attribution)
- [11. Using the tokens](#11-using-the-tokens)

## 1. Reference only

Nothing in this document binds an implementation. It exists because most of the design decisions a question bank needs are the same across exams, and making them again in every repository wastes effort and produces four interfaces that agree on nothing. Three ways to use it, all fine:

- **Adopt it.** Build the tokens in section 11 from the values here and use the component conventions as written. This is the path the reference implementation takes.
- **Fork it.** Keep the structure, meaning the token names, the shape of the ramps, the accessibility floor, and the rule that colour never carries meaning alone, and replace the hues and typefaces with your own.
- **Replace it.** Design something unrelated. You owe this document nothing.

Section 2 does not change across those three, because it is not a design preference. It is the visual half of the naming rule in [README.md](README.md#contributing), and it keeps the project clear of a claim it has never made.

Two rules bind every implementation and survive any redesign: **colour never carries meaning alone** (see section 3) and **body text meets WCAG 2.1 AA** (see section 9). Those are accessibility floors, not matters of taste, and a question bank that fails them is broken however it looks.

## 2. Do not adopt the examining body's visual identity

Hikma targets exams whose names, logos, and materials belong to other organisations. The project is independent, is not affiliated with or endorsed by any examining body, and does not want to appear otherwise. Using an exam name to say which exam a pipeline targets is a normal and lawful use of the name. Making an implementation look like the body's own product is a different act, and it is the one that turns a descriptive use into an implied claim of affiliation. Do not do it, even a little, and even where you think you could defend it.

Concretely, in any implementation, do not:

- **Reproduce the body's logo, wordmark, seal, or certificate design** anywhere, including favicons, avatars, social cards, README heroes, slide decks, and merchandise.
- **Rebuild the body's palette**, whether by copying its published hex values or by sampling its site, prospectus, answer sheets, or certificates for a near match.
- **Copy its typographic identity**, meaning its logotype lettering, its distinctive typeface pairing, or the way it sets its own level and score labels.
- **Imitate the layout of its official surfaces**, including the score report, the certificate, the answer sheet, and the registration page, closely enough that a screenshot could be mistaken for the real thing.
- **Use official exam paper or answer sheet artwork as a template**, in the product or in marketing.
- **Set the exam name in the body's own lettering.** The only correct treatment is your own interface typeface, as ordinary text. The name describes scope, it is not a badge.
- **Imply endorsement in visual shorthand**, for example a checkmark, seal, ribbon, or "official" styling next to the exam name.

And do:

- **Choose from your own ramp first**, then compare the result with the body's public material and change anything that lands close. When you are unsure, take the more different hue, never the closer one. No design goal requires looking like theirs.
- **Keep the non-affiliation disclaimer on every public surface**, in the interface itself and not only in a README. Section 10 gives the wording.
- **Attribute the marks you name.** Exam names are trademarks of their owners, they appear in an implementation only to describe which exam it targets, and the README says so.

This system applies the rule to itself. The level badge hues in section 8 come from Hikma's own ramp, and they were chosen so that the order can be seen, cool at the easiest band and warm at the hardest, not to match any official colour scheme. The mark in section 4 is a letter from the project's own name. Neither borrows from any exam body, and anything that replaces them should be able to say the same.

## 3. Principles

1. **Evidence over assertion.** Every question shows its provenance: model, prompt version, run ID, git SHA. The `ProvenanceBlock` is the main component of the system, not a footnote.
2. **Difficulty is measured, not declared.** Levels come from persona runs that measure them. Never style a level as a claim of authority.
3. **Two licences, never one.** Dataset and code are governed separately and must never appear as one combined mark.
4. **Colour never carries meaning alone.** Every ability band shows its label; every status shows a word.

## 4. Brand marks

| Asset | File | Use |
|---|---|---|
| Monogram | `logo-monogram.svg` | Avatars, favicons, ≥16px |
| Wordmark | `logo-wordmark.svg` | Headers, README hero |
| Mono / one-colour | `*-mono.svg` | Stamps, embroidery, single-ink print |

The files ship with the reference implementation, not with this repository. `core` carries the specification of the mark, which is what follows, so that a redrawn or re-exported version can be checked against it.

The mark is **Ḥ**: the Latin letter H carrying a dot below it. That is the standard romanization of the Arabic **ح** (*ḥāʾ*), the first letter of حكمة, "wisdom". The dot is not decoration and it is not optional. Remove it and the letter is a plain H and the name is spelled wrong. The idea: **a claim above, and the evidence that supports it below.**

The Arabic ح carries no dot of its own. A dot below it makes ج (*jīm*) and a dot above it makes خ (*khāʾ*). The dot belongs to the transliteration alone, so it appears on the Latin Ḥ and never on the Arabic letter.

Sizes: the monogram holds down to 16px and the wordmark down to 96px wide. Below 20px the inner border drops and the H and its dot carry the mark alone. Keep the dot in rose-350, never rosewood, because rosewood on plum-charcoal is 1.52:1 and disappears at that size (see section 9). Clear space on every side is the height of the dot.

Never: drop the dot, move it above the letter, put a dot on the Arabic ح, skew or rotate the mark, re-weight the strokes, round the counter of the H into a circle, set the wordmark in another typeface, or place the mark on a colour outside the palette.

This mark identifies the Hikma project. An implementation that keeps it says that it is a Hikma pipeline, which is accurate. An implementation that replaces this design system should replace the mark too rather than restyle it, because a changed Hikma mark still reads as the Hikma project rather than as your fork.

## 5. Colour

Four locked founding hues, expanded into named ramps. Locked steps marked ◆, never replace them.

### Ink (cool neutral: plum-charcoal to grey-green)

| Step | Hex | | Step | Hex |
|---|---|---|---|---|
| 990 | `#201E25` | | 400 | `#8A8A93` |
| 950 | `#26242B` | | 350 | `#8B9291` |
| 900 | `#2E2C34` | | 300 | `#A9A9B0` |
| ◆850 | `#37353E` | | 250 | `#C1C5C4` |
| 800 | `#3E3C46` | | ◆200 | `#D3DAD9` |
| ◆700 | `#44444E` | | 175 | `#DCE2E0` |
| 600 | `#55555F` | | 150 | `#E1E6E4` |
| 550 | `#63636D` | | 125 | `#E7EBEA` |
| 500 | `#6B6B75` | | 100 | `#ECEFEE` |
| | | | 050 | `#F4F6F5` |
| | | | 000 | `#FBFCFC` |

### Rose (the single warm hue, accent and brand fill)

`#3E3131` 900 · `#52403F` 800 · `#5F4A4A` 700 · ◆`#715A5A` 600 · `#866D6D` 500 · `#9C8282` 400 · `#B39B9B` 350 · `#C9B6B6` 300 · `#DECFCF` 200 · `#EFE6E6` 100

> **rose-600 is a fill, not a text colour.** On dark surfaces use rose-350 `#B39B9B`; on light surfaces use rose-800 `#52403F`.

### Semantic hues (tuned into the palette, not stock)

| Role | Fill | On-dark text | On-light text | Tint |
|---|---|---|---|---|
| Error | `#B85A4E` | `#E7A99F` | `#9A3E33` | `#F3DED9` |
| Warning | `#96702F` | `#D8B173` | `#7A5A22` | `#EFE2C9` |
| Success | `#4F7A64` | `#9DC1A9` | `#3B6650` | `#DCE9E0` |
| Info | `#556B8C` | `#A6BAD2` | `#3F5678` | `#DCE3ED` |

### Theme roles

Light is **not** an inversion of dark. Surfaces climb the ink ramp; text drops to plum, not black.

| Token | Dark | Light |
|---|---|---|
| `--surface-app` | ink-990 `#201E25` | ink-175 `#DCE2E0` |
| `--surface-sunken` | ink-950 `#26242B` | ink-125 `#E7EBEA` |
| `--surface-card` | ink-850 `#37353E` | white `#FFFFFF` |
| `--text-strong` | ink-050 `#F4F6F5` | ink-900 `#2E2C34` |
| `--text-body` | ink-200 `#D3DAD9` | ink-850 `#37353E` |
| `--text-muted` | ink-400 `#8A8A93` | ink-600 `#55555F` |
| `--text-faint` | ink-500 `#6B6B75` | ink-550 `#63636D` |
| `--border-default` | ink-700 `#44444E` | ink-350 `#8B9291` |
| `--accent` | rose-600 `#715A5A` | rose-700 `#5F4A4A` |
| `--accent-text` | rose-350 `#B39B9B` | rose-800 `#52403F` |

## 6. Type

| Role | Family | Notes |
|---|---|---|
| Latin | **Plus Jakarta Sans** | Bundled locally, OFL. Variable, 200 to 800. |
| Mono | **JetBrains Mono** | IDs, hashes, git SHAs, all provenance values |
| Arabic | **Amiri** | Naskh. Used for حكمة in running text. The wordmark is Latin, set in Plus Jakarta Sans Bold at -0.5 tracking. |
| Exam script | per implementation | The reference implementation targets the JLPT and uses **Noto Sans JP** at 400 / 500 / 700. An implementation for another exam substitutes the family its script needs. |

**Scale** (1.25 major third, whole px): 11 · 12 · 13 · 14 · **16** · 18 · 20 · 25 · 31 · 39 · 49 · 61

**Line height:** `1.15` tight · `1.3` snug · `1.55` normal · **`1.85` CJK** · `2` loose

**Mixed-script rhythm.** Latin body sets at 1.55. **Any block containing CJK text sets the whole block to 1.85**, and the two line heights never mix inside one paragraph. Match weights, not sizes: Jakarta 700 pairs with Noto 700. Amiri runs one step larger than its Latin neighbour.

## 7. Space, radius, motion

- **Spacing** (8px baseline): 4 · 8 · 12 · 16 · 24 · 32 · 48 · 64 · 96
- **Radius:** 2 `xs` · 4 `sm` · 6 `md` · 10 `lg` · 999 `full`
- **Borders:** 1px hair · 1.5px thin · 2px thick
- **Motion:** 90ms fast · 160ms base · 260ms slow, `cubic-bezier(.2,0,.2,1)`

## 8. Components

| Component | Purpose |
|---|---|
| `Button` | `primary` · `secondary` · `ghost` · `danger` |
| `Input` | Sunken field, hairline border, rosewood focus ring |
| `LevelBadge` | One per ability band. Order encoded by hue temperature; the band label always shows |
| `LicenseBadge` | Split badge: left label (`DATASET`/`CODE`), right value |
| `ProvenanceBlock` | **Hero.** Model, prompt version, run ID, git SHA, check results |
| `CodeBlock` | Filename/language header + copy action |
| `QuestionCard` | Stem, options, reveal state, evidence footer |

Band order runs cool to warm, easiest to hardest. The reference implementation maps the five JLPT bands as sage `#5F8A6F` N5 · steel `#5E7A9E` N4 · ink `#8A8A93` N3 · ochre `#B0863F` N2 · terracotta `#D98C81` N1. An implementation with a different number of bands spreads the same cool-to-warm run across its own bands rather than reusing these five, and it picks its hues under the limit in section 2.

## 9. Accessibility

All body text meets **WCAG 2.1 AA (4.5:1)**; UI boundaries meet **3:1**.

| Foreground | Background | Ratio |
|---|---|---|
| `#D3DAD9` grey-green | `#37353E` dark card | 8.50:1 |
| `#F4F6F5` ink-050 | `#37353E` dark card | 11.11:1 |
| `#B39B9B` rose-350 | `#37353E` dark card | 4.64:1 |
| `#37353E` plum | `#FFFFFF` light card | 12.06:1 |
| `#52403F` accent text | `#FFFFFF` light card | 9.70:1 |
| `#63636D` faint | `#E7EBEA` light well | 4.94:1 |
| `#8B9291` border | `#FFFFFF` light card | 3.17:1 |
| `#FFFFFF` white | `#715A5A` accent fill | 6.34:1 |

**Known failure, do not use:** `#715A5A` rosewood on `#44444E` slate is **1.52:1**. Rosewood is never text on an ink surface; use `--accent-text` (rose-350 dark, rose-800 light).

## 10. Licensing and attribution

- **Two licences, two badges.** A dataset and the code that generated it are governed separately, and the interface renders them as two distinct `LicenseBadge` components. Never blend them into one mark, and never let a badge for one imply anything about the other.
- **Each repository sets its own.** `core` is MIT and covers this document. An implementation chooses its own licence for its code and its dataset, and the MIT grant here does not extend to it. The reference implementation publishes its dataset under CC BY 4.0 and its pipeline code under MIT, which is a choice for that repository to state in its own words, not a rule this document imposes.
- **Never label something "open source" that is not.** If a badge says MIT, Apache, or "open source", the licence file has to back it. The status word for source-available code is **"Source-available."**
- **Fonts:** Plus Jakarta Sans, Noto Sans JP, Amiri, JetBrains Mono, all OFL. Ship the licence text alongside any font you bundle.
- **Non-affiliation, on every public surface.** The wording is "Not affiliated with, endorsed by, or sponsored by [examining body]", naming the body the implementation targets. It belongs in the interface, not only in a README, and it must not be styled until it disappears.
- **Exam names are trademarks of their owners** and appear in an implementation only to describe which exam it targets. See section 2 for what that permits and what it does not.

## 11. Using the tokens

`core` ships no stylesheet. What follows is the naming contract an implementation builds against, so that components written for one Hikma repository read correctly in another.

```html
<link rel="stylesheet" href="styles.css">
<html data-theme="dark">   <!-- or data-theme="light" -->
```

```css
.thing {
  background: var(--surface-card);
  color: var(--text-body);
  border: 1px solid var(--border-default);
  border-radius: var(--radius-md);
  padding: var(--space-4);
}
```

Token families: `--surface-*` and `--text-*` and `--border-*` and `--accent*` from section 5, `--font-*` and `--text-size-*` and `--leading-*` from section 6, `--space-*` and `--radius-*` and `--border-width-*` and `--duration-*` from section 7. Theme switching is a `data-theme` attribute on the root element, never a class, so that a server-rendered page can set it without a flash.

Never hardcode a hex in product code. If a value you need has no token, add the token.

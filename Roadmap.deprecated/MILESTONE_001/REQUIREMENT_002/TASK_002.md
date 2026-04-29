# TASK_002 — City Label Styles

**Status**: Done

## Overview

Add city, town, village, and hamlet label layers to both light and dark map styles, with visual indicators (dots/squares) that differentiate place importance and capital status.

## Repository

- **Repo**: swayrider
- **Subfolder**: `assets/map/styles/`
- **Tech**: MapLibre GL JS style JSON

## Scope: Existing

- `assets/map/styles/light.json` — Place dot and label layers
- `assets/map/styles/dark.json` — Place dot and label layers

## Technical Specification

### Dot/Circle Layers (visual indicators)

| Layer ID | Source Layer | Filter | Minzoom | Circle Size |
|----------|-------------|--------|---------|-------------|
| `places-capital-square` | places | capital=yes or capital=2 | 7 | 6–9px |
| `places-dot-city-major` | places | place=city, no population or empty | 8 | 4–7px |
| `places-dot-city-minor` | places | place=city, population < 25000 | 9 | 3–5px |
| `places-dot-town` | places | place=town | 10 | 3–4px |
| `places-dot-village` | places | place=village | 12 | 2–3px |
| `places-dot-hamlet` | places | place=hamlet/suburb | 14 | 1.5–2.5px |

### Label Layers (text)

| Layer ID | Filter | Minzoom | Font | Placement |
|----------|--------|---------|------|-----------|
| `places-labels-city-major` | city, pop≥25000 or unknown | 8 | Open Sans Regular | top-offset |
| `places-labels-city-minor` | city, pop<25000 | 9 | Open Sans Regular | top-offset |
| `places-labels-town` | town | 10 | Open Sans Regular | top-offset |
| `places-labels-village` | village | 12 | Open Sans Regular | top-offset |
| `places-labels-hamlet` | hamlet/suburb | 14 | Open Sans Regular | top-offset |

### Styling

- Dots: black fill, white stroke
- Capital square: larger, distinct color
- Text: halo outline for readability, size interpolated by zoom
- All styles applied to both light.json and dark.json

## Dependencies

- TASK_001 (data must be in tiles)

## Acceptance Criteria

- [x] Dot/circle layers for each place type
- [x] Label layers with name text
- [x] Capitals have distinct visual indicator
- [x] Styles in both light and dark themes
- [x] Text readability via halo

## Testing Notes

- Verify layer IDs match source-layer `places`
- Check filter expressions against exported OSM tags
- Test readability at all zoom levels

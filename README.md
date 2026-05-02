# Local Newspaper PDF Content Classification System (LNPCCS)

A Python-based, deterministic, local-only pipeline for classifying newspaper PDF content into:

- `article_text`
- `photograph_image`
- `advertisement`

It supports vector PDFs (native text layer) and scanned/raster PDFs (OCR fallback), then produces machine-readable and human-readable reports with traceable decision metadata.

## Goals

- Local execution only (no runtime external APIs)
- Windows 10/11 compatibility
- Deterministic, reproducible outputs
- Multi-page editions (20–100 pages)
- No manual labeling during runtime

## High-Level Pipeline

1. Input PDF(s)
2. PDF normalization and rendering
3. Page type determination (text layer vs image-only)
4. Layout segmentation into candidate regions
5. Region feature extraction (geometry, text, visual, typography, structure)
6. Region classification (`article_text` / `photograph_image` / `advertisement`)
7. OCR fallback for non-text-layer pages/regions
8. Page/document aggregation
9. Report generation (JSON + human-readable), logs, optional trace artifacts

## Functional Scope

### Ingestion

- Single PDF mode and batch directory mode
- Corrupt/encrypted/unreadable file detection
- Metadata extraction (filename, pages, dimensions, producer, creation date)
- Deterministic document ID (`sha256(file_bytes)` + canonical filename)

### Page Processing

- Sequential page processing by default
- Coordinate normalization to `[0,1]` space
- Text-layer detection
- Raster render at configurable DPI (`render_dpi`, default `300`)

### Layout Segmentation

- Non-overlapping region extraction
- Overlap resolution through deterministic precedence rules
- Hierarchy: `page -> block -> optional sub-block`

### Classification

- Region classes: `article_text`, `photograph_image`, `advertisement`
- Confidence score + rationale/trace
- Ambiguity flag + deterministic tie-breaker

### Text Extraction

- Native PDF extraction for vector blocks
- OCR for scanned pages/raster blocks
- Region-to-text lineage preservation

### Aggregation & Reporting

- Per-page and per-document counts/areas/ratios
- Full-page ad detection
- OCR usage metrics
- JSON report + TXT/HTML/Markdown summary
- Ambiguity and error summaries

## Non-Functional Requirements

- Determinism: pinned versions, fixed seeds, stable ordering
- Performance target: <=2.5s/page average at 300 DPI on 8-core CPU (baseline)
- Reliability: page-level failure isolation
- Portability: Python 3.10+, Windows-friendly local dependencies
- Observability: JSONL logs, stage timings, error taxonomy
- Privacy/security: local-only data path

## Architecture

### A. Ingestion & Preprocessing

- Parse metadata
- Render page image
- Text-layer coverage estimate
- Image preprocess (denoise, deskew, contrast normalization; binarization for OCR path)

### B. Layout Segmentation

- Column detection (whitespace projection)
- Block detection per column
- Contour/rule-line detection
- Region refinement via merge/split

### C. Candidate Detection

- Text-rich candidates
- Image-like candidates
- Ad-like candidates (promo cues + border + high visual density)

### D. Classification

Deterministic scoring channels:

- `article_score`
- `image_score`
- `ad_score`

Tie-break policy:

1. Hard ad indicators -> `advertisement`
2. High text density + low promo density -> `article_text`
3. High entropy + low text density -> `photograph_image`
4. Else ambiguous + fallback to highest score

### E. OCR & Native Text

- Native text for vector regions
- OCR for raster regions/pages
- OCR confidence handling + low-confidence flags

### F. Aggregation

- Class counts and area ratios
- Ad/editorial ratio
- Full-page ad count
- OCR usage metrics

### G. Outputs

- JSON schema-compliant document analysis
- Human-readable report
- Optional debug artifacts (rendered pages, overlays, snippets, decision traces)

## Data Model Overview

### Region (`ContentRegion`)

Includes deterministic `region_id`, normalized/pixel bbox, area, class, confidence, ambiguity metadata, scores, features, text source (`native_pdf | ocr | none`), OCR confidence, and decision trace.

### Page (`PageAnalysis`)

Includes dimensions, rotation, text-layer flag, regions, per-class stats/ratios, full-page ad flag, stage timings, warnings/errors.

### Document (`DocumentAnalysis`)

Includes document identifiers/hashes, page outcomes, aggregated statistics, ambiguity/error summaries, runtime summary, and config snapshot.

## Ambiguity Policy

Explicit ambiguous tagging is required for difficult content such as infographics, sponsored editorial-style content, comics/cartoons, and teaser-like house ads.

The system stores top competing class scores and uses a deterministic fallback class for aggregate metrics while preserving ambiguity visibility in outputs.

## Error Taxonomy

- `INPUT_ERROR`
- `RENDER_ERROR`
- `OCR_ERROR`
- `SEGMENTATION_ERROR`
- `CLASSIFICATION_ERROR`
- `OUTPUT_ERROR`

System behavior: fail soft at page level, continue processing remaining pages, and include structured error records.

## Suggested Project Layout

```text
lnpccs/
  cli.py
  config.py
  pipeline/
    ingest.py
    preprocess.py
    segment.py
    classify.py
    ocr.py
    aggregate.py
    report.py
  models/
    schemas.py
  rules/
    ad_lexicon_en.txt
  tests/
```

## Configuration (Example)

```yaml
render_dpi: 300
language: eng
ocr_enabled: true
classification_mode: rule_based
thresholds:
  ad_confidence: 0.75
  image_vs_ad_margin: 0.15
  text_density_min: 0.30
outputs:
  json: true
  markdown: true
artifacts:
  keep_rendered_pages: false
  keep_segmentation_overlays: false
  keep_text_snippets: false
  keep_decision_traces: true
```

## Testing Strategy

- Unit tests: ingestion, segmentation determinism, rule triggers, OCR extraction, aggregation math, schema validation
- Integration tests: vector-only/scanned-only/mixed/non-standard layouts
- Determinism tests: same input/config run 3x => identical hashes
- Validation dataset target: >=30 docs, >=1000 pages, diverse publishers and scan qualities

## Baseline Targets

- Region-level macro F1 >= 0.85 (in-domain scans)
- Full-page ad detection precision >= 0.95

## Current Status

This repository currently contains the formalized architecture/specification. Implementation modules are not yet scaffolded in code.

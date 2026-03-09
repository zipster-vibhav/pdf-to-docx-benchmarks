# PDF to DOCX Conversion Benchmarks

Benchmarking 5 different Python libraries for converting 60-page PDFs to DOCX, evaluated against a 90-second Celery task timeout constraint.

Each library was tested with the same 4 scenarios (text-only, simple tables, dense tables, mixed content) using synthetically generated 60-page PDFs. The source PDFs and converted DOCX outputs are included in `outputs/` so you can inspect the visual quality yourself.

## Libraries Tested

| Library | License | Approach |
|---------|---------|----------|
| **pdf2docx** | GPL v3 / AGPL (PyMuPDF) | Monolithic — single library handles everything via PyMuPDF C engine |
| **pdfplumber + python-docx** | MIT + MIT | pdfplumber extracts text/tables via pdfminer.six, python-docx generates DOCX |
| **pypdf + python-docx** | BSD + MIT | pypdf extracts text/images (no table detection), python-docx generates DOCX |
| **camelot + PyMuPDF + python-docx** | MIT + AGPL + MIT | camelot for tables (lattice/stream), PyMuPDF for text/images, python-docx for DOCX |
| **docling** | MIT | ML-based layout analysis, HTML intermediate, htmldocx for DOCX output |

## Speed Results (60-page PDFs)

### Head-to-Head Comparison

| Scenario | pdf2docx | pdfplumber | pypdf | camelot (stream) | docling |
|----------|----------|------------|-------|-------------------|---------|
| **Text-only** | 1.53s | 1.87s | 0.20s | 2.70s | 12.02s |
| **Simple tables** | 4.52s | 2.01s | 0.20s | 3.54s | 24.32s |
| **Dense tables** | **92.03s** | **6.04s** | 0.63s | 15.21s | 11.89s |
| **Mixed content** | 7.85s | 1.61s | 0.60s | 3.55s | 37.65s |

> The 90-second Celery timeout is the hard constraint. Only pdf2docx exceeds it (dense tables at 92s).

### Verdict Summary

| Library | Text-only | Simple tables | Dense tables | Mixed content | Overall |
|---------|-----------|---------------|--------------|---------------|---------|
| pdf2docx | Safe | Safe | **EXCEEDS 90s** | Safe | Risky |
| pdfplumber | Safe | Safe | Safe (6s) | Safe | **Safe** |
| pypdf | Safe | Safe | Safe (0.6s) | Safe | **Safe** |
| camelot (stream) | Safe | Safe | Safe (15s) | Safe | **Safe** |
| camelot (lattice) | Safe (37s) | Safe (36s) | Safe (47s) | Safe (36s) | Safe but slow |
| docling | Safe (12s) | Safe (24s) | Safe (12s) | Safe (38s) | Safe but slow |

## Quality Results

Speed is only half the story. Here's how each library handles different content types:

### Table Detection

| Library | Tables preserved? | Dense table accuracy | Notes |
|---------|-------------------|---------------------|-------|
| pdf2docx | Yes | High | Best table fidelity, but O(n^2) performance on dense tables |
| pdfplumber | Yes | High | Line-intersection detection, fast even on dense tables |
| pypdf | **No** | N/A | Tables become flat text lines — no structure preserved |
| camelot (lattice) | Yes | High (100%) | Requires Ghostscript; rasterizes each page |
| camelot (stream) | Partial | Low (47%) | Text-clustering approach; captures text as tables incorrectly |
| docling | Partial | Low (16/180 detected) | ML-based; misses many tables, merges others |

### Image Extraction

| Library | Images preserved? | Notes |
|---------|-------------------|-------|
| pdf2docx | Yes | Built-in via PyMuPDF |
| pdfplumber | No (alone) | Needs pypdf/pikepdf for images; text/table only natively |
| pypdf | Yes | Extracts embedded images; appends after text (not positioned) |
| camelot | Yes (via PyMuPDF) | PyMuPDF handles image extraction in the pipeline |
| docling | **No** | HTML export drops images entirely |

### Layout Fidelity

| Library | Layout preservation | Font/style mapping |
|---------|--------------------|--------------------|
| pdf2docx | Good — spatial positioning | Partial — maps fonts and sizes |
| pdfplumber | Moderate — word-level, line reconstruction | Manual — font size to heading style |
| pypdf | None — reading-order text only | None — default DOCX styles |
| camelot | Approximate — sequential elements | Explicit font size mapping |
| docling | Moderate — ML layout analysis | Limited — HTML intermediate loses detail |

## Recommendation

### For production use (90s timeout constraint):

**pdfplumber + python-docx** is the best balanced option:
- All MIT licensed (no AGPL concerns)
- 15x faster than pdf2docx on dense tables (6s vs 92s)
- Preserves table structure with high accuracy
- Pure Python — easy deployment, no system dependencies
- Only limitation: no native image extraction (pair with pypdf for images)

### Decision Matrix

| Priority | Best choice | Why |
|----------|-------------|-----|
| Speed + no timeout risk | pypdf + python-docx | 0.2-0.6s, but no table structure |
| Speed + table fidelity | **pdfplumber + python-docx** | 1.6-6s, tables preserved, MIT licensed |
| Maximum fidelity | pdf2docx | Best layout/style preservation, but AGPL + timeout risk |
| Table accuracy reporting | camelot (stream) + python-docx | Per-table accuracy scores, but slower |

## Test Scenarios

All scenarios use **60-page PDFs** (US Letter, 612x792 pt):

| Scenario | Content | Why it matters |
|----------|---------|----------------|
| **Text-only** | 25 lines/page, heading + body text | Best case — baseline performance |
| **Simple tables** | 8 paragraphs + 1 table (5x4) + 8 paragraphs per page | Typical business document |
| **Dense tables** | 3 tables per page (8x6 each) = 180 tables, 8,640 cells | Worst case — financial reports; pdf2docx bottleneck |
| **Mixed content** | Heading + 5 paragraphs + image + table (6x5) + 6 paragraphs | Most realistic contract/business document |

## Repo Structure

```
pdf-to-docx-benchmarks/
├── README.md
├── notebooks/
│   ├── pdf2docx_benchmark.ipynb        # pdf2docx (AGPL)
│   ├── pdfplumber_benchmark.ipynb      # pdfplumber + python-docx (MIT)
│   ├── pypdf_benchmark.ipynb           # pypdf + python-docx (BSD + MIT)
│   ├── camelot_benchmark.ipynb         # camelot + PyMuPDF + python-docx
│   └── docling_benchmark.ipynb         # docling (MIT, ML-based)
└── outputs/
    ├── pdf2docx/                       # Source PDFs + converted DOCXs
    ├── pdfplumber/
    ├── pypdf/
    ├── camelot/                        # Both lattice and stream flavor outputs
    └── docling/
```

Each `outputs/<library>/` directory contains:
- `*.pdf` — the generated test PDFs (open these to see the source)
- `*.docx` — the converted DOCX files (open these to compare quality)
- `*.png` — benchmark charts (where available)

## Licensing Summary

| Component | License | Used by |
|-----------|---------|---------|
| `pdf2docx` | GPL v3 | pdf2docx benchmark |
| `PyMuPDF` | AGPL-3.0 | pdf2docx, camelot (+ test PDF generation for all) |
| `pdfplumber` | MIT | pdfplumber benchmark |
| `pdfminer.six` | MIT | Transitive dep of pdfplumber |
| `python-docx` | MIT | All benchmarks (DOCX generation) |
| `pypdf` | BSD-3 | pypdf benchmark |
| `camelot-py` | MIT | camelot benchmark |
| `docling` | MIT | docling benchmark |
| `htmldocx` | MIT | docling benchmark (HTML to DOCX) |
| `reportlab` | BSD | docling benchmark (test PDF generation) |

## Running the Benchmarks

```bash
# Install all dependencies
pip install pdf2docx pdfplumber python-docx pypdf PyMuPDF camelot-py[cv] docling htmldocx reportlab matplotlib

# Run any notebook
jupyter notebook notebooks/pdfplumber_benchmark.ipynb
```

Each notebook is self-contained: it generates test PDFs, runs conversions, measures timing, and produces comparison charts.

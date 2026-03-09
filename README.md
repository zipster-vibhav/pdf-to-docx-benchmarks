# PDF to DOCX Conversion Benchmarks

Benchmarking 6 different approaches for converting 60-page PDFs to DOCX, evaluated against a 90-second Celery task timeout constraint.

Each approach was tested with the same scenarios (text-only, simple tables, dense tables, mixed content, and two-column MSA) using synthetically generated 60-page PDFs. The source PDFs and converted DOCX outputs are included in `outputs/` so you can inspect the visual quality yourself.

## Libraries Tested

| Library | License | Approach |
|---------|---------|----------|
| **LibreOffice (soffice)** | MPL-2.0 | Native rendering engine via `soffice --headless` subprocess |
| **pdf2docx** | GPL v3 / AGPL (PyMuPDF) | Monolithic — single library handles everything via PyMuPDF C engine |
| **pdfplumber + python-docx** | MIT + MIT | pdfplumber extracts text/tables via pdfminer.six, python-docx generates DOCX |
| **pypdf + python-docx** | BSD + MIT | pypdf extracts text/images (no table detection), python-docx generates DOCX |
| **camelot + PyMuPDF + python-docx** | MIT + AGPL + MIT | camelot for tables (lattice/stream), PyMuPDF for text/images, python-docx for DOCX |
| **docling** | MIT | ML-based layout analysis, HTML intermediate, htmldocx for DOCX output |

## Speed Results (60-page PDFs)

### Head-to-Head Comparison

| Scenario | LibreOffice | pdf2docx | pdfplumber | pypdf | camelot (stream) | docling |
|----------|-------------|----------|------------|-------|-------------------|---------|
| **Text-only** | 2.50s | 1.77s | 1.87s | 0.20s | 2.70s | 11.48s |
| **Simple tables** | 2.58s | 5.11s | 2.01s | 0.20s | 3.54s | 22.22s |
| **Dense tables** | **14.45s** | **97.89s** | **6.04s** | 0.63s | 15.21s | 10.05s |
| **Mixed content** | 5.07s | 8.54s | 1.61s | 0.60s | 3.55s | 29.98s |
| **Two-column MSA** | 2.49s | 2.48s | N/A | N/A | N/A | 5.32s |

> The 90-second Celery timeout is the hard constraint. Only pdf2docx exceeds it (dense tables at 98s).

### Verdict Summary

| Library | Text-only | Simple tables | Dense tables | Mixed content | Two-column MSA | Overall |
|---------|-----------|---------------|--------------|---------------|----------------|---------|
| LibreOffice | Safe (2.5s) | Safe (2.6s) | Safe (14s) | Safe (5s) | Safe (2.5s) | **Safe** |
| pdf2docx | Safe | Safe | **EXCEEDS 90s** | Safe | Safe | Risky |
| pdfplumber | Safe | Safe | Safe (6s) | Safe | N/A | **Safe** |
| pypdf | Safe | Safe | Safe (0.6s) | Safe | N/A | **Safe** |
| camelot (stream) | Safe | Safe | Safe (15s) | Safe | N/A | **Safe** |
| camelot (lattice) | Safe (37s) | Safe (36s) | Safe (47s) | Safe (36s) | N/A | Safe but slow |
| docling | Safe (12s) | Safe (24s) | Safe (12s) | Safe (38s) | Safe (5s) | Safe but slow |

## Quality Results

Speed is only half the story. Here's how each approach handles different content types:

### Table Detection

| Library | Tables preserved? | Dense table accuracy | Notes |
|---------|-------------------|---------------------|-------|
| LibreOffice | Yes | High | Native rendering engine reconstructs tables from PDF drawing commands |
| pdf2docx | Yes | High | Best table fidelity, but O(n^2) performance on dense tables |
| pdfplumber | Yes | High | Line-intersection detection, fast even on dense tables |
| pypdf | **No** | N/A | Tables become flat text lines — no structure preserved |
| camelot (lattice) | Yes | High (100%) | Requires Ghostscript; rasterizes each page |
| camelot (stream) | Partial | Low (47%) | Text-clustering approach; captures text as tables incorrectly |
| docling | Partial | Low (16/180 detected) | ML-based; misses many tables, merges others |

### Image Extraction

| Library | Images preserved? | Notes |
|---------|-------------------|-------|
| LibreOffice | Yes | Native — preserves embedded images with positioning |
| pdf2docx | Yes | Built-in via PyMuPDF |
| pdfplumber | No (alone) | Needs pypdf/pikepdf for images; text/table only natively |
| pypdf | Yes | Extracts embedded images; appends after text (not positioned) |
| camelot | Yes (via PyMuPDF) | PyMuPDF handles image extraction in the pipeline |
| docling | **No** | HTML export drops images entirely |

### Layout Fidelity

| Library | Layout preservation | Font/style mapping | Two-column support |
|---------|--------------------|--------------------|-------------------|
| LibreOffice | Good — native rendering | Good — preserves fonts/sizes | Yes — detects column layouts |
| pdf2docx | Good — spatial positioning | Partial — maps fonts and sizes | Yes |
| pdfplumber | Moderate — word-level, line reconstruction | Manual — font size to heading style | No |
| pypdf | None — reading-order text only | None — default DOCX styles | No |
| camelot | Approximate — sequential elements | Explicit font size mapping | No |
| docling | Moderate — ML layout analysis | Limited — HTML intermediate loses detail | Partial |

## Recommendation

### For production use (90s timeout constraint):

**LibreOffice** is the highest-fidelity option if you can install it on the server:
- 2-14s across all scenarios, well within timeout
- Tables, images, fonts, two-column layouts — all handled natively
- MPL-2.0 licensed (permissive)
- Requires `soffice` binary (~200MB) and subprocess call

**pdfplumber + python-docx** is the best pure-Python option:
- All MIT licensed (no AGPL concerns)
- 15x faster than pdf2docx on dense tables (6s vs 98s)
- Preserves table structure with high accuracy
- Pure Python — easy deployment, no system dependencies
- Only limitation: no native image extraction (pair with pypdf for images)

### Decision Matrix

| Priority | Best choice | Why |
|----------|-------------|-----|
| Maximum fidelity + layout | **LibreOffice** | Native engine, handles everything, MPL-2.0 |
| Speed + table fidelity (pure Python) | **pdfplumber + python-docx** | 1.6-6s, tables preserved, MIT licensed |
| Speed + no timeout risk | pypdf + python-docx | 0.2-0.6s, but no table structure |
| Maximum fidelity (Python-only) | pdf2docx | Best layout/style preservation, but AGPL + timeout risk |

## Test Scenarios

All scenarios use **60-page PDFs** (US Letter, 612x792 pt):

| Scenario | Content | Why it matters |
|----------|---------|----------------|
| **Text-only** | 25 lines/page, heading + body text | Best case — baseline performance |
| **Simple tables** | 8 paragraphs + 1 table (5x4) + 8 paragraphs per page | Typical business document |
| **Dense tables** | 3 tables per page (8x6 each) = 180 tables, 8,640 cells | Worst case — financial reports; pdf2docx bottleneck |
| **Mixed content** | Heading + 5 paragraphs + image + table (6x5) + 6 paragraphs | Most realistic contract/business document |
| **Two-column MSA** | Two-column legal text, 10 sections repeated ~12x | Master Service Agreement layout |

## Repo Structure

```
pdf-to-docx-benchmarks/
├── README.md
├── notebooks/
│   ├── libreoffice_benchmark.ipynb     # LibreOffice soffice (MPL-2.0)
│   ├── pdf2docx_benchmark.ipynb        # pdf2docx (AGPL)
│   ├── pdfplumber_benchmark.ipynb      # pdfplumber + python-docx (MIT)
│   ├── pypdf_benchmark.ipynb           # pypdf + python-docx (BSD + MIT)
│   ├── camelot_benchmark.ipynb         # camelot + PyMuPDF + python-docx
│   └── docling_benchmark.ipynb         # docling (MIT, ML-based)
└── outputs/
    ├── libreoffice/                    # Source PDFs + converted DOCXs (incl. two-column MSA)
    ├── pdf2docx/
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
| `LibreOffice` | MPL-2.0 | LibreOffice benchmark |
| `pdf2docx` | GPL v3 | pdf2docx benchmark |
| `PyMuPDF` | AGPL-3.0 | pdf2docx, camelot (+ test PDF generation for all) |
| `pdfplumber` | MIT | pdfplumber benchmark |
| `pdfminer.six` | MIT | Transitive dep of pdfplumber |
| `python-docx` | MIT | All benchmarks (DOCX generation) |
| `pypdf` | BSD-3 | pypdf benchmark |
| `camelot-py` | MIT | camelot benchmark |
| `docling` | MIT | docling benchmark |
| `htmldocx` | MIT | docling benchmark (HTML to DOCX) |
| `reportlab` | BSD | docling, LibreOffice benchmarks (two-column PDF generation) |

## Running the Benchmarks

```bash
# Install Python dependencies
pip install pdf2docx pdfplumber python-docx pypdf PyMuPDF camelot-py[cv] docling htmldocx reportlab matplotlib

# LibreOffice (macOS)
brew install --cask libreoffice

# Run any notebook
jupyter notebook notebooks/libreoffice_benchmark.ipynb
```

Each notebook is self-contained: it generates test PDFs, runs conversions, measures timing, and produces comparison charts.

**LibreOffice note:** The key flag is `--infilter=writer_pdf_import` — without it, LibreOffice opens PDFs as Draw documents and cannot export to DOCX.

# PDF to DOCX Conversion Benchmarks

Benchmarking 7 PDF-to-DOCX conversion approaches for 60-page documents against a 90-second Celery task timeout constraint. Includes two-column layout preservation testing for MSA/legal contracts.

## Libraries Tested

| Library | License | Approach |
|---------|---------|----------|
| **PyMuPDF (fitz)** + python-docx | AGPL-3.0 | Direct text block extraction with coordinates |
| **pdfplumber** + python-docx | MIT | Word-level extraction via pdfminer.six |
| **Camelot** + python-docx | MIT | Table-focused extraction (lattice + stream flavors) |
| **LibreOffice** (soffice) | MPL-2.0 | Native rendering engine via subprocess |
| **pdf2docx** | GPL-3.0 | PyMuPDF + layout reconstruction (monolithic) |
| **Docling** | MIT | ML-based layout detection |
| **Tesseract OCR** | Apache 2.0 | Rasterize → OCR → DOCX (for scanned PDFs) |

## Speed Results

| Scenario | PyMuPDF | pdfplumber | Camelot | LibreOffice | pdf2docx | Docling | Tesseract |
|----------|---------|------------|---------|-------------|----------|---------|-----------|
| **Text-only** | 0.19s | 1.84s | 2.92s | 2.77s | 1.72s | 11.3s | 103.8s |
| **Simple tables** | 0.19s | 2.08s | 3.43s | 2.63s | 5.09s | 22.7s | 28.0s |
| **Dense tables** | 0.64s | 6.77s | 14.07s | 13.92s | **101.4s** | 10.4s | 50.3s |
| **Mixed content** | 0.36s | 1.86s | 3.09s | 5.06s | 8.52s | 28.3s | 32.3s |
| **Two-column MSA** | 0.11s | 0.96s | 2.10s | 2.59s | 2.39s | 4.3s | 26.0s |

**Bold** = exceeds 90s timeout. Only pdf2docx (dense tables) and Tesseract (text-only) fail.

## Two-Column Layout Preservation

Two-column MSA contracts are a priority use case. Each library was enhanced with a "v2" conversion that detects columns via spatial coordinates and produces DOCX files with `w:cols` two-column section formatting.

| Library | v2 Time | Seq Match | Word Recall | Approach |
|---------|---------|-----------|-------------|----------|
| PyMuPDF | 0.09s | — | — | `page.get_text("blocks")` coordinates |
| Camelot | 0.42s | 71.4% | 100% | pdfminer.six `LTTextBox` bounding boxes |
| pdfplumber | 1.01s | 42.0% | 100% | `page.extract_words()` bounding boxes |
| Docling | 4.00s | 52.5% | 100% | ML layout + `prov[0].bbox` provenance |
| Tesseract | 26.7s | 100% | 99.5% | `image_to_data()` OCR bounding boxes |
| LibreOffice | 2.59s | 18.9% | 100% | Native column detection (no v2 needed) |

## Quality Comparison

| Library | Tables | Images | Layout | Two-Column | Best For |
|---------|--------|--------|--------|------------|----------|
| pdf2docx | Excellent | Good | Good | Native | General-purpose with tables |
| PyMuPDF | None (flat text) | Extractable | Coordinates | v2 | Speed-critical text extraction |
| pdfplumber | Structured | No | Word-level | v2 | Text + table data extraction |
| Tesseract | None | None | OCR boxes | v2 (100%) | Scanned PDFs only |
| Docling | Partial | None | ML-detected | v2 | Research/experimental |
| Camelot | Excellent (lattice) | No | pdfminer | v2 | Table-heavy documents |
| LibreOffice | Native | Native | Native | Native | Highest fidelity |

## Tesseract OCR Accuracy

Tesseract is the only OCR-based tool — it re-reads text from rasterized images. Text similarity metrics measure accuracy against ground truth.

| Scenario | Seq Match | Word Recall | Char Ratio | Time |
|----------|-----------|-------------|------------|------|
| Text-only | 62.8% | 100% | 1.01 | 103.8s |
| Simple tables | 74.5% | 96.0% | 1.00 | 28.0s |
| Dense tables | 98.2% | 100% | 1.00 | 50.3s |
| Mixed content | 91.4% | 95.9% | 1.00 | 32.3s |
| Two-column MSA | 100% | 99.5% | 1.00 | 26.0s |

## Recommendation

| Use Case | Recommended Library | Why |
|----------|-------------------|-----|
| Highest fidelity | **LibreOffice** | Native tables, images, fonts, columns. 2-14s. |
| Speed-critical | **PyMuPDF + python-docx** | 0.1-0.6s, 10-100x faster than alternatives |
| Balanced speed + tables | **pdfplumber + python-docx** | MIT licensed, tables preserved, 2-7s |
| Scanned PDFs | **Tesseract OCR** | Only option for PDFs without text layer |
| Avoid for dense tables | pdf2docx | Exceeds 90s timeout on table-heavy docs |

## Repo Structure

```
pdf-to-docx-benchmarks/
├── README.md
├── notebooks/
│   ├── final_comparison.ipynb       # Cross-library charts and summary
│   ├── pypdf_benchmark.ipynb        # PyMuPDF + python-docx
│   ├── pdfplumber_benchmark.ipynb   # pdfplumber + python-docx
│   ├── camelot_benchmark.ipynb      # Camelot (lattice + stream)
│   ├── libreoffice_benchmark.ipynb  # LibreOffice soffice
│   ├── pdf2docx_benchmark.ipynb     # pdf2docx
│   ├── docling_benchmark.ipynb      # Docling (ML-based)
│   └── tesseract_benchmark.ipynb    # Tesseract OCR
└── outputs/
    ├── pypdf/          # PDFs + DOCXs + charts
    ├── pdfplumber/
    ├── camelot/        # Lattice + stream + v2 outputs
    ├── libreoffice/
    ├── pdf2docx/
    ├── docling/
    ├── tesseract/
    └── *.png           # Cross-library comparison charts
```

Each notebook is self-contained: generates test PDFs, runs conversions, measures timing, and produces comparison charts.

## Test Scenarios

| Scenario | Content | Pages |
|----------|---------|-------|
| **Text-only** | 20-25 lines/page, heading + body text | 60 |
| **Simple tables** | Text paragraphs + 1 table (5x4) per page | 60 |
| **Dense tables** | 3 tables/page (8x6 each) = 180 tables, 8640 cells | 60 |
| **Mixed content** | Text + embedded image + table + closing text | 60 |
| **Two-column MSA** | Legal contract in two-column layout, 10 sections | ~18 |

## Running

```bash
pip install pdf2docx pdfplumber python-docx pypdf PyMuPDF camelot-py[cv] \
            docling htmldocx reportlab matplotlib pytesseract pdf2image

# Tesseract + poppler (macOS)
brew install tesseract poppler

# LibreOffice (macOS)
brew install --cask libreoffice

# Run any notebook
jupyter notebook notebooks/final_comparison.ipynb
```

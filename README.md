# PDF to DOCX Conversion Benchmarks

7 libraries benchmarked on 60-page PDFs against a 90-second Celery task timeout. Test scenarios: text-only, simple tables (1 per page, 5x4), dense tables (3 per page, 8x6, 180 total), mixed content (text + image + table), and two-column MSA contract (~18 pages).

## Results

| Library | License | Text-only | Simple Tables | Dense Tables | Mixed Content | Two-Col MSA |
|---------|---------|-----------|---------------|--------------|---------------|-------------|
| **PyMuPDF** + python-docx | AGPL-3.0 | 0.19s | 0.19s | 0.64s | 0.36s | 0.11s |
| **pdfplumber** + python-docx | MIT | 1.84s | 2.08s | 6.77s | 1.86s | 0.96s |
| **Camelot** + python-docx | MIT | 2.92s | 3.43s | 14.07s | 3.09s | 2.10s |
| **LibreOffice** (soffice) | MPL-2.0 | 2.77s | 2.63s | 13.92s | 5.06s | 2.59s |
| **pdf2docx** | GPL-3.0 | 1.72s | 5.09s | **101.4s** | 8.52s | 2.39s |
| **Docling** | MIT | 11.3s | 22.7s | 10.4s | 28.3s | 4.3s |
| **Tesseract OCR** | Apache-2.0 | **103.8s** | 28.0s | 50.3s | 32.3s | 26.0s |

**Bold** = exceeds 90s timeout.

## Quality

| Library | Tables | Images | Layout | Two-Column | Notes |
|---------|--------|--------|--------|------------|-------|
| PyMuPDF | None | Extractable | Coordinates | v2 | Fastest, text-only extraction |
| pdfplumber | Structured | No | Word-level | v2 | MIT, good table detection |
| Camelot | Excellent | No | pdfminer | v2 | Best for table-heavy docs |
| LibreOffice | Native | Native | Native | Native | Highest fidelity, needs soffice binary |
| pdf2docx | Excellent | Good | Good | Native | Fails on dense tables (O(n^2) table detection) |
| Docling | Partial | No | ML-detected | v2 | Slow, ML-based layout analysis |
| Tesseract | None | None | OCR boxes | v2 | Only option for scanned PDFs |

Tesseract OCR accuracy (re-reads text from rasterized images):

| Scenario | Seq Match | Word Recall |
|----------|-----------|-------------|
| Text-only | 62.8% | 100% |
| Simple tables | 74.5% | 96.0% |
| Dense tables | 98.2% | 100% |
| Mixed content | 91.4% | 95.9% |
| Two-column MSA | 100% | 99.5% |

All other libraries extract native PDF text (100% word recall by definition).

## Recommendation

| Use Case | Library | Why |
|----------|---------|-----|
| Highest fidelity | LibreOffice | Native tables, images, fonts, columns. 2-14s. |
| Speed-critical | PyMuPDF | 0.1-0.6s, 10-100x faster than alternatives |
| Balanced (speed + tables + MIT) | pdfplumber | 1-7s, tables preserved, MIT licensed |
| Scanned PDFs | Tesseract | Only option for PDFs without a text layer |
| Avoid for dense tables | pdf2docx | Exceeds 90s timeout on table-heavy docs |

## Repo Structure

Each library has a `benchmark.ipynb` and `outputs/` directory with generated PDFs and converted DOCXs.

```
pymupdf/    pdfplumber/    camelot/    libreoffice/
pdf2docx/   docling/       tesseract/  final_comparison.ipynb
```

## Running

```bash
pip install pdf2docx pdfplumber python-docx pypdf PyMuPDF camelot-py[cv] \
            docling htmldocx reportlab matplotlib pytesseract pdf2image
brew install tesseract poppler libreoffice  # macOS
```

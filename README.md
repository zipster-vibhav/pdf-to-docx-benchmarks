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

| Library | Word Recall | Tables | Images | Two-Column | Notes |
|---------|-------------|--------|--------|------------|-------|
| PyMuPDF | 100% | No detection, flat text | Extractable | v2 (coordinate-based) | Fastest, text-only extraction |
| pdfplumber | 100% | Detected via line intersections | No | v2 (word bounding boxes) | MIT, good table detection |
| Camelot | 100% | Lattice + stream detection | No | v2 (pdfminer bounding boxes) | Best for table-heavy docs |
| LibreOffice | 100% | Preserved as drawing objects (0 in python-docx API) | Preserved | Detected and converted | Highest fidelity, needs soffice binary (~200MB) |
| pdf2docx | 100% | Reconstructed as Word tables | Repositioned | Detected and converted | O(n^2) table detection, fails on dense tables |
| Docling | 100% | Partial (ML-based) | No | v2 (ML layout provenance) | Slow, research/experimental |
| Tesseract | 96-100% | No detection, flat text | No (rasterized away) | v2 (OCR bounding boxes) | Only option for scanned PDFs |

Word recall = % of ground truth words found in converted DOCX. All libraries except Tesseract extract native PDF text (100%). Tesseract re-reads from rasterized images: 100% (text-only, dense tables), 99.5% (two-col MSA), 96% (simple tables), 95.9% (mixed).

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

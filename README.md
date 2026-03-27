# PDF to DOCX Conversion Benchmarks

8 libraries benchmarked on 60-page PDFs against a 90-second Celery task timeout. Test scenarios: text-only, simple tables (1 per page, 5x4), dense tables (3 per page, 8x6, 180 total), mixed content (text + image + table), and two-column MSA contract (~18 pages).

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
| **Adobe PDF Services** | Commercial | 5.4s* | 6.9s* | — | 6.7s* | 5.4s* |

**Bold** = exceeds 90s timeout. *Adobe tested on real contract PDFs (not synthetic), see `adobe/` for details.

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
| Adobe PDF Services | 94-98% | Preserved as Word tables | Preserved | Preserved natively | Commercial API, OCR support, 5-7s per doc |

Word recall = % of ground truth words found in converted DOCX. All libraries except Tesseract extract native PDF text (100%). Tesseract re-reads from rasterized images: 100% (text-only, dense tables), 99.5% (two-col MSA), 96% (simple tables), 95.9% (mixed).

## Structural Recall

How well each converter preserves document structure (tables, headings, columns, image placement, reading order) — not just text content. Scored by manual inspection across all test scenarios.

| Library | Structural Recall | Notes |
|---------|:-----------------:|-------|
| PyMuPDF | **22%** | Flat text dump — no tables, no columns, no layout awareness |
| pdfplumber | **52%** | Tables detected via line intersections, but no images or column handling |
| Camelot | **58%** | Best table detection (lattice + stream), but no images or column layout |
| LibreOffice | **96%** ⚠️ | Tables, images, fonts, columns all preserved — but every word is its own text box. Document *looks* correct but is uneditable (can't select paragraphs, reflow text, or make normal edits) |
| pdf2docx | **55%** | Reconstructs Word tables + images, detects columns, but O(n²) breaks on dense tables |
| Docling | **28%** | ML-based layout is experimental — partial tables, no images, inconsistent headings |
| Tesseract | **20%** | OCR bounding boxes only — no table, column, or image structure preserved |
| Adobe PDF Services | **95%** | Native Word tables, images preserved, columns handled. Clean editable output with real paragraphs (0 VML boxes on simple docs) |

> **Why does LibreOffice score 96% but get a warning?** Structural recall measures whether layout elements *exist* in the output — and soffice gets nearly everything right. But the output wraps every word in a VML/WPS drawing object, making the DOCX a collection of positioned text boxes rather than flowing text. If your use case is read-only viewing or PDF archival, soffice is the best. Adobe matches at 95% structural recall *and* produces clean, editable Word files — making it the clear winner for any use case that requires an actual usable document.

## Recommendation

| Use Case | Library | Why |
|----------|---------|-----|
| **Best overall** | **Adobe PDF Services** | **95% structural recall with clean, editable output. 5-7s, native tables/images/columns, OCR support. The only converter that matches LibreOffice fidelity without the text-box problem.** |
| Highest fidelity (read-only) | LibreOffice | 96% structural recall, but output is uneditable — every word is a text box. Fine for viewing, not for editing. |
| Speed-critical | PyMuPDF | 0.1-0.6s, 10-100x faster than alternatives. No structure preservation. |
| Balanced (speed + tables + MIT) | pdfplumber | 1-7s, tables preserved, MIT licensed |
| Scanned PDFs | Tesseract | Only option for PDFs without a text layer |
| Avoid for dense tables | pdf2docx | Exceeds 90s timeout on table-heavy docs |

## Repo Structure

Each library has a `benchmark.ipynb` and `outputs/` directory with generated PDFs and converted DOCXs.

```
pymupdf/    pdfplumber/    camelot/    libreoffice/
pdf2docx/   docling/       tesseract/  adobe/
final_comparison.ipynb
```

## Running

```bash
pip install pdf2docx pdfplumber python-docx pypdf PyMuPDF camelot-py[cv] \
            docling htmldocx reportlab matplotlib pytesseract pdf2image \
            pdfservices-sdk  # Adobe PDF Services
brew install tesseract poppler libreoffice  # macOS
```

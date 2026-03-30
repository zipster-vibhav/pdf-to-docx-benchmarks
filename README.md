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
| **Adobe PDF Services** | Commercial | 5.4s | 6.9s | — | 6.7s | 5.4s |

**Bold** = exceeds 90s timeout.

## Quality — Unified Benchmark

All 8 libraries tested on the **same 5 synthetic PDFs** with the **same ground truth** text. See `unified_benchmark.ipynb` for the full methodology.

We measure two word recall metrics:

- **Word Recall (relaxed):** Strip all punctuation, compare pure alphanumeric words. Answers: *did the converter capture the actual content?*
- **Word Recall (strict):** Keep punctuation as-is, exact token matching. Answers: *did the converter preserve formatting, punctuation, and structure?*

### Word Recall (relaxed)

| Library | Plain Text | Tables | Images | Two-Col MSA | Mixed | AVG |
|---------|-----------|--------|--------|-------------|-------|-----|
| PyMuPDF | 100% | 100% | 100% | 100% | 100% | **100%** |
| pdfplumber | 100% | 100% | 100% | 100% | 100% | **100%** |
| Camelot | 100% | 100% | 100% | 100% | 100% | **100%** |
| LibreOffice | 100% | 100% | 100% | 100% | 100% | **100%** |
| pdf2docx | 100% | 100% | 100% | 100% | 100% | **100%** |
| Docling | 100% | 100% | 100% | 100% | 100% | **100%** |
| Adobe PDF Services | 100% | 100% | 100% | 98.9% | 100% | **99.8%** |
| Tesseract | 97.4% | 100% | 97.1% | 100% | 100% | **98.9%** |

### Word Recall (strict)

| Library | Plain Text | Tables | Images | Two-Col MSA | Mixed | AVG |
|---------|-----------|--------|--------|-------------|-------|-----|
| PyMuPDF | 100% | 100% | 100% | 100% | 100% | **100%** |
| pdfplumber | 100% | 100% | 100% | 100% | 100% | **100%** |
| Camelot | 100% | 100% | 100% | 100% | 100% | **100%** |
| LibreOffice | 100% | 100% | 100% | 100% | 100% | **100%** |
| pdf2docx | 100% | 100% | 100% | 100% | 100% | **100%** |
| Docling | 100% | 100% | 100% | 100% | 100% | **100%** |
| Tesseract | 98.0% | 100% | 97.4% | 99.5% | 100% | **99.0%** |
| Adobe PDF Services | 100% | 93.3% | 100% | 97.5% | 100% | **98.2%** |

> **Why does Adobe's strict recall drop on tables/two-col?** Adobe collapses repeating per-page headings into DOCX headers using `{PAGE}` field codes — visually correct, but some punctuated tokens (e.g., `"Section 7:"`) are lost as distinct words when the header stores only the last page's value. The relaxed metric confirms all actual words are captured.
>
> **Why does Tesseract lose words?** It OCRs from rasterized images rather than extracting native text, introducing character recognition errors.

### Methodology

The unified benchmark (`unified_benchmark.ipynb`) generates 5 synthetic PDFs with known ground truth text, then runs all 8 libraries against the same inputs:

| PDF | Pages | Features |
|-----|-------|----------|
| `plain_text` | 10 | Headings + 20 body paragraphs per page |
| `tables_and_text` | 10 | Text + bordered 5x4 tables |
| `logo_and_images` | 10 | Text + embedded PNG logo + photo images |
| `two_column_msa` | 5 | Two-column legal MSA layout (ReportLab) |
| `mixed_everything` | 10 | Text + logo image + 4x3 table per page |

Text extraction uses XML-level `<w:t>` parsing across all DOCX files (document.xml + headers + footers), with `{PAGE}` field code expansion for Adobe's dynamic headers.

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

Each library has a `benchmark.ipynb` and `outputs/` directory with generated PDFs and converted DOCXs. The unified benchmark tests all libraries on the same synthetic PDFs.

```
pymupdf/    pdfplumber/    camelot/    libreoffice/
pdf2docx/   docling/       tesseract/  adobe/
unified_benchmark.ipynb     # All 8 libraries, same PDFs, same ground truth
final_comparison.ipynb
```

## Running

```bash
pip install pdf2docx pdfplumber python-docx pypdf PyMuPDF camelot-py[cv] \
            docling htmldocx reportlab matplotlib pytesseract pdf2image \
            pdfservices-sdk  # Adobe PDF Services
brew install tesseract poppler libreoffice  # macOS
```

# 📄 PDF Extraction Process Explanation

This document explains the PDF extraction script used in the EPR Research Chatbot project. The script converts a folder of PDF documents into structured Markdown files with page labels, suitable for downstream chunking and semantic embedding.

---

## 🧩 Overview

The script processes all PDF files in a directory, determines whether each PDF is scanned or digital, extracts its content accordingly, adds page markers, and saves the output as `.md` files. It also logs failures and scanned files.

---

## 📁 Input and Output Paths

```python
PDF_DIR = "/gpfs/gibbs/project/yse/shared/data/epr_rag/PDFS"
OUTPUT_DIR = "/gpfs/gibbs/project/yse/shared/yl2739/output_md"
FAILED_DIR = "/gpfs/gibbs/project/yse/shared/yl2739/failed_pdfs"
REPORT_FILE = "/gpfs/gibbs/project/yse/shared/yl2739/processing_report.csv"
FAILED_REPORT_FILE = "/gpfs/gibbs/project/yse/shared/yl2739/failed_report.csv"
SCANNED_CSV = "/gpfs/gibbs/project/yse/shared/yl2739/scanned_pdfs.csv"
```

* All outputs are saved in their respective folders.
* Reports are generated as CSV files.

---

## 🏷️ Key Thresholds

```python
OCR_CHAR_THRESHOLD = 50
SCANNED_PAGE_RATIO_THRESHOLD = 0.8
```

* A page with fewer than 50 characters and images is considered scanned.
* If 80%+ of pages meet this criterion, the entire PDF is treated as scanned.

---

## 🧠 Class: `PDFProcessor`

This class wraps the logic for:

* Identifying scanned PDFs
* Extracting text via OCR or `to_markdown`
* Handling and logging failures
* Saving success/failure reports

---

### 🔍 `is_scanned_pdf()`

Checks each page for:

* Minimal text length
* Presence of embedded images

Returns `True` if scanned page ratio exceeds the threshold.

---

### 📥 `process_pdf()`

If scanned:

* Use `pytesseract` to perform OCR per page (via rendered image)
* Output formatted as Markdown with `## Page: X`

If not scanned:

* Use `pymupdf4llm.to_markdown()` to get structured text
* Estimate page markers based on line count and insert them evenly

Fails if result is too short (< 100 characters).

---

### 🧹 `move_failed_files()` and `save_*_report()`

* Failed PDFs are moved to `FAILED_DIR`
* Logs reasons in `failed_report.csv`
* Generates full `processing_report.csv`
* Lists all scanned PDFs in `scanned_pdfs.csv`

---

## ▶️ Main Loop

Processes all `.pdf` files in the input folder:

```python
for pdf_file in tqdm(pdf_files):
    content = processor.process_pdf(pdf_file)
    if content:
        # Save Markdown
```

Tracks successes and failures, prints summary:

```python
print(f"- 成功处理: {success_count}")
print(f"- 失败文件: {len(processor.failed_files)}")
```

---

This extraction script ensures that each PDF:

* Is properly detected as scanned or not
* Has page markers inserted
* Is exported to a clean Markdown format
* Is logged for processing status

It creates reliable inputs for chunking and embedding in the chatbot pipeline.

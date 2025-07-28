# 📄 PDF Extraction Script:

This document explains the full content of the `Extraction.ipynb` script, including inline English annotations. It is used in the EPR research semantic retrieval system to convert PDF documents into structured Markdown files.

---

## 📦 Import Dependencies

```python
import os
import shutil
import pandas as pd
import traceback
from pathlib import Path
from tqdm import tqdm
import fitz  # PyMuPDF
from pymupdf4llm import to_markdown
import pytesseract
from PIL import Image
import io
```

🔹 **Explanation**: These are the necessary libraries for PDF parsing, OCR, file path handling, logging, and image processing.

---

## 📁 Path Configuration

```python
PDF_DIR = "/gpfs/gibbs/project/yse/shared/data/epr_rag/PDFS"
OUTPUT_DIR = "/gpfs/gibbs/project/yse/shared/yl2739/output_md"
FAILED_DIR = "/gpfs/gibbs/project/yse/shared/yl2739/failed_pdfs"
REPORT_FILE = "/gpfs/gibbs/project/yse/shared/yl2739/processing_report.csv"
FAILED_REPORT_FILE = "/gpfs/gibbs/project/yse/shared/yl2739/failed_report.csv"
SCANNED_CSV = "/gpfs/gibbs/project/yse/shared/yl2739/scanned_pdfs.csv"
```

🔹 **Explanation**: Defines paths for input PDFs, Markdown outputs, failed PDFs, reports, and scanned file records.

---

## 🔧 Create Output Directories

```python
os.makedirs(OUTPUT_DIR, exist_ok=True)
os.makedirs(FAILED_DIR, exist_ok=True)
```

🔹 **Explanation**: Ensures the required output folders exist.

---


## 🧠 Class Definition: PDFProcessor

```python
class PDFProcessor:
    def __init__(self):
        self.failed_files = []
        self.report_data = []
        self.scanned_files = []
```

🔹 **Explanation**: Initializes tracking lists for failures, reports, and scanned files.

---

## 🕵️ Method: is\_scanned\_pdf()

```python
def is_scanned_pdf(self, pdf_path):
    try:
        doc = fitz.open(pdf_path)
        scanned_pages = 0
        total_pages = doc.page_count
        for page in doc:
            text = page.get_text("text").strip()
            text_len = len(text)
            img_list = page.get_images(full=True)
            has_images = len(img_list) > 0
            if text_len < OCR_CHAR_THRESHOLD and has_images:
                scanned_pages += 1
        ratio = scanned_pages / total_pages
        return ratio >= SCANNED_PAGE_RATIO_THRESHOLD
    except Exception:
        return False
```

🔹 **Explanation**: Checks whether most pages are "image + little text" and marks the file as scanned accordingly.

---

## 📝 Method: process\_pdf()

```python
def process_pdf(self, pdf_path):
    try:
        doc = fitz.open(pdf_path)

        if self.is_scanned_pdf(pdf_path):
            self.scanned_files.append(pdf_path.name)
            full_text = []
            for i, page in enumerate(doc, start=1):
                pix = page.get_pixmap()
                img = Image.open(io.BytesIO(pix.tobytes()))
                ocr_text = pytesseract.image_to_string(img, lang='eng')
                full_text.append(f"## Page: {i}\n\n{ocr_text.strip()}")
            final_text = "\n\n".join(full_text)
            if not final_text.strip() or len(final_text) < 100:
                raise ValueError("OCR extracted text too short or empty")
            return final_text
        else:
            markdown = to_markdown(str(pdf_path))
            if not markdown.strip() or len(markdown) < 100:
                raise ValueError("Markdown content too short or empty")

            num_pages = doc.page_count
            lines = markdown.splitlines()
            avg_lines_per_page = max(1, len(lines) // num_pages)
            output_lines = []
            for i in range(num_pages):
                output_lines.append(f"## Page: {i + 1}\n")
                output_lines.extend(lines[i * avg_lines_per_page : (i + 1) * avg_lines_per_page])
            return "\n".join(output_lines)

    except Exception as e:
        error_msg = traceback.format_exc()
        self.failed_files.append((pdf_path, error_msg))
        self.report_data.append({
            "filename": pdf_path.name,
            "status": "Failed",
            "reason": error_msg,
            "action": "Not moved"
        })
        return None
```

🔹 **Explanation**:

* For scanned files, use OCR per page and add page markers
* For digital PDFs, use `to_markdown` and simulate page breaks based on line distribution
* On failure, log the error and mark the file as failed

---

## 📦 Methods: move\_failed\_files() and Reporting

```python
def move_failed_files(self):
    for pdf_path, _ in self.failed_files:
        try:
            dest = os.path.join(FAILED_DIR, pdf_path.name)
            shutil.move(str(pdf_path), dest)
            self._update_report(pdf_path.name, "Moved to failed folder")
        except Exception as e:
            self._update_report(pdf_path.name, f"Move failed: {str(e)}")
```

🔹 **Explanation**: Moves failed files to the designated folder.

```python
def _update_report(self, filename, action):
    for item in self.report_data:
        if item["filename"] == filename:
            item["action"] = action
            break
```

🔹 **Explanation**: Updates the action taken on a failed file.

```python
def save_report(self):
    df = pd.DataFrame(self.report_data)
    df.sort_values(by="status", ascending=False, inplace=True)
    df.to_csv(REPORT_FILE, index=False, encoding='utf-8-sig')
```

🔹 **Explanation**: Saves the full report of all processed files.

```python
def save_failed_report(self):
    if not self.failed_files:
        return
    df_fail = pd.DataFrame([
        {"filename": f[0].name, "error_msg": f[1]} for f in self.failed_files
    ])
    df_fail.to_csv(FAILED_REPORT_FILE, index=False, encoding='utf-8-sig')
```

🔹 **Explanation**: Saves error messages for all failed files.

```python
def save_scanned_list(self):
    if not self.scanned_files:
        return
    df_scan = pd.DataFrame({"filename": self.scanned_files})
    df_scan.to_csv(SCANNED_CSV, index=False, encoding='utf-8-sig')
```

🔹 **Explanation**: Records filenames of all scanned PDFs.

---

## ▶️ Main Script Entry Point

```python
if __name__ == "__main__":
    processor = PDFProcessor()
    pdf_files = list(Path(PDF_DIR).glob("*.pdf"))

    print("=" * 50)
    print(f"Processing {len(pdf_files)} PDF files")
    print("=" * 50)

    for pdf_file in tqdm(pdf_files, desc="Processing PDFs"):
        content = processor.process_pdf(pdf_file)
        if content:
            output_path = os.path.join(OUTPUT_DIR, f"{pdf_file.stem}.md")
            with open(output_path, "w", encoding="utf-8") as f:
                f.write(content)
            processor.report_data.append({
                "filename": pdf_file.name,
                "status": "Success",
                "reason": "",
                "action": ""
            })

    if processor.failed_files:
        print(f"\nDetected {len(processor.failed_files)} failed files")
        processor.move_failed_files()
        processor.save_failed_report()

    processor.save_scanned_list()
    processor.save_report()

    success_count = len([x for x in processor.report_data if x['status'] == 'Success'])
    print("\n" + "=" * 50)
    print("Processing Summary:")
    print(f"- Successfully processed: {success_count}")
    print(f"- Failed files: {len(processor.failed_files)}")
    print(f"- Scanned PDFs detected: {len(processor.scanned_files)}")
    print(f"\nOutput locations:")
    print(f"- Markdown output: {os.path.abspath(OUTPUT_DIR)}")
    print(f"- Failed PDFs: {os.path.abspath(FAILED_DIR)}")
    print(f"- Failure report: {os.path.abspath(FAILED_REPORT_FILE)}")
    print(f"- Scanned files list: {os.path.abspath(SCANNED_CSV)}")
    print(f"- Processing report: {os.path.abspath(REPORT_FILE)}")
    print("=" * 50)
```

🔹 **Explanation**:

* Iterates through all PDFs, processes and writes Markdown
* Saves success/failure statistics and logs

---

## ✅ Summary

This script distinguishes between scanned and digital PDFs and applies the most appropriate extraction method (OCR or text-based). It outputs clean, structured Markdown files and logs every file’s processing outcome. This serves as the foundation for downstream chunking and embedding in the chatbot pipeline.


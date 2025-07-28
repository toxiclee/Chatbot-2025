# 📄 PDF Extraction Script

This document provides a section-by-section explanation of the `Extraction.ipynb` script used to extract and convert PDF documents into structured Markdown with page markers, as part of a preprocessing pipeline for a semantic search chatbot.

---

## 📦 1. Import Libraries

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

🔹 These libraries support PDF parsing (`fitz`), Markdown conversion, OCR (`pytesseract`), image handling (`PIL`), and logging.

---

## 📁 2. Path Configuration

```python
PDF_DIR = "/gpfs/gibbs/project/yse/shared/data/epr_rag/PDFS"
OUTPUT_DIR = "/gpfs/gibbs/project/yse/shared/yl2739/output_md"
FAILED_DIR = "/gpfs/gibbs/project/yse/shared/yl2739/failed_pdfs"
REPORT_FILE = "/gpfs/gibbs/project/yse/shared/yl2739/processing_report.csv"
FAILED_REPORT_FILE = "/gpfs/gibbs/project/yse/shared/yl2739/failed_report.csv"
SCANNED_CSV = "/gpfs/gibbs/project/yse/shared/yl2739/scanned_pdfs.csv"
```

🔹 These are the input/output folders and logs for processed, failed, and scanned files.

---

## ⚙️ 3. Create Output Folders

```python
os.makedirs(OUTPUT_DIR, exist_ok=True)
os.makedirs(FAILED_DIR, exist_ok=True)
```

🔹 Ensures output and failed folders exist before processing.

---

## 🔍 4. Heuristic for Scanned PDFs

```python
OCR_CHAR_THRESHOLD = 50
SCANNED_PAGE_RATIO_THRESHOLD = 0.8
```

🔹 These thresholds help decide if a PDF is scanned based on whether most pages have little text and embedded images.

---

## 🧠 5. PDFProcessor Class

```python
class PDFProcessor:
    def __init__(self):
        self.failed_files = []
        self.report_data = []
        self.scanned_files = []
```

🔹 Initializes lists to track failed files, status logs, and scanned file names.

---

## 🕵️ 6. Detect Scanned PDFs

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

🔹 Opens the PDF and analyzes each page to see if it’s image-heavy and text-light. If most pages meet this pattern, the document is considered scanned.

---

## 📝 7. Process a Single PDF

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

🔹 Uses OCR if scanned, otherwise uses structured Markdown extraction. Logs and returns extracted content or error.

---

## 📦 8. Move Failed Files

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

🔹 Moves failed PDFs into a designated folder.

---

## 🧾 9. Reporting Helpers

```python
def _update_report(self, filename, action):
    for item in self.report_data:
        if item["filename"] == filename:
            item["action"] = action
            break

def save_report(self):
    df = pd.DataFrame(self.report_data)
    df.sort_values(by="status", ascending=False, inplace=True)
    df.to_csv(REPORT_FILE, index=False, encoding='utf-8-sig')

def save_failed_report(self):
    if not self.failed_files:
        return
    df_fail = pd.DataFrame([
        {"filename": f[0].name, "error_msg": f[1]} for f in self.failed_files
    ])
    df_fail.to_csv(FAILED_REPORT_FILE, index=False, encoding='utf-8-sig')

def save_scanned_list(self):
    if not self.scanned_files:
        return
    df_scan = pd.DataFrame({"filename": self.scanned_files})
    df_scan.to_csv(SCANNED_CSV, index=False, encoding='utf-8-sig')
```

🔹 Generates CSV logs for all processed files, failed files (with errors), and scanned PDFs.

---

## ▶️ 10. Main Execution Block

```python
if __name__ == "__main__":
    processor = PDFProcessor()
    pdf_files = list(Path(PDF_DIR).glob("*.pdf"))

    print("=" * 50)
    print(f"开始处理 {len(pdf_files)} 个PDF文件")
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
        print(f"\n发现 {len(processor.failed_files)} 个失败文件")
        processor.move_failed_files()
        processor.save_failed_report()

    processor.save_scanned_list()
    processor.save_report()

    success_count = len([x for x in processor.report_data if x['status'] == 'Success'])
    print("\n" + "=" * 50)
    print("处理结果摘要:")
    print(f"- 成功处理: {success_count}")
    print(f"- 失败文件: {len(processor.failed_files)}")
    print(f"- 识别为扫描件文件数: {len(processor.scanned_files)}")
    print(f"\n输出位置:")
    print(f"- 成功提取Markdown文件夹: {os.path.abspath(OUTPUT_DIR)}")
    print(f"- 失败的PDF文件夹: {os.path.abspath(FAILED_DIR)}")
    print(f"- 失败详细报告: {os.path.abspath(FAILED_REPORT_FILE)}")
    print(f"- 扫描件文件列表: {os.path.abspath(SCANNED_CSV)}")
    print(f"- 总处理报告: {os.path.abspath(REPORT_FILE)}")
    print("=" * 50)
```

🔹 This block coordinates the entire processing loop and prints summary stats to the console.

---

## ✅ Summary

This script performs automated and structured PDF extraction, distinguishing scanned and digital files, and logging the process. It produces clean, page-annotated Markdown suitable for semantic chunking and embedding downstream.

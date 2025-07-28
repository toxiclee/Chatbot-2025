# ✂️ Markdown Chunking Script

This document explains the chunking script used to convert extracted Markdown files into structured CSV chunks, annotated with metadata such as page number, character count, and token count. The output is used for downstream embedding and retrieval.

---

## 📁 1. Input and Output Setup

```python
input_md_folder = "output_md"
output_md_folder = "Gradient_Chunks_CSV_withpages"
```

🔹 Reads all `.md` files from the input folder and writes one `.csv` per file in the output folder. A summary file `chunking_summary.csv` is also generated.

---

## 📦 2. Chunking Parameters

```python
TARGET_CHAR = 500
MIN_CHAR = 200
MAX_CHAR = 800
```

🔹 Each chunk aims for \~500 characters. Chunks smaller than `MIN_CHAR` are discarded, and those longer than `MAX_CHAR` are split.

---

## 🧐 3. Tokenizer Setup

```python
enc = tiktoken.get_encoding("cl100k_base")
```

🔹 Uses the OpenAI tokenizer to calculate token counts.

---

## 🧱 4. Chunking Strategy (Gradient Method)

```python
def split_text_into_chunks(md_text: str, pdf_name: str) -> list:
    lines = md_text.splitlines()
    chunks = []
    current_chunk = []
    current_char_count = 0
    current_page = 1
    buffer = []

    def flush():
        nonlocal current_chunk, current_char_count, buffer
        text = "\n".join(buffer).strip()
        if not text:
            return
        if current_char_count + len(text) > MAX_CHAR:
            chunks.append({
                "pdf_name": pdf_name,
                "page_number": current_page,
                "chunk_text": "\n".join(current_chunk).strip()
            })
            current_chunk.clear()
            current_char_count = 0
        current_chunk.append(text)
        current_char_count += len(text)
        buffer.clear()

    for line in lines:
        if line.startswith("## Page: "):
            match = re.match(r"## Page: (\d+)", line)
            if match:
                current_page = int(match.group(1))
        buffer.append(line)
        if re.search(r"[.!?。！？]$", line):
            flush()

    flush()
    if current_chunk:
        chunks.append({
            "pdf_name": pdf_name,
            "page_number": current_page,
            "chunk_text": "\n".join(current_chunk).strip()
        })

    return [c for c in chunks if len(c["chunk_text"]) >= MIN_CHAR]
```

This function implements a **gradient-style chunking strategy**. Here's how it works:

* 📈 Instead of splitting text at fixed intervals, the script dynamically adjusts chunk size based on accumulated content.
* 🧠 It builds a "gradient" of text by buffering lines until a natural breakpoint (like a period or question mark).
* 🧹 Once a complete sentence or semantic unit is formed, the buffer is flushed to the current chunk.
* 🛑 If the current chunk exceeds a target size (`MAX_CHAR`), it is finalized and a new chunk begins.
* 📌 Page numbers are tracked using lines formatted as `## Page: N`, so each chunk can be traced back to its original source.
* 📐 This ensures chunks are semantically meaningful, not mid-sentence, and not too short or too long.

This method balances readability, coherence, and embedding efficiency—ideal for downstream LLM-based search tasks.

---

## 📄 5. Metadata for Each Chunk

Each chunk includes:

* `pdf_name`: original PDF file name
* `chunk_id`: sequential number
* `page_number`: page the chunk originated from
* `character_count`: number of characters in chunk
* `token_count`: number of tokens in chunk
* `content`: the chunk's actual text

---

## 📊 6. Summary Statistics

For each `.md` file, the script computes:

* Total number of chunks, characters, and tokens
* Average and standard deviation of characters and tokens per chunk
* Placeholder columns for evaluation metrics (`precision`, `recall`, `f1_score`)

These are stored in a summary file:

```bash
chunking_summary.csv
```

---


## ✅ Use Case

These structured, page-aware chunks can be directly embedded and indexed in a vector database, enabling accurate and efficient semantic search in the chatbot application.

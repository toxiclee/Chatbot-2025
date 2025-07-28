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

🔹 Each chunk will aim to be around 500 characters, with a lower and upper bound. Chunks smaller than `MIN_CHAR` are discarded.

---

## 🧐 3. Tokenizer Setup

```python
enc = tiktoken.get_encoding("cl100k_base")
```

🔹 Uses the OpenAI tokenizer to compute token count per chunk using the `cl100k_base` encoding.

---

## 🧱 4. Chunking Strategy (Gradient Method)

```python
def split_text_into_chunks(md_text: str, pdf_name: str) -> list:
    lines = md_text.splitlines()
    chunks = []
    current_chunk = []
    current_char_count = 0
    inside_table = False
    current_page = 1
    buffer = []

    def flush_buffer():
        nonlocal buffer, current_chunk, current_char_count, current_page
        if not buffer:
            return
        section = "\n".join(buffer).strip()
        length = len(section)
        if length == 0:
            return
        if current_char_count + length > MAX_CHAR and current_chunk:
            chunks.append({
                "pdf_name": pdf_name,
                "page_number": current_page,
                "chunk_text": "\n".join(current_chunk).strip()
            })
            current_chunk.clear()
            current_char_count = 0
        current_chunk.append(section)
        current_char_count += length
        buffer.clear()

    for line in lines:
        line = line.rstrip()
        page_match = re.match(r"^## Page: (\d+)", line)
        if page_match:
            current_page = int(page_match.group(1))

        if re.match(r"^\|.*\|$", line):
            inside_table = True
            buffer.append(line)
        elif inside_table and (line.startswith("|") or re.match(r"^[-| ]+$", line)):
            buffer.append(line)
        else:
            if inside_table:
                flush_buffer()
                inside_table = False
            buffer.append(line)
            if re.search(r"[.!?。！？]$", line):
                flush_buffer()

    flush_buffer()
    if current_chunk:
        chunks.append({
            "pdf_name": pdf_name,
            "page_number": current_page,
            "chunk_text": "\n".join(current_chunk).strip()
        })

    return [chunk for chunk in chunks if len(chunk["chunk_text"]) >= MIN_CHAR]
```

🔹 This function reads the input Markdown and segments it into logical chunks:

* Paragraphs or sentences are added to a `buffer` until a sentence-ending character is encountered.
* If the current chunk size exceeds `MAX_CHAR`, a new chunk is flushed.
* Page numbers are inferred from lines like `## Page: 2`, and propagated to the corresponding chunk.
* Tables (identified by pipe `|` symbols) are treated as indivisible blocks.

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


These are stored in a summary file:

```bash
chunking_summary.csv
```

---

## ✅ Use Case

These structured, page-aware chunks can be directly embedded and indexed in a vector database, enabling accurate and efficient semantic search in the chatbot application.

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
```

This function implements a **gradient-style chunking method**, characterized by:

* ✅ **Variable chunk sizes**: Chunks range from 200 to 800 characters, centered around 500.
* ✅ **Semantic boundary splitting**: The script flushes content when encountering sentence-ending punctuation (like `.`, `!`, `?`, `。`, `！`, `？`) to preserve meaning.
* ✅ **Structure preservation**: Markdown tables (`|...|` lines) are treated as atomic units and are never split mid-way.
* ✅ **Page awareness**: `## Page: X` headers are parsed and assigned to each chunk for traceability.

Together, these features ensure that each chunk is semantically coherent, structure-preserving, and page-referenced — ideal for high-quality embedding.

---

## 🧠 Alternative: SemanticGradientSplitter Class (Token-based)

An alternative implementation based on tokens rather than characters can be achieved using a semantic gradient chunker with LangChain’s `RecursiveCharacterTextSplitter`. Example:

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

class SemanticGradientSplitter:
    def __init__(self,
                 breakpoint_threshold_amount=95.0,
                 target_chunk_size=512,
                 min_chunk_size=100):
        self.threshold = breakpoint_threshold_amount
        self.target_size = target_chunk_size
        self.min_size = min_chunk_size
        self._token_len = fast_token_len

        self.base_splitter = RecursiveCharacterTextSplitter(
            chunk_size=target_chunk_size,
            chunk_overlap=int(target_chunk_size * 0.1),
            length_function=self._token_len,
            separators=["\n\n", "\n", ". ", "; ", " ", ""]
        )

    def split(self, text: str):
        return self.base_splitter.split_text(text)
```

🔸 Differences compared to the character-based implementation:

* Token-aware splitting
* Recursively uses punctuation and whitespace for better semantic alignment
* Supports overlap for context continuity
* Does not include page number tracking by default (must be extended)

This method offers higher precision for downstream language model processing, especially with English content and fine-grained control needs.

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

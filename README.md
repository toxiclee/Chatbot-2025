# Chatbot-2025
# ♻️ EPR Research Chatbot – Preprocessing Pipeline

This project builds the preprocessing pipeline for a semantic retrieval chatbot designed to assist users in accessing and querying EPR (Extended Producer Responsibility) research literature.

The pipeline prepares the knowledge base for embedding and semantic search by converting PDFs to Markdown, chunking the content while preserving structure, and exporting chunk metadata to CSVs.

---

## 🧱 Pipeline Overview

```text
📄 PDF files
   ↓
📝 Extract text to Markdown
   ↓
✂️ Semantic chunking with structure preservation
   ↓
📊 Output chunks with metadata to CSV
   ↓
🔍 Embedding → Vector Store → Chatbot retrieval

...

🔗 Qwen Embedding Script (CSV → Markdown-friendly guide)

This document explains the end-to-end script that reads a CSV with a content column, generates embeddings using Qwen/Qwen3-Embedding-0.6B, and writes them back to a new CSV.

📦 1. Import Libraries
```python
import argparse
import pandas as pd
import torch
from tqdm import tqdm
from transformers import AutoTokenizer, AutoModel
```
🔹 Core stack for data I/O (pandas), batching/progress (tqdm), GPU/CPU inference (torch), and the embedding model (transformers).

🤖 2. Load Model & Tokenizer
```python
MODEL_NAME = "Qwen/Qwen3-Embedding-0.6B"
tokenizer = AutoTokenizer.from_pretrained(MODEL_NAME)
model = AutoModel.from_pretrained(MODEL_NAME, trust_remote_code=True, device_map="auto")
```

📄 3. Read CSV & Validate Schema
```python
df = pd.read_csv(args.input)
if "content" not in df.columns:
    raise ValueError("CSV must contain a 'content' column.")
texts = df["content"].fillna("").astype(str).tolist()
```
🔹 Ensures the expected content column exists and normalizes text inputs.

🧠 4. Embedding Function (CLS pooling)
```python
def get_embeddings(texts, batch_size=16, max_length=512):
    embeddings = []
    model.eval()
    with torch.no_grad():
        for i in tqdm(range(0, len(texts), batch_size), desc="Embedding"):
            batch = texts[i:i+batch_size]
            inputs = tokenizer(batch, padding=True, truncation=True,
                               return_tensors="pt", max_length=max_length)
            inputs = {k: v.to(model.device) for k, v in inputs.items()}
            outputs = model(**inputs)
            batch_emb = outputs.last_hidden_state[:, 0, :].cpu()  # CLS token
            embeddings.extend(batch_emb.numpy())
    return embeddings
```
🔹 Uses the [CLS] token vector for each row as a compact sentence embedding (fast, widely used).

⚙️ 5. Generate Embeddings
```python
embeddings = get_embeddings(texts, batch_size=args.batch_size, max_length=args.max_length)
```
🔹 Processes all rows in batches with progress reporting

🧱 6. Build Embedding Columns
```python
import pandas as pd  # already imported
embedding_dim = len(embeddings[0])
cols = [f"emb_{i}" for i in range(embedding_dim)]
emb_df = pd.DataFrame(embeddings, columns=cols)
result_df = pd.concat([df.reset_index(drop=True), emb_df], axis=1)
```
💾 7. Save Output CSV
```python
result_df.to_csv(args.output, index=False)
print(f"✅ Embeddings saved to: {args.output}")
```




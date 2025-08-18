# Explanation of Qwen Embedding Script

This script **reads a CSV file of text chunks, generates embeddings using the Qwen3-Embedding-0.6B model, and saves the results into a new CSV file**.  
These embeddings can later be used for **semantic search, clustering, or Retrieval-Augmented Generation (RAG)**.

---

## Step-by-Step Explanation

### 1. Import libraries
```python
import pandas as pd
import torch
from transformers import AutoTokenizer, AutoModel
from tqdm import tqdm

### 2. Load Model and Tokenizer
MODEL_NAME = "Qwen/Qwen3-Embedding-0.6B"
tokenizer = AutoTokenizer.from_pretrained(MODEL_NAME)
model = AutoModel.from_pretrained(MODEL_NAME, trust_remote_code=True, device_map="auto")

## 3.Read CSV
csv_path = ".../combined_chunks (1).csv"
df = pd.read_csv(csv_path)

##  4. Check column existence
if "content" not in df.columns:
    raise ValueError("CSV file missing 'content' column")


##  5. Define embedding function
def get_embeddings(texts, batch_size=16):
    ...

##  6.Generate Embedding
all_texts = df["content"].fillna("").astype(str).tolist()
embeddings = get_embeddings(all_texts)


##  7. Build Embedding Dataframe

embedding_dim = len(embeddings[0])
embedding_columns = [f"emb_{i}" for i in range(embedding_dim)]
emb_df = pd.DataFrame(embeddings, columns=embedding_columns)

##  8.Combined with Original data
result_df = pd.concat([df.reset_index(drop=True), emb_df], axis=1)

##  9.Save result
output_path = ".../combined_chunks_with_qwen_embedding.csv"
result_df.to_csv(output_path, index=False)





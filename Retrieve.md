# RAG (Retrieval-Augmented Generation)
RAG is an architecture that gives an LLM access to external knowledge at query time. When generate answer, RAG allows the model to take the information retrived from a given knowledge base as context background.
* Advantages: get the updated info without retraining the model; answers are from cited resources; ability to use private data in production.
* Pipeline: Query → Retrieve relevant chunks → Build prompt with those chunks → LLM generates answer.
## Embeddings
A function that turns text (a word, sentence, paragraph) into a vector of numbers, such that texts with similar meaning end up as vectors that are close together in that space.

Example: "How do I reset my password?" and "I forgot my login credentials" would embed to nearby vectors, even though they share almost no words in common. This is the key trick that lets retrieval go beyond keyword matching.

Embeddings are produced by a neural network (an "embedding model") trained specifically so that semantic similarity maps to geometric closeness (usually measured by cosine similarity or dot product).

## Vector databases
A database that is designed to store and manage vector embeddings (with associated metadata/text). It uses algorithms such as:
* approximate nearest neighbor (ANN) algorithms (HNSW, IVF, and LSH)
* Exact Nearest Neighbor (k-NN / Flat Search)
* Compression & Quantization Algorithms (PQ, SQ)
* Metadata Filtering & Hybrid Search Algorithms
depending on the use cases.
## Semantic search
A data retrieval method that focuses on understanding the intent and contextual meaning of a query rather than just matching exact keywords.
## Chunking

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
The method used to split large documents into smaller, manageable pieces before turning them into vector embeddings. Chunk size is used to define the maximum number of tokens or words allowed in a single chunk. Chunk overlap is used to including a small amount of text from the end of one chunk at the start of the next to prevent losing context at the boundaries.
* Fixed-Size Chunking: Cuts text into exact block sizes based on a set number of characters, words, or tokens. It's best for large pipelines where speed and simplicity matter more than deep context matching. The drawback is it may cut sentences right in the middle of a thought.
* Recursive / Sentence-Aware Chunking: Tries to split text at natural breaks like paragraph marks first, then sentences, and finally words. It's best for general articles, blog posts, and text documents.
* Semantic Chunking: Measures the meaning between adjacent sentences and starts a new chunk when the topic or tone shifts. It's best for complex documents that mix multiple distinct topics.
* Document-Structure Chunking: Splits text using existing markers like Markdown headers (##), HTML tags, or page breaks. It's best for technical wikis, manuals, and structured documentation.
## Metadata filtering
A technique to restrict and refine search results using structured, non-vector attributes that attached to the documents. It helps increasing the precision in context retrieval. 

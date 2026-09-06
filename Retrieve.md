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
* Recursive / Sentence Aware Chunking: Tries to split text at natural breaks like paragraph marks first, then sentences, and finally words. It's best for general articles, blog posts, and text documents.
* Semantic Chunking: Measures the meaning between adjacent sentences and starts a new chunk when the topic or tone shifts. It's best for complex documents that mix multiple distinct topics.
* Document Structure Chunking: Splits text using existing markers like Markdown headers (##), HTML tags, or page breaks. It's best for technical wikis, manuals, and structured documentation.
## Metadata filtering
A technique to restrict and refine search results using structured, non-vector attributes that attached to the documents. It helps increasing the precision in context retrieval. 
## Hybrid search
A combination of keyword search and vector (semantic) search, which will merge the two result sets at the end. Keyword search only looks for exact match, while semantic search can miss precise identifiers; combining them fixes both gaps.
## Reranking
A two-stage retrieval process that takes an initial list of candidate documents and uses a more powerful, computationally expensive AI model to re-order them for maximum relevance. In hybrid search or Retrieval Augmented Generation (RAG), it acts as a final filter to ensure the absolute best context reaches the end user or large language model (LLM). 

In a Two-Stage Retrieval Process, the query is first pass through the first stage, which is the Retrieval stage. It is fast but less accurate. E.g. It retrieves top 100-1000 candidates via Bi-Encoder / Vector DB. The second stage is Reranking, which is slower but is highly accurate. E.g. The Cross-Encoder computes exact relevance scores for top 10-25. 

#### Why Reranking is Critical
* Fusion Blindspots: When merging keyword and vector results via algorithms like Reciprocal Rank Fusion (RRF), the math relies on rank positions rather than semantic fit. A reranker evaluates the actual content of the merged list.
* Maximizes LLM Context Windows: LLMs suffer from "lost in the middle" phenomena, where they ignore information placed in the middle of a long prompt. Reranking ensures the top 3 to 5 most vital chunks are placed exactly at the top.
* Balances Speed and Cost: Running a massive Cross-Encoder model across an entire database of millions of documents is too slow and expensive. Applying it only to the top 100 results delivers deep learning accuracy at millisecond speeds.

## Retrieval evaluation
The process of measuring how effectively an information retrieval (IR) system or search layer finds and extracts the most relevant documents or context chunks from a dataset.
### Evaluation Metrics
* Order Unaware (presence-based)
  * Precision@K:
    * Out of the K elements retrieved, what percentage are actually relevant?
    * Ideal Use Case: Minimizing noise in context windows.
  * Recall@K:
    * Out of all the relevant items in the database, what percentage did the system successfully catch in the top K?
    * Ideal Use Case: Finding all necessary evidence for a complex, multi-part prompt.
* Order Aware (rank-based)
  * MRR (Mean Reciprocal Rank):
    * How close to the very top (1st place) is the first relevant result?
    * Ideal Use Case: "Fact seeking" actions (e.g., finding a single specific code or part number).
  * MAP (Mean Average Precision)
    * Evaluates the precise ranking position of all relevant documents in the list, penalizing systems that bury good results.
    * Ideal Use Case: Research style queries requiring multiple diverse, highly ranked sources.
  * NDCG (Normalized Discounted Cumulative Gain)
    * A highly expressive metric that handles graded relevance labels (e.g., scoring a document as "highly relevant," "partially relevant," or "irrelevant") while penalizing items that appear late.
    * Ideal Use Case: The industry standard for evaluating real world search engines and recommendation grids.

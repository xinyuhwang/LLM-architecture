# LLM-architecture
## Transformer Architecture
### 1. Formatting
Assembling conversation into one flat sequence of text that is separated by special role markers. 
### 2. Tokenization
It's the process to split the text into tokens (often sub-word pieces). Each token is assigned with an integer id.
### 3. Embedding
It's the process to convert taken id into a vector of numbers, the numbers correlated to each token's representation in the table of fixed vocabulary entries and the position information that represent the order of each token within the context. The reason of including position information is attention has no inherent sense of order. 
### 4. Context Window
The model's entire working memory, which is the conversation boundary. 
### 5. Attention
The embeddings are processed by identical transformer layers. Each transformer layer process the correlated embeddings based on its trained expertise. After the last transformer layer finish process, the output is a vector that represents the context understands so far.
### 6. Convert the model's contextual understanding into a probability distribution over what the next token could be
#### Scoring: 
The final token's vector (from Attention step) gets multiplied against a matrix (often the same embedding table, reused) to produce one raw score(called a logit) per vocabulary token. 
#### Normalizing:
Those raw scores get passed through a softmax to turn them into actual probabilities (numbers between 0 and 1 that sum to 1).

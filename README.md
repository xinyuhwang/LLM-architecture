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
### 7. Sampling
Randomly draw one token from the probability distribution, using temperature and top-p to control how much the draw favors the most likely tokens versus giving weaker tokens a chance. In other words, this process is to deliberately introduce controlled randomness into the choice.
#### Temperature:
Reshapes the distribution before sampling, by dividing all the raw scores (logits) by a number T, before applying softmax:
* T < 1 → sharpens the distribution (makes the model more confident, more likely to pick the top choice — closer to greedy)
* T = 1 → leaves the distribution as originally computed
* T > 1 → flattens the distribution (gives lower-probability tokens a chance — more random, more "creative")
#### Top-p (nucleus sampling):
* Sort all tokens by probability, highest to lowest
* Keep adding tokens to a "nucleus" until their cumulative probability crosses p (e.g., 0.9)
* Throw away the long tail of implausible tokens
* Renormalize the remaining probabilities and sample from that shrunk set
##### Then the actual sampling step draws one token at random from whatever distribution is left, which are weighted by probability.
### 8. Structured Output
When request JSON or a specific schema, there are two common approaches:
* Prompting-based: the model is just instructed ("respond only in valid JSON matching this schema") and relies on its training to comply.
* Constrained decoding: at each generation step, the sampler is restricted to only tokens that would keep the output valid according to a grammar/schema (e.g., after `{"name":` it's not allowed to sample a token that would break JSON syntax). This gives a hard guarantee of valid structure and is how most "guaranteed JSON" / function-calling features work under the hood.

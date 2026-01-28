LLM = 
	DATA (in tera/peta Bytes)+ 
		ARCHITECTURE (Transformer) + 
			TRAINING (finally trained on large data using *backpropagation*)
![[LLM vs RAG vs Agentic AI.gif]]


ChatGPT = Generative Pre-Trained [Transformer](**https://www.youtube.com/watch?v=wjZofJX0v4M**)  
* Transformer = a specific kind of NN, a machine learning model, 
	* tokens
	* vector embeddings
	* attentions
	* multi-layer perceptron / feed forward layer 
Some facts on ChatGPT-3 : 
* Parameters : 175B
* tokens (vocabulary size) : 50,257 words
* embedding space : 12,288 dimensional
* context size : 2,048

What are their limitations of LLMs?  
* not up-to-date
* no source of answer (less trustworthy)
* risk of responding with incorrect or fabricated information (hallucinations)

How to overcome these limitations ? 
1. User providing the Context with every prompt (tedious)
2. Fine tuning : model can be fine tuned on a smaller, more specific dataset w/ additional training.
3. RAG pipeline : remove the user's responsibility to provide context , augmenting the prompt w/ context is done by RAG

Why RAG ? 
* to overcome limitations of LLMs.
RAG enables the LLM to : 
1. maintain **up-to-date information** or 
2. **access domain-specific knowledge**.
![[Pasted image 20251019115951.png]]
![[Pasted image 20251019120012.png]]
### RAG = Generation that is augmented by retrieval

![[RAG.png]]

https://www.perplexity.ai/page/an-introduction-to-rag-models-jBULt6_mSB2yAV8b17WLDA 
https://zilliz.com/learn/Retrieval-Augmented-Generation 

![[Pasted image 20250905030342.png]]

#### R : Retrieval
Retrieve what ? the content/ relevant chuck of document or database
#### A : Augmented
> **“Augmenting the LLM's prompt input at generation time with external retrieved content.”**

In other words:
- The LLM is given *extra context at runtime* (like search results, document chunks, or notes)
- This **retrieved content is injected (augmented)** into the prompt before the LLM generates
#### G : Generation 
better output Generation with better input (augmented prompt)


![[Pasted image 20250905030444.png]]

![[Pasted image 20250905032359.png]]

![[Pasted image 20250905032423.png]]

![[Pasted image 20250905032524.png]]

![[Pasted image 20250905032555.png]]

![[Pasted image 20250905032607.png]]

## 🔁 Actual Workflow (Correct Timeline)
![[rag pipeline.png]]
Here’s what really happens:
```
`User Input ➝ [Retriever] ➝ Fetch Relevant Chunks
            ➝ [Augment]   ➝ Prompt with Retrieved Chunks
            ➝ [LLM]       ➝ Generate Final Answer`
```

### RAG Architecture

![[RAG arch model.png]]

![[rag f.png]]

The architecture of RAG systems involves several key components: 
1. data preparation, 
2. indexing, 
3. retrieval, and 
4. response generation. 
 
Initially, **external data** is **processed** and transformed into a format suitable for quick retrieval. This involves creating **embeddings** of the data, which are then **indexed** in a vector search engine. When a query is received, RAG systems match the query against these indices to **find the most relevant information**, which is subsequently used to inform the LLM's response. This method not only reduces the likelihood of generating incorrect or misleading information but also allows for the inclusion of citations, enhancing transparency and trust in the generated content.

RAG) systems consist of two primary components: 
1. the retrieval system and 
2. the generative model. 

**The retrieval system** typically employs a vector database to store and efficiently search through document embeddings. This allows for quick identification of relevant information based on semantic similarity to the input query. 
**The generative model**, often a large language model (LLM), is responsible for producing coherent and contextually appropriate responses. 

The RAG mechanism operates in several steps:
1. Query embedding: The input query is converted into a vector representation.
2. Retrieval: The system searches the vector database for documents similar to the query embedding.
3. Context integration: Retrieved documents are combined with the original query.
4. Generation: The LLM uses the augmented input to generate a response.
![[Pasted image 20250905042517.png]]
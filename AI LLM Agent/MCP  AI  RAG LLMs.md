terms : 
- AI/ML, LLM 
- Prompt 
- RAG 
- MCP 
- Agent 
- Tool 
- skills
![[AI Agent Roadmap.png]]
LLM = 
- DATA (in tera/peta Bytes) +
- ARCHITECTURE (Transformer) + 
- TRAINING (finally trained on large data using *backpropagation*)
![[Pasted image 20260422023246.png|360]]
![[Pasted image 20260422023302.png|364]]

![[rag, mcp, agent.gif]]
![[LLM vs RAG vs Agentic AI.gif]]

MCP (Model Context Protocol) 
MCP standardizes how LLMs use tools. Instead of building custom integrations every time, MCP lets models: 
- discover available tools
- call them
- receive structured responses 
- Examples of tools: Files, databases, GitHub, Slack, internal APIs. 
- Important: MCP doesn’t decide what to do. It only defines how tools are exposed and used. !



RAG (Retrieval-Augmented Generation) 
RAG improves what the model knows at runtime. 
Flow: User question → retrieve relevant docs → add to prompt → generate answer. 

Common sources:
- PDFs
- code repositories
- vector databases
- internal documentation 
Best for: 
- company knowledge bases 
- private or up-to-date data 
- reducing hallucinations 
But remember: RAG doesn’t execute actions. It only improves answers.  

AI Agents 
Agents are focused on doing things. 
Typical loop: Observe → Reason → Act → Repeat 
Agents can: 
- call tools 
- write code 
- browse the web 
- store memory 
- run workflows 
- delegate tasks 

How they fit together The most powerful AI systems use all three: 
Agents → decide what to do 
RAG → provide the knowledge 
MCP → connect tools 

Different layers. Same architecture. 


----
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

#### R : Retrieval
Retrieve what ? the content/ relevant chuck of document or database
#### A : Augmented
> **“Augmenting the LLM's prompt input at generation time with external retrieved content.”**

In other words:
- The LLM is given *extra context at runtime* (like search results, document chunks, or notes)
- This **retrieved content is injected (augmented)** into the prompt before the LLM generates
#### G : Generation 
better output Generation with better input (augmented prompt)

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

----

## RAG, MCP, Agents

Topics in AI stack
- RAG
- MCP
- Agent
- Vector Database (storage)
- Fine-tuning (training)
- Prompt Engineering
- Guardrails(Safety)
- LLMOps (Ops)
- Agentic Patters
- Context Window & Memory
- Evals (Evaluations)

**RAG** is about _knowledge_ — it gives AI access to information beyond its training, pulling from your own documents or databases in real time.

**AI Agents** are about _action_ — they let AI plan and execute multi-step tasks autonomously, using tools like search, code execution, or APIs.

**MCP** is about _connectivity_ — it's the universal standard that lets any agent talk to any tool without writing custom integrations from scratch.


### RAG — Retrieval-Augmented Generation
Memory for AI

An AI model like ChatGPT or Claude is trained on data up to a certain date — after that, it knows nothing new. RAG fixes this by giving the AI a way to **look things up** before answering.

> [!NOTE]
> 💡 Think of it like an open-book exam. Without RAG, the AI can only use what it memorised. With RAG, it can open a book and find the right page before writing the answer.

#### How it works — step by step
.1. Your question → 2. Search knowledge base → 3. Retrieve relevant docs → 4. AI reads + answers
#### What's the "knowledge base"?
It can be anything: your company's PDFs, a website, a database of support tickets, product docs — even emails. These are first broken into chunks and stored in a **vector database** (more on that in "More Trends").
#### Real-world examples
• Customer support bot that reads your own help docs  
• Legal AI that searches case law  
• Internal HR assistant that knows your company handbook  
• Medical AI updated with latest research

### AI Agents
AI that takes action

Normal AI just answers questions. An **agent** can actually _do things_ — browse the web, write and run code, send emails, fill forms, use APIs, and chain multiple steps together on its own.

> [!NOTE]
>💡 A regular AI is like a very smart advisor who gives you advice. An AI agent is like a smart employee who goes and actually does the work — including deciding _how_ to do it.
#### What makes an agent different?

1 **Goal**
Given a high-level objective: "Book me a flight to NYC next Friday under $400"

2 **Plan**
Breaks the goal into sub-tasks: search flights, compare prices, check calendar, confirm budget

3 **Act**
Uses tools (web browser, calendar API, booking site) to complete each step

4 **Reflect**
Checks if the result looks right, retries if something failed
#### Popular agent frameworks you'll hear about
`LangGraph` `AutoGen` `CrewAI` `LangChain Agents` `OpenAI Assistants`

These are tools/libraries that help developers build agents without writing everything from scratch.

### MCP — Model Context Protocol
The universal plug

When an AI agent wants to use a tool — say, read your Google Drive or check your calendar — someone has to write custom code for each connection. MCP is a **standard protocol** that makes this plug-and-play, like USB-C for AI tools.

MCP is for **talking to services**. MCP is a communication protocol between an AI and a _service_.
MCP is for — services with structured data and business logic.
MCP is NOT for — local apps, files, and visual UIs.

MCP is a **standard language for AI to talk to online services** — it only makes sense when there is a server on the other end with data, logic, and permissions to manage.

> [!NOTE]
> 💡 Before USB, every device had its own unique cable. USB standardised everything. MCP does the same for AI — instead of custom code for every tool, any AI can connect to any MCP-compatible tool instantly.

![[MCP for.png]]
#### Without MCP vs. with MCP

> [!Without MCP]
>  
> Developer writes custom code to connect AI → Google Drive  
> Then more custom code for AI → Slack  
> Then more for AI → GitHub  
> And so on… forever.

> [!With MCP]
>  
> Google Drive publishes an MCP server.  
> Slack publishes one. GitHub too.  
> Any AI that speaks MCP can use _all_ of them — no custom code needed.

![[Pasted image 20260315115231.png]]

![[AI stack.png]]


#### Layer 2

![[Tool.png]]

#### Layer 3
![[Skills.png]]


#### Layer 
![[Orchestration.png|697]]


![[Google AI stack.png]]
Here's the flow I'd draw on a whiteboard. Tap any box to go deeper.
<img width="1266" height="742" alt="image" src="https://github.com/user-attachments/assets/28880786-0ae4-4f3e-a0ca-0a4e7c0ef1fd" />

**The 20-second version:** RAG (Retrieval-Augmented Generation) means the LLM answers from your company's data instead of only what it learned in training. You index your documents ahead of time. When a question comes in, you fetch the most relevant chunks and add them to the prompt as context. The model then answers from those chunks and cites them.

**Explain it as two pipelines:**

1. **Ingestion (offline):** Documents land in Blob Storage. An Azure Function picks them up, and Document Intelligence pulls out the text. The text is split into chunks of roughly 500–1000 tokens with some overlap. An Azure OpenAI embedding model (e.g. `text-embedding-3-large`) turns each chunk into a vector. The chunks go into Azure AI Search along with metadata such as source, ACL group and timestamp.
2. **Query (runtime):** The ASP.NET Core API embeds the user's question. It runs a hybrid search in AI Search (keyword plus vector, with semantic reranking) to get the top 3–5 chunks. It builds a prompt like "Answer only from this context, cite sources", sends it to the GPT deployment and streams the answer back.

**The .NET side:** Use `Azure.Search.Documents` for retrieval. Use Semantic Kernel or `Microsoft.Extensions.AI` to orchestrate the chat and embedding calls. Use `DefaultAzureCredential` with Managed Identity so there are no API keys in config.

**Tie it to your own work (this is what makes it stand out):**
- Run ingestion with **Durable Functions** (fan-out per document, retries, status tracking), the same pattern as your re-sharing orchestration.
- Use your **content-hash de-duplication** to skip re-embedding unchanged files. That cuts embedding cost the same way it cut your media-processing cost.

**Follow-ups interviewers usually ask:**
- **Security trimming:** filter search results by the user's Entra ID groups so nobody retrieves chunks they aren't allowed to see.
- **Hallucinations:** use a grounded system prompt, require citations, and say "I don't know" when retrieval comes back weak.
- **Quality:** tune chunk size and top-k, and measure groundedness and relevance with Foundry evaluations.
- **Cost and latency:** cache embeddings and frequent answers, stream responses, and use a smaller model where it's good enough.

I can also write a sample `/ask` endpoint in C# that you could walk through in an interview.



Hybrid Search in Azure AI Search means combining two search methods:

1. Keyword Search (BM25): Finds documents containing the exact words entered by the user.
2. Vector Search: Finds documents based on their meaning, even when they use different words.

Azure AI Search combines the results from both methods and merges them into one ranked result list, typically using Reciprocal Rank Fusion (RRF).

## Simple example

Imagine an investigator searches in your CCTV Investigate application:

"Find videos where processing failed because of a video decoding issue."

Your search index contains:

| Document | Description                                    |
| -------- | ---------------------------------------------- |
| DOC-101  | Video decoding failed due to unsupported codec |
| DOC-102  | Video processing error occurred                |
| DOC-103  | Successful video upload                        |
| DOC-104  | Decoder failure reported for CCTV footage      |

What happens?

- Keyword search finds documents containing words like `video`, `decoding`, and `failed`.
- Vector search finds documents with similar meaning, such as `decoder failure`, even if the exact words differ.
- Hybrid search merges both result sets and ranks the most relevant documents higher.

 <img width="848" height="509" alt="image" src="https://github.com/user-attachments/assets/a614ae70-619c-4517-9352-c1d8176bc2af" />


## Difference between search techniques

| Technique          | What it does                                             |
| ------------------ | -------------------------------------------------------- |
| Keyword Search     | Finds matching words                                     |
| Vector Search      | Finds similar meaning                                    |
| Hybrid Search      | Combines keyword + vector                                |
| Semantic Reranking | Reorders retrieved results based on contextual relevance |

## How it helps in RAG

For your Investigate application, the architecture could be:

1. User asks a question.
2. Azure AI Search performs hybrid search.
3. Relevant case/evidence documents are retrieved.
4. Semantic reranking can improve their order.
5. Top relevant documents are passed to the LLM.
6. LLM generates an answer with source citations.

Important: Hybrid search and semantic reranking are different. Hybrid search combines two retrieval methods; semantic reranking improves the order of the retrieved results.

## Interview-ready answer

"Hybrid search in Azure AI Search combines traditional keyword search using BM25 with vector search using embeddings. Keyword search finds exact terms, while vector search understands the semantic meaning of the query. Azure AI Search combines both result sets using Reciprocal Rank Fusion (RRF) to provide more relevant search results. We can then apply semantic reranking and use the retrieved documents as context for RAG."

Remember: Keyword Search + Vector Search = Hybrid Search.




Solution architect:

Expect the panel to push on trade-offs and non-functional requirements more than on how RAG works. Here are the likely cross-questions, each with the angle a strong answer should take.

**Design choices**
1. *Why RAG and not fine-tuning?* Use RAG when knowledge changes often, answers need citations, or access is per-user. Fine-tuning changes style and behaviour, not facts. The two can be combined.
2. *Why Azure AI Search and not Cosmos DB vector, PostgreSQL pgvector or Qdrant?* AI Search gives you hybrid search, semantic reranking, security filters and managed indexers. pgvector or Cosmos DB fits when you want vectors next to your transactional data and lower cost.
3. *How did you choose chunk size and overlap?* Pick by document type, test it, then measure. Small chunks are precise but lose context. Large chunks keep context but add noise and tokens. Mention structure-aware chunking that splits by headings and tables.
4. *Hybrid vs pure vector search?* Use hybrid. Vectors miss exact terms like product codes, error IDs and names, and keyword search catches them. Semantic reranking on top improves precision.

**Security (expect deep probing)**
5. *How do you stop user A from seeing user B's documents?* Store ACL groups on each chunk and apply a security filter from the user's Entra ID token on every query. Never rely on the prompt to enforce access.
6. *What about prompt injection hidden inside a document?* Treat retrieved content as untrusted. Use a strict system prompt and Azure AI Content Safety Prompt Shields. Give the model no tool or write access it doesn't need.
7. *PII and data residency?* Use private endpoints, Managed Identity, keep data in-region, set abuse-monitoring and data-retention terms, and redact PII before indexing if needed.

**Scale, performance, cost**
8. *Ingesting 10 million documents: how?* Event Grid → queue → Durable Functions fan-out, with batched embedding calls and backoff on 429s. Hash files so only changed ones get re-embedded. This is your strongest story, so tell it.
9. *Azure OpenAI throttles you. What happens?* Plan TPM quotas, put APIM in front as an AI gateway to load-balance across regions or deployments, use Polly retries with jitter, and consider PTU for predictable load.
10. *What's your latency budget?* Embedding the query plus search takes roughly a few hundred ms, and generation dominates. Stream tokens, cap context size, and cache frequent questions with a semantic cache.
11. *How do you control cost?* Use smaller models for simple queries (routing), limit top-k and prompt size, cache, and avoid re-embedding. Track tokens per tenant.

**Quality and operations**
12. *How do you know it's answering correctly?* Build a golden question set and measure groundedness, relevance and retrieval recall with Foundry evaluations. Run these in CI before you change prompts, models or chunking.
13. *Upgrading the embedding model: what breaks?* Old and new vectors aren't compatible, so you re-embed everything. Use a blue-green index with an alias swap.
14. *A document gets deleted or updated: how does the index stay in sync?* Use change events or indexer change detection, and delete chunks by document ID. Talk about how stale the index can be and what's acceptable.
15. *What do you log and monitor?* Log the query, retrieved chunk IDs, scores, tokens, latency and user feedback in App Insights, masking PII. This lets you tell a retrieval failure apart from a generation failure.

**Architect-level curveballs**
- *Multi-tenant SaaS: one index or one per tenant?* It's a trade-off between isolation, cost and index limits. Use a tenant filter for many small tenants and a dedicated index for large or regulated ones.
- *When would you NOT use RAG?* When the data is structured (use text-to-SQL or function calling instead), when the answer must be exact and deterministic, or when the corpus is small enough to fit in the context window.
- *What fails first in production?* Usually retrieval quality, not the LLM. Bad chunking or missing metadata produces confident wrong answers.
- *Build vs buy?* Compare Azure AI Foundry "on your data" or Copilot Studio with a custom build, based on how much control you need over chunking, security and evaluation.

The safest pattern when you answer: state the trade-off, say what you'd pick, say how you'd measure it, then connect it to something you've actually built.

I can turn these into flashcards so you can drill them before the interview.




Here are the same questions, each with an answer in plain English.

**Design choices**

**1. Why use RAG instead of fine-tuning the model?**
RAG looks up your documents at the moment someone asks a question. Fine-tuning teaches the model new habits, but it's bad at remembering facts. If your data changes often, or you need to show where an answer came from, RAG is the better choice.

**2. Why Azure AI Search and not another vector database?**
AI Search can search by meaning and by exact words at the same time. It also re-ranks results and can filter by user permissions. If your data already lives in PostgreSQL or Cosmos DB, keeping the vectors there can be simpler and cheaper.

**3. How did you decide the chunk size?**
Small chunks find precise answers but can lose context. Big chunks keep context but bring in extra noise and cost more tokens. I start around 500–1000 tokens with a little overlap, split by headings where possible, then test and adjust.

**4. Why hybrid search instead of only vector search?**
Vector search understands meaning but can miss exact things like error codes or product IDs. Keyword search catches those. Combining both gives better results.

**Security**

**5. How do you stop users from seeing documents they shouldn't?**
Each chunk stores which groups are allowed to see it. When a user asks something, we read their groups from their login token and filter the search results. The model only ever sees chunks that user is allowed to see.

**6. What if a document contains hidden instructions to trick the AI (prompt injection)?**
We treat document text as data, never as commands. We use a strict system prompt and Azure's Prompt Shields to detect attacks. The model also gets no permission to do anything risky, like changing or deleting data.

**7. How do you protect personal data?**
Services talk over private networks, we use Managed Identity instead of passwords or keys, and data stays in the required region. If needed, we remove personal details before storing documents.

**Scale, speed, cost**

**8. How would you load 10 million documents?**
Don't process them one by one. New files trigger events that go into a queue, and Durable Functions process many files in parallel with automatic retries. We hash each file so unchanged files aren't processed again, which saves time and money.

**9. What if Azure OpenAI says "too many requests"?**
We retry with small waits in between. We put API Management in front to spread load across several deployments or regions. For steady heavy traffic, we buy reserved capacity (PTU).

**10. How do you keep answers fast?**
Searching is quick; generating the answer takes most of the time. So we stream the answer word by word, send only the most useful chunks, and cache answers to common questions.

**11. How do you keep costs low?**
Use a cheaper model for easy questions, send fewer chunks, cache repeated answers, and never re-process unchanged files. Track token usage per customer so you know who costs what.

**Quality and operations**

**12. How do you know the answers are correct?**
Keep a fixed list of test questions with known good answers. Every time we change the prompt, model or chunking, we run the tests and check the answers are based on the documents and actually relevant.

**13. What happens if you change the embedding model?**
Vectors from the old and new model can't be mixed, so every document must be processed again. We build a new index alongside the old one, then switch over once it's ready, with no downtime.

**14. How do you handle updated or deleted documents?**
When a file changes, an event tells the system to update or remove its chunks. The question to agree on is how quickly the index must catch up, for example within minutes or overnight.

**15. What do you monitor?**
The question, which chunks were found, their scores, token usage, response time and user feedback (with personal data masked). This shows whether a bad answer came from bad search or from the model.

**Tricky architect questions**

**One index for all customers or one per customer?**
For many small customers, use one shared index filtered by customer ID, which is cheaper. For big or highly regulated customers, give them their own index for stronger separation.

**When should you NOT use RAG?**
When the data sits in database tables, let the AI write queries or call APIs instead. When answers must be exactly right every time, use normal code. When the data is small enough to fit in one prompt, you don't need search at all.

**What usually breaks first in production?**
Usually the search part, not the AI model. If chunking is poor or important info is missing, the model confidently gives a wrong answer.

**Build it yourself or use a ready-made Azure tool?**
Ready-made tools like Copilot Studio are faster to launch. A custom build gives full control over security, chunking and testing. Choose based on how much control the business needs.

**A simple formula for any answer:** say the options, say which one you'd pick and why, say how you'd measure it, and give a real example from your own work.

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

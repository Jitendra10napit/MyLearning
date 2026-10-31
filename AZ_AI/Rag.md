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

# Generative AI & Agentic AI – Interview Preparation Guide

**Role:** Solution Architect / Lead .NET Engineer  
**Experience positioning:** 13+ years of software engineering, .NET, Azure, distributed systems, and architecture.

---

# PART 1: GENERATIVE AI FUNDAMENTALS

## 1. What is Artificial Intelligence (AI)?

Artificial Intelligence is a technology that enables machines to perform tasks that typically require human intelligence.

Examples:
- Understanding natural language.
- Recognizing images.
- Making predictions.
- Solving problems.
- Automating decisions.

### AI categories

| Category | Description | Example |
|---|---|---|
| AI | Broad field of intelligent systems | Recommendation systems |
| Machine Learning | Learns patterns from data | Fraud detection |
| Deep Learning | Uses neural networks with multiple layers | Image recognition |
| Generative AI | Generates new content | ChatGPT |
| Agentic AI | Uses models and tools to perform tasks toward goals | AI developer assistant |

## 2. What is Generative AI?

Generative AI is a branch of AI that creates new content based on learned patterns.

It can generate:
- Text
- Source code
- Images
- Audio
- Video
- Structured data

**Real-world example:**

A developer asks:

"Generate an ASP.NET Core Web API with JWT authentication and Entity Framework Core."

A Generative AI model can generate:
- Controller classes.
- Service interfaces.
- Repository implementations.
- DTOs.
- Authentication configuration.
- Unit tests.

**Interview answer:**

Generative AI uses trained models to generate new content based on user prompts and contextual information. LLMs are one of the most widely used technologies behind text-based Generative AI applications.

## 3. What is an LLM?

LLM stands for Large Language Model.

An LLM is a model trained on large amounts of text and other data to understand and generate human-like language.

Examples:
- GPT models from OpenAI.
- Claude models from Anthropic.
- Gemini models from Google.
- Llama models from Meta.

### How does an LLM work?

1. User provides a prompt.
2. Input text is converted into tokens.
3. The model processes the tokens using its neural network.
4. Transformer attention mechanisms help identify relationships between tokens.
5. The model predicts and generates subsequent tokens.
6. The application returns the generated response.

### Important LLM terminology

| Term | Meaning |
|---|---|
| Token | A unit of text processed by the model |
| Tokenization | Converting text into tokens |
| Context window | Maximum input and conversation context the model can process |
| Transformer | Neural network architecture used by many modern LLMs |
| Attention | Mechanism that helps the model focus on relevant tokens |
| Hallucination | Plausible but incorrect or unsupported output |
| Inference | Running a trained model to generate predictions or responses |
| Fine-tuning | Further training a model to adapt its behavior |
| Prompt engineering | Designing effective instructions and input context |

### What is a Transformer?

Transformer is a neural network architecture that uses attention mechanisms to understand relationships between tokens.

It is a foundation of modern language models.

**Interview answer:**

Transformers allow models to process relationships between different parts of input text efficiently. Self-attention helps the model understand which tokens are relevant to other tokens when generating an output.

---

# PART 2: UNDERSTANDING AI TOOLS AND MODELS

## 4. ChatGPT vs Claude vs GitHub Copilot vs Ollama

These tools are not all the same type of technology.

| Tool | Description | Main purpose |
|---|---|---|
| ChatGPT | OpenAI AI assistant | Learning, reasoning, architecture, coding |
| Claude | Anthropic AI assistant | Document analysis, coding, reasoning |
| GitHub Copilot | AI coding assistant | Code completion and development assistance |
| Ollama | Local model runtime | Running compatible LLMs locally |

### ChatGPT

An AI assistant powered by OpenAI models.

Uses:
- Generate C# code.
- Explain architecture.
- Design microservices.
- Analyze documents.
- Generate SQL queries.

### Claude

An AI assistant from Anthropic.

Uses:
- Code reviews.
- Large-document analysis.
- Technical problem-solving.
- Software development assistance.

### GitHub Copilot

An AI coding assistant integrated into supported development environments.

Uses:
- Code completion.
- Unit test generation.
- Code explanations.
- Refactoring suggestions.
- Coding agent workflows.

### Ollama

Ollama allows developers to download and run supported language models locally.

Example:

```bash
ollama pull llama3.2

ollama run llama3.2
```

Uses:
- Local AI experimentation.
- Prototyping.
- Local inference.
- Development without relying on a hosted inference API.

Important: Ollama is not an LLM itself. It is a runtime for running compatible models.

### Hosted vs Local LLM

| Feature | Hosted LLM | Local LLM |
|---|---|---|
| Hosting | External provider | Your infrastructure |
| Setup | Easier to start | Requires model and hardware setup |
| Scaling | Provider-managed options | You manage capacity |
| Privacy | Depends on provider and configuration | Can remain local |
| Cost | Usage-based pricing often applies | Infrastructure and operational costs |
| Control | Provider-dependent | More deployment control |

---

# PART 3: EMBEDDINGS

## 5. What is an Embedding?

An embedding is a numerical representation of data, such as text, that captures semantic information.

It converts text into a vector containing floating-point numbers.

Example:

Input:

"How can I reset my password?"

Embedding model produces an illustrative vector:

```text
[0.21, -0.45, 0.82, 0.17, ...]
```

The vector represents the text in a mathematical space.

Another sentence:

"How do I recover my password?"

Will generally have a similar vector representation because both sentences have related meanings.

### Why do we need embeddings?

Embeddings help applications:
- Find semantically similar documents.
- Implement semantic search.
- Build RAG applications.
- Identify duplicate support tickets.
- Recommend similar products.
- Classify text.

## 6. OpenAI Embedding Models

| Model | Dimensions by default | Usage |
|---|---:|---|
| text-embedding-3-small | 1,536 | Cost-effective semantic search and RAG |
| text-embedding-3-large | 3,072 | Higher-quality retrieval where justified |
| text-embedding-ada-002 | 1,536 | Legacy embedding model |

The newer embedding models support configurable dimensions.

### How does embedding work?

```text
Input text
    |
    v
Embedding model
    |
    v
Numerical vector
    |
    v
Vector database
    |
    v
Similarity search
    |
    v
Relevant documents
```

Important: Embedding models generate vectors, not natural-language answers.

## 7. What is Cosine Similarity?

Cosine similarity measures the directional similarity between two vectors.

Formula:

Cosine Similarity = (A . B) / (||A|| * ||B||)

Interpretation:
- Close to 1: Highly similar direction.
- Around 0: Little directional similarity.
- Close to -1: Opposite direction.

For normalized embeddings, cosine similarity is equivalent to the dot product.

Example:

| Sentence 1 | Sentence 2 | Expected relationship |
|---|---|---|
| Reset password | Recover password | High similarity |
| Reset password | Cook pasta | Low similarity |

The exact scores depend on the model and data. Similarity is not a guarantee that two statements are factually equivalent.

---

# PART 4: VECTOR DATABASES

## 8. What is a Vector Database?

A vector database stores numerical embeddings and supports similarity searches.

Example:

An organization has 10,000 technical documents.

A developer asks:

"How do we handle failed messages in Azure Service Bus?"

The system searches vector representations to retrieve relevant documents about:
- Dead-letter queues.
- Message retries.
- Message locks.
- Idempotency.

### Popular vector search technologies

| Technology | Description |
|---|---|
| Azure AI Search | Enterprise search with vector and hybrid search |
| FAISS | Vector similarity-search library |
| PostgreSQL + pgvector | Relational database with vector capabilities |
| Qdrant | Vector database |
| Pinecone | Managed vector database |

### What is metadata filtering?

Metadata stores additional information associated with a vector.

Example:

```json
{
  "documentId": "DOC101",
  "department": "Engineering",
  "category": "Architecture",
  "version": "2.0",
  "content": "Azure Service Bus retry policy..."
}
```

Metadata filtering helps restrict retrieval based on permissions, departments, document types, versions, or other attributes.

### What is Hybrid Search?

Hybrid search combines:

1. Keyword search.
2. Vector search.

Example:

A developer searches for:

"Error code CS0311 in EmployeeApi.API."

Keyword search is useful for exact identifiers such as CS0311. Vector search can find related troubleshooting documentation.

Combining both can improve retrieval quality.

---

# PART 5: RAG – RETRIEVAL-AUGMENTED GENERATION

## 9. What is RAG?

RAG stands for Retrieval-Augmented Generation.

RAG is an architecture that retrieves relevant information from external data sources and supplies it to an LLM to generate a contextual answer.

### Why do we need RAG?

LLMs have limitations:

- Their training knowledge may be outdated.
- They do not automatically know private company documents.
- They may generate unsupported answers.
- Retraining a model whenever company documents change is expensive and unnecessary for many knowledge-retrieval use cases.

RAG addresses these problems by retrieving relevant external knowledge at query time.

## 10. RAG Architecture

### A. Document ingestion process

```text
PDF / Word / Database / APIs
             |
             v
       Document Loader
             |
             v
       Text Extraction
             |
             v
          Chunking
             |
             v
      Embedding Model
             |
             v
       Vector Database
```

### B. User query process

```text
        User Question
              |
              v
       Query Embedding
              |
              v
      Vector Similarity Search
              |
              v
       Retrieve Top-K Chunks
              |
              v
       Construct LLM Prompt
              |
              v
          LLM Model
              |
              v
       Contextual Answer
```

## 11. Important RAG concepts

### Chunking

Chunking means splitting large documents into smaller meaningful sections.

Example:

A 100-page PDF is split into multiple chunks.

Why?
- Reduces input size.
- Improves retrieval relevance.
- Helps retrieve specific information.
- Controls token consumption.

A practical starting point is 300–800 tokens per chunk with some overlap, then evaluate and tune based on document structure.

### Overlapping chunks

A small portion of text is repeated between adjacent chunks to preserve context across boundaries.

### Top-K retrieval

Retrieves the K most relevant document chunks.

Example:

```text
Top-K = 5

Retrieve the five most relevant chunks.
```

### RAG advantages

- Uses company-specific knowledge.
- Reduces dependence on model training knowledge.
- Can provide source citations.
- Documents can be updated without retraining the LLM.
- Supports permission-aware retrieval when designed correctly.

### RAG limitations

- Poor chunking leads to poor retrieval.
- Incorrect retrieval leads to incorrect answers.
- Large context increases cost and latency.
- Data access controls must be enforced.
- RAG does not eliminate hallucinations.

### RAG vs Fine-tuning

| RAG | Fine-tuning |
|---|---|
| Retrieves external information | Further trains model behavior |
| Good for changing knowledge | Good for adapting task behavior |
| Documents can be updated independently | Requires training workflow |
| Provides retrieved source context | Does not inherently provide source citations |
| Often easier for enterprise knowledge assistants | Useful for specialized model behavior |

**Interview answer:**

I would generally prefer RAG when an application needs access to frequently changing enterprise knowledge. Fine-tuning is more appropriate when I need to adapt model behavior, style, or performance on a specific task. Both approaches can be combined.

---

# PART 6: PROMPT ENGINEERING AND LLM PARAMETERS

## 12. What is Prompt Engineering?

Prompt engineering is the process of designing instructions and context to guide an LLM toward a useful and reliable response.

Example:

```text
You are a senior .NET Solution Architect.

Review the following ASP.NET Core code.

Identify:
1. SOLID principle violations.
2. Performance issues.
3. Security risks.
4. Design improvements.

Provide corrected C# code.
```

### Types of prompts

| Type | Description |
|---|---|
| Zero-shot | Ask directly without examples |
| One-shot | Provide one example |
| Few-shot | Provide multiple examples |
| System instructions | Define model behavior and boundaries |
| Chain-of-thought prompting | Ask for reasoning; internal reasoning should not be exposed as a requirement |
| Structured prompting | Request output in a defined schema |

### Important LLM parameters

| Parameter | Purpose |
|---|---|
| Temperature | Controls randomness or variability |
| Max output tokens | Limits response length |
| Top-P | Nucleus sampling parameter |
| Frequency penalty | Can discourage repetitive tokens |
| Presence penalty | Can encourage introducing new topics |

Not all models or API endpoints support all parameters.

---

# PART 7: FUNCTION CALLING AND STRUCTURED OUTPUT

## 13. What is Function Calling?

Function calling allows an LLM to request that an application execute an external function or tool.

Example:

User:

"Find order 1052."

The model may return:

```json
{
  "name": "GetOrder",
  "arguments": {
    "orderId": 1052
  }
}
```

The application then:

1. Validates the tool request.
2. Checks authorization.
3. Calls the C# method.
4. Retrieves order information.
5. Returns the result to the model.
6. Generates the final response.

Important: The LLM does not directly execute arbitrary application code. The host application controls tool execution.

### Function Calling vs Structured Output

| Function Calling | Structured Output |
|---|---|
| Requests execution of a tool | Requests output matching a schema |
| Useful for external operations | Useful for reliable data formatting |
| Example: GetOrder(1052) | Example: Return OrderSummary JSON |

---

# PART 8: AGENTIC AI

## 14. What is Agentic AI?

Agentic AI refers to AI systems that can use models, tools, and feedback to pursue a goal through multiple steps.

An AI agent can:
- Understand a user objective.
- Plan or select next actions.
- Invoke tools.
- Observe results.
- Adjust its approach.
- Complete a task within defined permissions.

### Example: Developer troubleshooting agent

User:

"Investigate why Employee API returns HTTP 401."

The agent may:

1. Inspect API configuration.
2. Review JWT authentication settings.
3. Check token issuer and audience.
4. Review approved application logs.
5. Identify a likely authentication problem.
6. Recommend a fix.
7. Generate test cases.

The agent should not modify production configuration without authorization and appropriate approval.

## 15. AI Assistant vs RAG vs AI Agent

| Capability | AI Assistant | RAG | AI Agent |
|---|---|---|---|
| Answers questions | Yes | Yes | Yes |
| Retrieves external knowledge | Optional | Core capability | Optional |
| Uses tools | Optional | Optional | Core capability |
| Multi-step tasks | Limited | Not inherent | Yes |
| Executes workflows | Not inherent | Not inherent | Can do so with tools |

An agent can use RAG as one of its capabilities.

## 16. Agent Architecture

```text
             User Request
                  |
                  v
          Agent Orchestrator
                  |
                  v
             LLM Model
                  |
          Decision / Planning
                  |
          +-------+--------+
          |       |        |
          v       v        v
        RAG     Tools    Memory
          |       |        |
          +-------+--------+
                  |
                  v
           Observe Results
                  |
                  v
          Continue or Finish
                  |
                  v
            Final Answer
```

### Agent components

| Component | Responsibility |
|---|---|
| LLM | Reasoning and action selection |
| Instructions | Define agent role and constraints |
| Tools | Provide external capabilities |
| Memory/state | Maintain relevant context and workflow state |
| Orchestrator | Controls execution and tool calls |
| Guardrails | Enforce safety and business policies |
| Human approval | Authorizes sensitive actions |

### Types of agents

1. Single agent: One agent handles a task.
2. Multi-agent: Multiple specialized agents collaborate.
3. Supervisor agent: Coordinates other agents.
4. Sequential agent: Executes predefined steps.
5. Collaborative agents: Share results to complete a common goal.

Multi-agent systems introduce additional orchestration complexity, so use them only when specialization or independent execution provides measurable value.

---

# PART 9: AGENT SKILLS

## 17. What are Agent Skills?

A skill is a reusable set of task-specific instructions, knowledge, and optionally scripts or supporting resources.

Example skills for a development team:

| Skill | Responsibility |
|---|---|
| Code Review Skill | Review code quality and SOLID principles |
| API Design Skill | Validate REST conventions and API standards |
| Unit Testing Skill | Generate xUnit test cases |
| SQL Optimization Skill | Review query performance and indexing |
| Azure Architecture Skill | Evaluate Azure architecture choices |
| Documentation Skill | Generate technical documentation |

### Example: Code Review Skill

```text
Role:
Act as a senior .NET reviewer.

Instructions:
1. Review SOLID principles.
2. Identify security vulnerabilities.
3. Detect performance issues.
4. Check exception handling.
5. Review asynchronous operations.
6. Suggest unit tests.

Output:
- Findings
- Severity
- Explanation
- Recommended fix
```

Skills are not necessarily independent LLMs. They package reusable expertise and workflows that an agent can apply.

---

# PART 10: MODEL CONTEXT PROTOCOL (MCP)

## 18. What is MCP?

MCP stands for Model Context Protocol.

It is an open protocol that standardizes how AI applications connect to external tools, data sources, and context.

Think of MCP as a standardized integration mechanism between AI applications and external capabilities.

### Example

You have:
- GitHub repository.
- Azure DevOps work items.
- Technical documentation.
- SQL Server.
- Application monitoring.

Instead of implementing every integration differently, you can expose appropriate capabilities through MCP servers.

An MCP client can discover and invoke tools exposed by these servers.

### MCP Architecture

```text
          AI Application
          (MCP Host)
               |
               v
           MCP Client
               |
       MCP Protocol
               |
       +-------+--------+
       |       |        |
       v       v        v
     GitHub  Database  Azure DevOps
     Server   Server     Server
       |       |          |
       v       v          v
   Repository Data      Work Items
```

### MCP core concepts

| Concept | Description |
|---|---|
| MCP Host | Application that contains and coordinates MCP clients |
| MCP Client | Connects to an MCP server |
| MCP Server | Exposes capabilities to clients |
| Tools | Executable operations |
| Resources | Context or data exposed by servers |
| Prompts | Reusable prompt templates |

### MCP vs Function Calling

| Function Calling | MCP |
|---|---|
| Model requests a tool call | Standardizes external tool integration |
| Usually application-specific tool definitions | Reusable protocol-based integrations |
| Application handles execution | MCP client communicates with MCP server |
| Defines how the model requests an operation | Defines how capabilities and context are exposed and accessed |

MCP does not replace function calling. They can work together.

### MCP security considerations

- Authenticate clients and servers.
- Authorize every operation.
- Validate all inputs.
- Use least-privilege access.
- Protect secrets.
- Audit tool execution.
- Require human approval for sensitive operations.

---

# PART 11: LANGCHAIN AND AI ORCHESTRATION

## 19. What is LangChain?

LangChain is a framework for developing LLM-powered applications.

It provides abstractions for:
- Model integration.
- Prompt templates.
- Retrieval.
- Tool calling.
- Chains and workflows.
- Agent orchestration.

LangChain has a Python and JavaScript/TypeScript ecosystem. For .NET, evaluate Microsoft.Extensions.AI, Semantic Kernel, and supported agent frameworks according to your architecture.

### Related technologies

| Technology | Purpose |
|---|---|
| LangChain | LLM application framework |
| LangGraph | Stateful agent workflows and orchestration |
| Semantic Kernel | AI integration and orchestration framework |
| Microsoft.Extensions.AI | .NET abstractions for chat and embeddings |
| Ollama | Local model runtime |
| MCP | Standardized external tool and context integration |

### Important distinction

- LangChain is a framework.
- LangGraph is an orchestration approach/framework.
- Ollama is a model runtime.
- MCP is a protocol.
- GPT, Claude, and Llama are model families.
- ChatGPT and Claude are AI assistant products.

---

# PART 12: ENTERPRISE AZURE AI ARCHITECTURE

## 20. Reference Architecture for an Enterprise AI Assistant

Consider building an internal knowledge assistant for your engineering organization.

### Technology stack

| Layer | Technology |
|---|---|
| Frontend | Angular |
| Backend | ASP.NET Core Web API |
| Identity | Microsoft Entra ID |
| Model inference | Azure OpenAI or another supported model provider |
| Embeddings | OpenAI embedding model or supported Azure deployment |
| Vector search | Azure AI Search |
| Document storage | Azure Blob Storage |
| Asynchronous processing | Azure Service Bus |
| Background processing | Azure Functions |
| Secrets | Azure Key Vault |
| Monitoring | Application Insights |
| Deployment | Azure DevOps CI/CD |

### Architecture

```text
                 Angular UI
                     |
                     v
              Azure API Management
                     |
                     v
             ASP.NET Core Web API
                     |
          +----------+-----------+
          |          |           |
          v          v           v
        LLM        RAG         Agent
          |          |           |
          |          v           |
          |     Azure AI Search  |
          |                      |
          +----------+-----------+
                     |
                     v
                AI Services

Document Ingestion:

Blob Storage
     |
     v
Azure Service Bus
     |
     v
Azure Functions
     |
     v
Text Extraction
     |
     v
Chunking and Embeddings
     |
     v
Azure AI Search
```

### Why use Azure Service Bus?

Document ingestion can be asynchronous because processing large documents may take time.

Service Bus provides:
- Message queues.
- Decoupled processing.
- Retry handling.
- Dead-letter queues.
- Message delivery guarantees with appropriate application handling.

### Why use Azure Functions?

Azure Functions can process document ingestion events without requiring a continuously running application instance.

### Why use API Management?

- Centralized API gateway.
- Authentication and authorization policies.
- Rate limiting.
- API versioning.
- Monitoring and governance.

---

# PART 13: .NET IMPLEMENTATION

## 21. Calling OpenAI Embeddings API using curl

Set your API key:

```bash
export OPENAI_API_KEY="your-api-key"
```

Call the embeddings endpoint:

```bash
curl https://api.openai.com/v1/embeddings \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
    "model": "text-embedding-3-small",
    "input": "Azure Service Bus supports asynchronous messaging."
  }'
```

Expected response structure:

```json
{
  "object": "list",
  "data": [
    {
      "object": "embedding",
      "index": 0,
      "embedding": [
        0.0123,
        -0.0456,
        0.0789
      ]
    }
  ],
  "model": "text-embedding-3-small",
  "usage": {
    "prompt_tokens": 8,
    "total_tokens": 8
  }
}
```

The vector values above are illustrative. The actual default embedding contains 1,536 values.

## 22. Calling OpenAI Embeddings from C#

Using the OpenAI .NET SDK:

```csharp
using OpenAI.Embeddings;

var client = new EmbeddingClient(
    model: "text-embedding-3-small",
    apiKey: Environment.GetEnvironmentVariable(
        "OPENAI_API_KEY"));

var embedding = await client.GenerateEmbeddingAsync(
    "Azure Service Bus supports asynchronous messaging.");

ReadOnlyMemory<float> vector =
    embedding.Value.ToFloats();

Console.WriteLine(vector.Length);
```

Expected dimensions:

```text
1536
```

This is an illustrative SDK pattern; use the installed package version's API signature.

## 23. Calling an LLM from C#

```csharp
using OpenAI.Chat;

var client = new ChatClient(
    model: "gpt-4.1-mini",
    apiKey: Environment.GetEnvironmentVariable(
        "OPENAI_API_KEY"));

var response = await client.CompleteChatAsync(
    "Explain dependency injection in C#.");

Console.WriteLine(
    response.Value.Content[0].Text);
```

This demonstrates the basic model invocation pattern. The model must be available to your API account.

## 24. Suggested .NET Architecture

Use Clean Architecture to keep your AI integration independent of business logic.

```text
AIDeveloperAssistant
│
├── AIDeveloperAssistant.API
│   ├── Controllers
│   ├── Middleware
│   └── Program.cs
│
├── AIDeveloperAssistant.Application
│   ├── Interfaces
│   ├── Services
│   └── UseCases
│
├── AIDeveloperAssistant.Domain
│   ├── Entities
│   └── ValueObjects
│
└── AIDeveloperAssistant.Infrastructure
    ├── OpenAI
    ├── Embeddings
    ├── VectorStore
    ├── MCP
    └── Persistence
```

Define interfaces such as:

```csharp
public interface IEmbeddingService
{
    Task<float[]> GenerateEmbeddingAsync(
        string text);
}

public interface IChatService
{
    Task<string> GenerateResponseAsync(
        string prompt);
}

public interface IVectorStore
{
    Task SaveAsync(
        string id,
        string content,
        float[] embedding);

    Task<IReadOnlyList<string>> SearchAsync(
        float[] queryEmbedding,
        int topK);
}
```

This allows you to replace providers without changing the application layer.

---

# PART 14: PRODUCTION CONSIDERATIONS

## 25. Security

Important areas:
- Entra ID authentication.
- Role-based and attribute-based authorization.
- Managed identities.
- Azure Key Vault.
- Data encryption.
- Document-level permissions.
- Prompt injection protection.
- Tool allowlisting.
- Audit logs.

Never treat the LLM as an authorization engine.

## 26. Performance and scalability

| Concern | Solution |
|---|---|
| High API latency | Streaming responses and efficient model selection |
| Expensive token usage | Reduce unnecessary context and output |
| Repeated requests | Cache safe, reusable results |
| Large document ingestion | Queue-based processing |
| Vector search performance | Appropriate indexing and metadata filters |
| Model rate limits | Bounded retries and backoff |
| Concurrent requests | Controlled concurrency and capacity planning |
| Long-running agent tasks | Durable workflow state and asynchronous processing |

## 27. Reliability

Use:
- Retry policies with exponential backoff and jitter.
- Circuit breakers where appropriate.
- Timeouts.
- Idempotency.
- Dead-letter queues.
- Fallback models where justified.
- Human approval for sensitive actions.

## 28. Monitoring and Evaluation

Track:

- Request latency.
- Token consumption.
- Model usage cost.
- Retrieval relevance.
- Answer correctness.
- Tool execution failures.
- Hallucination and grounding issues.
- User feedback.

For RAG, evaluate retrieval quality separately from answer quality.

---

# PART 15: INTERVIEW QUESTIONS AND ANSWERS

## Fundamental questions

### Q1. What is Generative AI?

Generative AI creates new content such as text, code, images, and audio using trained models. LLMs are commonly used for text generation and conversational applications.

### Q2. What is an LLM?

An LLM is a Large Language Model trained on large datasets to understand and generate language. It uses learned patterns to predict subsequent tokens and produce responses.

### Q3. What is the Transformer architecture?

Transformer is a neural network architecture that uses attention mechanisms to process relationships between tokens. It is the foundation of many modern LLMs.

### Q4. What are tokens?

Tokens are units of text processed by a language model. They can represent words, parts of words, or punctuation.

### Q5. What is hallucination?

Hallucination occurs when a model generates plausible but incorrect or unsupported information. RAG, grounding, evaluation, and validation can help reduce this risk.

## Embedding and RAG questions

### Q6. What is an embedding?

An embedding converts text into a numerical vector representing semantic information. It enables similarity search and semantic retrieval.

### Q7. What is the difference between an embedding model and an LLM?

An embedding model produces vectors for similarity comparison. An LLM generates responses, summaries, code, and other content.

### Q8. What is a vector database?

A vector database stores embeddings and supports similarity search to retrieve semantically relevant information.

### Q9. What is cosine similarity?

Cosine similarity measures the directional similarity between two vectors. It is commonly used to compare text embeddings.

### Q10. What is RAG?

RAG combines information retrieval with LLM generation. It retrieves relevant external data and provides it to the LLM as contextual input.

### Q11. Why use RAG instead of fine-tuning?

RAG is useful when an application needs frequently changing private or external knowledge. Fine-tuning is useful for adapting model behavior or specialized task performance.

### Q12. What is chunking?

Chunking splits large documents into smaller sections to improve retrieval relevance and manage context limits.

### Q13. What is hybrid search?

Hybrid search combines keyword-based search with vector-based semantic search.

### Q14. What is top-K retrieval?

It retrieves the K most relevant results from a search operation.

## Agentic AI questions

### Q15. What is Agentic AI?

Agentic AI refers to systems that use models and tools to perform multi-step tasks toward a goal.

### Q16. What is an AI agent?

An AI agent uses an LLM to interpret objectives, select tools, observe results, and continue executing within defined constraints.

### Q17. What is the difference between an AI assistant and an AI agent?

An assistant primarily responds to user requests. An agent can additionally plan and execute multi-step operations using tools.

### Q18. What are agent skills?

Skills are reusable task-specific instructions, knowledge, and workflows that help agents perform specialized activities.

### Q19. What is MCP?

MCP is an open protocol that standardizes how AI applications connect to external tools, resources, and context.

### Q20. What is the difference between MCP and function calling?

Function calling is a model interaction pattern for requesting tool execution. MCP standardizes how external tool capabilities are exposed and accessed by compatible clients.

### Q21. What is a multi-agent system?

It is a system in which multiple specialized agents collaborate or coordinate to solve a complex problem.

### Q22. What is an agent orchestrator?

An orchestrator manages agent execution, model calls, tools, state, workflow progression, and execution policies.

## Architecture and implementation questions

### Q23. How would you build an enterprise RAG application?

I would use ASP.NET Core for the backend, an embedding model for vector generation, Azure AI Search for retrieval, Blob Storage for documents, and an LLM for answer generation. I would also implement identity, document-level permissions, logging, and evaluation.

### Q24. How do you reduce hallucinations?

I would use relevant retrieval context, clear prompts, source citations, output validation, model evaluation, and explicit handling when information is unavailable.

### Q25. How do you secure an AI agent?

I would implement least-privilege authorization, approved tool allowlists, server-side validation, secure credentials, execution limits, audit logging, and human approval for sensitive operations.

### Q26. How would you scale an AI application?

I would use stateless API services, asynchronous ingestion through Service Bus, controlled concurrency, caching, efficient retrieval, and model usage limits. I would monitor latency, token consumption, and throughput.

### Q27. How do you control AI costs?

I would select an appropriately sized model, optimize prompts and context, reduce unnecessary output, cache suitable results, batch embedding operations where supported, and monitor token usage.

### Q28. How would you handle a failed model API call?

I would implement timeouts, bounded retries with exponential backoff and jitter, circuit breakers where appropriate, telemetry, and controlled fallback behavior.

### Q29. How do you evaluate a RAG system?

I would measure retrieval relevance, recall, answer correctness, grounding, citation accuracy, latency, cost, and access-control correctness using a representative evaluation dataset.

### Q30. What is your preferred AI architecture for an enterprise application?

I would use a modular architecture with ASP.NET Core, a model abstraction layer, an embedding service, a vector search service, RAG orchestration, and controlled agent tools. I would deploy using Azure services based on security, scalability, and business requirements.

---

# PART 16: PRACTICAL PROJECT FOR INTERVIEW

## Project: Agentic Developer Assistant

**Objective:** Build an internal AI assistant that helps developers understand company architecture, troubleshoot issues, and access approved development tools.

### Functional requirements

1. Upload architecture documents and technical PDFs.
2. Extract and chunk document content.
3. Generate embeddings.
4. Store vectors in Azure AI Search.
5. Implement semantic and hybrid retrieval.
6. Generate grounded answers using an LLM.
7. Create an agent for developer troubleshooting.
8. Expose approved tools through an MCP server.
9. Generate code review suggestions.
10. Record tool executions and audit information.

### Proposed architecture

```text
             Angular Frontend
                    |
                    v
            ASP.NET Core API
                    |
             Agent Orchestrator
                    |
       +------------+------------+
       |            |            |
       v            v            v
      LLM          RAG        MCP Client
       |            |            |
       |            v            v
       |       Azure AI       MCP Server
       |        Search            |
       |                       +--+------+
       |                       |         |
       |                       v         v
       |                    GitHub   Azure DevOps
       |                    Tools      Tools
       |
       v
    Response
```

### Non-functional requirements

- Authentication and authorization.
- Secure document access.
- Scalable ingestion.
- API rate limiting.
- Prompt injection protection.
- Observability.
- Cost monitoring.
- Tool execution approval.
- Reliability and error handling.

### Suggested development milestones

| Milestone | Deliverable |
|---|---|
| 1 | Basic LLM integration |
| 2 | Embedding generation |
| 3 | Vector database integration |
| 4 | RAG implementation |
| 5 | Agent tool calling |
| 6 | MCP server and client |
| 7 | Security and evaluation |
| 8 | Azure deployment and monitoring |

---

# PART 17: HOW TO PRESENT YOUR AI KNOWLEDGE IN AN INTERVIEW

## Sample Solution Architect introduction

"I have been exploring Generative AI and Agentic AI from an enterprise architecture perspective, focusing on how these technologies can improve software engineering productivity and business automation.

I understand LLM integration, embedding models, semantic search, vector databases, and RAG architectures. I have also been working through how AI agents can use tools and external systems through function calling and MCP.

From a .NET architecture perspective, I see opportunities to integrate these capabilities using ASP.NET Core, Azure AI services, Azure AI Search, Service Bus, and Azure Functions.

My focus is on designing secure, scalable, observable, and cost-effective AI solutions rather than simply integrating an LLM API."

## Key concepts to remember

| Concept | Remember |
|---|---|
| Generative AI | Creates new content |
| LLM | Understands and generates language |
| Transformer | Architecture using attention mechanisms |
| Token | Unit of text processed by a model |
| Embedding | Numerical representation of semantic information |
| Vector DB | Stores and searches embeddings |
| Cosine similarity | Measures directional vector similarity |
| RAG | Retrieves external knowledge for LLM generation |
| Chunking | Splits documents into meaningful sections |
| Fine-tuning | Adapts model behavior through additional training |
| Function calling | Allows a model to request application tool execution |
| AI Agent | Uses models and tools to pursue goals |
| Agent Skill | Reusable task-specific instructions and workflows |
| MCP | Standard protocol for AI tool and context integration |
| Ollama | Runs compatible models locally |
| LangChain | Framework for building LLM applications |
| Azure AI Search | Supports enterprise and vector retrieval |
| Guardrails | Enforce safety and business constraints |

---

# FINAL INTERVIEW PREPARATION CHECKLIST

Before your Solution Architect interview, ensure you can:

- Explain LLM architecture and Transformer fundamentals.
- Differentiate ChatGPT, Claude, Copilot, and Ollama.
- Explain embeddings with an actual example.
- Generate an embedding using the OpenAI API.
- Explain vector databases and cosine similarity.
- Design a complete RAG architecture.
- Explain RAG versus fine-tuning.
- Implement LLM integration using C#.
- Explain function calling and structured outputs.
- Design a single-agent and multi-agent workflow.
- Explain agent skills and MCP.
- Design an enterprise Azure AI architecture.
- Discuss AI security, scalability, monitoring, and cost optimization.
- Explain trade-offs and justify architectural decisions.

**Final advice:** Don't position yourself as someone who has only learned AI terminology. Position yourself as a Solution Architect who understands how to integrate AI into existing enterprise systems using established software engineering principles, distributed systems, security, and cloud architecture.


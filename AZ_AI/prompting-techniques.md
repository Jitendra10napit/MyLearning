# Different Types of Prompting Techniques in Generative AI

Prompt engineering is the process of designing effective instructions and providing context to an LLM to get accurate, relevant, and structured responses.

As a .NET developer and aspiring AI Solution Architect, you should understand these prompting techniques and know when to use each one.

Let's use practical examples from a .NET, Azure, and CCTV investigation application.

## 1. Zero-Shot Prompting

Definition: Give the LLM a task without providing any examples.

You directly ask the model to perform a task using instructions.

Example:

```
You are an experienced .NET developer.

Explain the Repository Design Pattern in C#.
Provide:
1. Definition
2. Benefits
3. Real-world use case
4. C# implementation
```

Expected result: The LLM generates an explanation based on its learned knowledge.

Use cases:

- Code explanations.
- Summarization.
- Text classification.
- General technical questions.

## 2. One-Shot Prompting

Definition: Provide one example to show the LLM the expected output format or behavior.

Example:

```
You are an API response classifier.

Example:

Input: Employee API returned HTTP 401.
Output: Authentication Error

Now classify:

Input: Employee API returned HTTP 500.
Output:
```

Expected result:

```
Internal Server Error
```

Use cases:

- Standardizing API responses.
- Generating structured outputs.
- Classification tasks.

## 3. Few-Shot Prompting

Definition: Provide multiple examples so the model understands the expected pattern.

Example:

```
Classify the following application errors.

Example 1:
Input: HTTP 401
Output: Authentication Error

Example 2:
Input: HTTP 403
Output: Authorization Error

Example 3:
Input: HTTP 404
Output: Resource Not Found

Now classify:

Input: HTTP 503
Output:
```

Expected result:

```
Service Unavailable
```

Use cases:

- Business rule classification.
- Ticket categorization.
- Sentiment analysis.
- Consistent response generation.

## 4. Role-Based Prompting (Persona Prompting)

Definition: Assign a specific role or expertise to the LLM.

Example:

```
Act as a Senior .NET Solution Architect
with 15 years of experience.

Review the following architecture:

ASP.NET Core API
Azure Functions
Azure Service Bus
SQL Server

Identify:
1. Scalability issues
2. Performance bottlenecks
3. Security risks
4. Architecture improvements
```

Expected result: The model responds from the perspective of a Solution Architect.

Use cases:

- Architecture reviews.
- Code reviews.
- Technical mentoring.
- Security assessments.

## 5. Chain-of-Thought Prompting

Definition: Encourage the model to work through a complex problem systematically before presenting its answer.

Example:

```
You are a system design expert.

Design an order processing system using
.NET 8 and Azure Service Bus.

Analyze:
1. Functional requirements
2. Non-functional requirements
3. Architecture components
4. Message processing
5. Failure handling
6. Scalability
7. Security

Provide the key design decisions and their rationale.
```

Expected result: A structured design with explanations of important decisions.

Use cases:

- System design.
- Complex problem solving.
- Architecture decision-making.
- Debugging.

Note: You can ask for a concise explanation and rationale without requiring the model to expose private internal reasoning.

## 6. Contextual Prompting

Definition: Provide relevant background information before asking a question.

Example:

```
Context:

Our application is built using:
- .NET 8 Web API
- Entity Framework Core
- SQL Server
- Azure Service Bus
- Azure Functions

We process CCTV investigation evidence.

Question:

How can we implement reliable asynchronous
processing for evidence uploads?

Consider:
- Duplicate message handling
- Retry mechanism
- Dead-letter queue
- Idempotency
```

Expected result: The response is tailored to your actual technology stack and business context.

Use cases:

- Enterprise applications.
- Domain-specific questions.
- Architecture recommendations.
- RAG applications.

## 7. Instruction-Based Prompting

Definition: Give clear, explicit instructions describing exactly what the model should do.

Example:

```
Review the following C# code.

Instructions:
- Identify performance issues.
- Check SOLID principles.
- Identify potential exceptions.
- Suggest improvements.
- Do not change the business logic.
- Return the improved code.
```

Use cases:

- Code generation.
- Refactoring.
- Documentation.
- Technical analysis.

## 8. Structured Output Prompting

Definition: Ask the LLM to return data in a predefined structure, such as JSON.

Example:

```
Analyze the following error:

"Database connection timeout"

Return the result using this JSON structure:

{
  "errorType": "",
  "severity": "",
  "possibleCause": "",
  "recommendedAction": ""
}
```

Expected result:

```
{
  "errorType": "DatabaseTimeout",
  "severity": "High",
  "possibleCause": "Database server is unavailable or overloaded",
  "recommendedAction": "Check database connectivity and query performance"
}
```

Use cases:

- .NET API integration.
- Automated workflows.
- AI agents.
- Data extraction.

For production applications, prefer schema-constrained structured output where supported instead of relying only on prompt instructions.

## 9. Constraint-Based Prompting

Definition: Specify limitations or rules that the LLM must follow.

Example:

```
Generate a C# implementation.

Constraints:
- Use .NET 8.
- Follow SOLID principles.
- Use async/await.
- Use dependency injection.
- Do not use static service classes.
- Include cancellation token support.
- Include exception handling.
- Do not introduce additional NuGet packages.
```

Use cases:

- Enterprise coding standards.
- Secure code generation.
- Technology restrictions.
- Consistent implementation.

## 10. Retrieval-Augmented Prompting (RAG Prompting)

Definition: Provide retrieved external information as context to the LLM before asking it to generate an answer.

This is especially relevant to your existing Elasticsearch indexes.

Example:

```
You are an AI assistant for a CCTV investigation platform.

User Question:
Find the investigation details for case CASE-10245.

Retrieved Context from Elasticsearch:

{
  "caseId": "CASE-10245",
  "vehicleNumber": "MP04AB1234",
  "status": "Under Review",
  "evidenceCount": 5
}

Instructions:
- Answer only using the retrieved context.
- Do not invent missing information.
- Mention the case ID in the response.
- If information is unavailable, say so.
```

Expected result:

The investigation CASE-10245 is currently Under Review. It is associated with vehicle MP04AB1234 and contains five evidence records.

Use cases:

- Enterprise knowledge assistants.
- Document search.
- CCTV investigation assistants.
- Internal knowledge bases.

## 11. Decomposition Prompting

Definition: Break a complex task into smaller manageable subtasks.

Example:

```
Design an AI-powered employee management system.

Break the task into:

1. Database design
2. API design
3. Authentication
4. Business logic
5. AI integration
6. Testing
7. Deployment

Explain each component separately.
```

Use cases:

- Large software development tasks.
- Complex architecture.
- Multi-step workflows.
- AI agent planning.

## 12. Self-Consistency Prompting

Definition: Generate multiple candidate solutions and compare them to identify a more reliable answer.

Example:

```
Evaluate three approaches for processing
10,000 messages per minute:

1. Azure Functions
2. ASP.NET Core Background Service
3. Azure Container Apps

Compare:
- Scalability
- Cost
- Reliability
- Operational complexity

Recommend the most suitable approach
and explain the trade-offs.
```

In production, you can run multiple independent generations and compare their answers using evaluation criteria or another model.

Use cases:

- Architecture decisions.
- Complex reasoning.
- Comparing implementation approaches.
- Reducing reliance on a single generated answer.

## 13. Summary: Prompting techniques comparison

| Technique         | Main purpose               | Example                           |
| ----------------- | -------------------------- | --------------------------------- |
| Zero-shot         | No examples                | Explain dependency injection      |
| One-shot          | One example                | Classify an API error             |
| Few-shot          | Multiple examples          | Categorize support tickets        |
| Role-based        | Assign expertise           | Act as Solution Architect         |
| Chain-of-thought  | Structured problem solving | Design a distributed system       |
| Contextual        | Provide background         | Explain based on our architecture |
| Instruction-based | Explicit directions        | Review C# code                    |
| Structured output | Fixed response format      | Generate JSON                     |
| Constraint-based  | Enforce restrictions       | Follow SOLID and .NET 8           |
| RAG prompting     | Use retrieved knowledge    | Search investigation records      |
| Decomposition     | Break down tasks           | Design a complete application     |
| Self-consistency  | Compare candidate answers  | Evaluate architecture options     |

## 14. Practical example: Combining prompting techniques in a .NET application

In real-world applications, we generally combine multiple prompting techniques rather than using only one.

Suppose you are building an AI-powered CCTV Investigation Assistant.

## Combined prompt example

Enterprise use case

System instruction – Role-based + Constraint-based

```
You are an AI assistant for a CCTV
investigation platform.

Follow these rules:
- Use only authorized retrieved data.
- Never invent investigation details.
- Protect sensitive information.
- Return structured JSON.
```

Retrieved context – RAG prompting

```
{
  "caseId": "CASE-10245",
  "status": "Under Review",
  "vehicleNumber": "MP04AB1234"
}
```

User request – Instruction-based

```
Find the case status and vehicle details.
Return the result in the requested JSON format.
```

Expected output – Structured prompting

```
{
  "caseId": "CASE-10245",
  "status": "Under Review",
  "vehicleNumber": "MP04AB1234"
}
```

## 15. How to use prompting in .NET with OpenAI

A simplified example using the OpenAI .NET SDK:

```
using OpenAI.Chat;

var chatClient = new ChatClient(
    model: "gpt-4.1-mini",
    apiKey: configuration["OpenAI:ApiKey"]
);

var messages = new List<ChatMessage>
{
    new SystemChatMessage("""
        You are a senior .NET Solution Architect.
        Follow SOLID principles.
        Return concise and structured responses.
        """),

    new UserChatMessage("""
        Explain dependency injection in .NET 8.
        Include a practical C# example.
        """)
};

var response = await chatClient.CompleteChatAsync(messages);

Console.WriteLine(response.Value.Content[0].Text);
```

Here:

- `SystemChatMessage` defines the role and general instructions.
- `UserChatMessage` provides the actual task.
- `CompleteChatAsync` sends the prompt to the LLM.

## 16. Interview questions to prepare

1. What is Prompt Engineering?
2. What is the difference between Zero-Shot and Few-Shot prompting?
3. What is Role-Based prompting?
4. What is Chain-of-Thought prompting?
5. What is Contextual prompting?
6. How does RAG prompting work?
7. How do you enforce JSON output from an LLM?
8. What are hallucinations, and how can prompting reduce them?
9. What is the difference between prompting and fine-tuning?
10. How do you design reusable prompts in an enterprise .NET application?

My architectural recommendation: In an enterprise .NET application, don't hardcode large prompts inside controllers. Maintain reusable prompt templates, version them, separate system instructions from user input, validate structured outputs, and evaluate prompt quality with representative test cases. For sensitive investigation data, enforce authorization before retrieval and before tool execution.

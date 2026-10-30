Function Calling is a feature that allows an AI model (LLM) to request your application to execute a specific function or method to perform a task.

In simple English:

Instead of only answering your question, AI can ask your application to perform an action and use the result to give you an answer.

### Example: .NET Employee Application

Suppose you ask:

"Get employee details for employee ID 101."

Without Function Calling:

- AI might generate a response based on its existing knowledge.
- It cannot directly access your employee database.

With Function Calling:

1. You ask AI to get employee details.
2. AI understands that it needs employee data.
3. AI requests your application to call `GetEmployeeById(101)`.
4. Your .NET application executes the method and fetches data from SQL Server.
5. The result is sent back to AI.
6. AI generates a response.

User

Get employee details for ID 101.

LLM (AI)

Decides to call `GetEmployeeById(101)`.

.NET Web API

Executes the requested method.

SQL Server

Returns employee information.

LLM Response

Employee 101 is John, working as a Software Engineer.

### Simple C# function example

```
public Employee GetEmployeeById(int employeeId)
{
    return _dbContext.Employees
        .First(e => e.Id == employeeId);
}
```

You provide the LLM with the function's name, description, and parameter schema.

The LLM can then request something like:

```
{
  "name": "GetEmployeeById",
  "arguments": {
    "employeeId": 101
  }
}
```

Your .NET application receives this request and executes the actual C# method.

### Important: Who executes the function?

The LLM does not execute your C# method directly.

- LLM decides which function should be called.
- .NET application executes the function.
- Function result goes back to the LLM.
- LLM generates the final response.

### Where can we use Function Calling?

- Get employee details from SQL Server.
- Search investigation records in Elasticsearch.
- Retrieve documents from Azure AI Search.
- Send notifications using Azure Functions.
- Create support tickets.
- Retrieve order status from an API.

### Function Calling vs Agentic AI

| Function Calling                  | Agentic AI                                   |
| --------------------------------- | -------------------------------------------- |
| Calls a specific function         | Can plan and execute multiple steps          |
| Usually handles a defined action  | Can decide which actions are needed          |
| Application executes the function | Agent orchestrates tools and actions         |
| Building block for agents         | Uses function calling and other capabilities |

Interview-ready definition:

"Function Calling allows an LLM to identify when an external function or API is required, generate the required arguments, and request the application to execute it. The application executes the function and sends the result back to the LLM to generate the final response."

Remember: Function Calling connects AI intelligence with your actual business application and data.

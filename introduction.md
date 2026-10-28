using Asp.Versioning;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace OrderService.Api.Controllers;

[ApiController]
[ApiVersion(1.0)]
[Route("api/v{version:apiVersion}/orders")]
[Authorize]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;
    private readonly ILogger<OrdersController> _logger;

    public OrdersController(
        IOrderService orderService,
        ILogger<OrdersController> logger)
    {
        _orderService = orderService;
        _logger = logger;
    }

    // POST /api/v1/orders
    [HttpPost]
    [Authorize(Policy = "Order.Create")]
    [ProducesResponseType(typeof(OrderResponse), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> CreateOrder(
        [FromBody] CreateOrderRequest request,
        CancellationToken cancellationToken)
    {
        if (!ModelState.IsValid)
            return ValidationProblem(ModelState);

        var userId = User.FindFirst("sub")?.Value;

        if (string.IsNullOrEmpty(userId))
            return Unauthorized();

        var order = await _orderService.CreateAsync(
            request,
            userId,
            cancellationToken);

        return CreatedAtAction(
            nameof(GetOrder),
            new
            {
                orderId = order.OrderId,
                version = "1"
            },
            order);
    }


    // GET /api/v1/orders/{orderId}
    [HttpGet("{orderId:guid}")]
    [Authorize(Policy = "Order.Read")]
    [ProducesResponseType(typeof(OrderResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetOrder(
        Guid orderId,
        CancellationToken cancellationToken)
    {
        var userId = User.FindFirst("sub")?.Value;

        if (string.IsNullOrEmpty(userId))
            return Unauthorized();

        var order = await _orderService.GetByIdAsync(
            orderId,
            userId,
            cancellationToken);

        if (order == null)
            return NotFound();

        return Ok(order);
    }


    // GET /api/v1/orders?pageNumber=1&pageSize=20
    [HttpGet]
    [Authorize(Policy = "Order.Read")]
    [ProducesResponseType(typeof(PagedResult<OrderResponse>), StatusCodes.Status200OK)]
    public async Task<IActionResult> GetOrders(
        [FromQuery] int pageNumber = 1,
        [FromQuery] int pageSize = 20,
        CancellationToken cancellationToken = default)
    {
        if (pageNumber <= 0)
            return BadRequest("pageNumber must be greater than 0.");

        if (pageSize <= 0 || pageSize > 100)
            return BadRequest("pageSize must be between 1 and 100.");

        var userId = User.FindFirst("sub")?.Value;

        if (string.IsNullOrEmpty(userId))
            return Unauthorized();

        var result = await _orderService.GetOrdersAsync(
            userId,
            pageNumber,
            pageSize,
            cancellationToken);

        return Ok(result);
    }


    // PATCH /api/v1/orders/{orderId}
    [HttpPatch("{orderId:guid}")]
    [Authorize(Policy = "Order.Update")]
    [ProducesResponseType(typeof(OrderResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> UpdateOrder(
        Guid orderId,
        [FromBody] UpdateOrderRequest request,
        CancellationToken cancellationToken)
    {
        if (!ModelState.IsValid)
            return ValidationProblem(ModelState);

        var userId = User.FindFirst("sub")?.Value;

        if (string.IsNullOrEmpty(userId))
            return Unauthorized();

        var order = await _orderService.UpdateAsync(
            orderId,
            request,
            userId,
            cancellationToken);

        if (order == null)
            return NotFound();

        return Ok(order);
    }


    // POST /api/v1/orders/{orderId}/cancel
    [HttpPost("{orderId:guid}/cancel")]
    [Authorize(Policy = "Order.Cancel")]
    [ProducesResponseType(typeof(OrderResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    [ProducesResponseType(StatusCodes.Status409Conflict)]
    public async Task<IActionResult> CancelOrder(
        Guid orderId,
        CancellationToken cancellationToken)
    {
        var userId = User.FindFirst("sub")?.Value;

        if (string.IsNullOrEmpty(userId))
            return Unauthorized();

        try
        {
            var order = await _orderService.CancelAsync(
                orderId,
                userId,
                cancellationToken);

            if (order == null)
                return NotFound();

            return Ok(order);
        }
        catch (OrderAlreadyCancelledException)
        {
            return Conflict("Order is already cancelled.");
        }
        catch (InvalidOrderStateException ex)
        {
            return Conflict(ex.Message);
        }
    }
}


With cte as (

    SELECT *, ROW_NUMBER() over(
        partition by name, department, salary order by id
    ) as rn

    from employee
)

DELETE FROM CTE Wehre RN>1





Sure. I’m Jitendra Napit, and I’m a Senior Software Engineering professional with over 13 years of experience in software development, technical leadership, and solution architecture.

My core expertise is in the Microsoft technology stack, particularly C#, .NET, ASP.NET Core, Web APIs, Azure, SQL Server, and modern frontend technologies like Angular and Vue.js. Over the years, I have worked extensively on enterprise applications, microservices, distributed systems, cloud-native solutions, and performance and scalability improvements.

Currently, I’m working as a Specialist Software Engineer with solution-architecture responsibilities at NICE, where I have been working on the NICE Investigate public-safety product that helps police departments so public-safety investigators 
manage cases and digital evidence — things like CCTV video, phone footage, audio, photos and documents. Think
of it as a secure digital case file, shared safely by many different police forces at once, where each
force only ever sees its own data.

What can people do with it?
• Create and manage cases.
• Collect and review evidence (video, audio, images, files).
• Ask the public and businesses to send in evidence.
• Search across huge amounts of evidence in seconds.
• Get notifications the moment something happens.
• Be sure that only the right people can open each piece of evidence.


In my current role, my responsibilities go beyond development. I work on solution design, HLD and LLD, technical decisions, API and microservice design, Azure services, code reviews, performance optimisation, production troubleshooting, and mentoring team members. I also collaborate with different stakeholders to understand requirements and convert them into scalable technical solutions.

From a leadership perspective, I enjoy solving complex technical problems, improving system architecture and performance, mentoring engineers, and making technical decisions that balance scalability, maintainability, security, and business requirements.

So overall, I would describe myself as a hands-on technical leader who can work across the complete lifecycle—from understanding the business problem and designing the architecture to implementation, deployment, and production support.


Software Architect | .NET & Azure | Microservices | Cloud-Native Architecture | Distributed Systems | Technical Leadership | AI-Driven Engineering



In my current role, I primarily have responsibilities across five areas.

**First, Solution and Architecture:**
I participate in HLD and LLD, evaluate technical approaches, define service boundaries, select appropriate Azure services, and contribute to architecture and technical decisions.

**Second, Hands-on Development:**
I continue to be hands-on with C#, .NET, ASP.NET Core, Web APIs, microservices, SQL Server and Azure. I work on both new features and complex technical problems.

**Third, Performance and Production:**
I investigate production issues using application and Azure logs, correlation IDs and monitoring tools, identify root causes, and implement solutions for performance, reliability and scalability.

**Fourth, Technical Leadership:**
I perform code reviews, mentor developers, help the team with technical challenges, and drive improvements in coding standards, design and engineering practices.

**Fifth, AI-assisted Engineering:**
I am also applying Generative AI and agent-based approaches to engineering workflows, including code analysis, PR reviews, root-cause analysis and feature development, using concepts such as AI Agents, Agent Skills and MCP.

So my role is a combination of **hands-on engineering, architecture, technical leadership, production problem-solving and mentoring**, rather than being limited to just development.




Sure. I have over 13 years of experience in software engineering, primarily working with the Microsoft technology stack, including C#, .NET, ASP.NET Core, Web APIs, SQL Server and Azure.
Currently, I'm working as a Specialist Software Engineer at NICE, where I've been involved in developing enterprise-level applications and backend services. My experience covers the complete software development lifecycle, from understanding requirements and designing solutions to development, testing, deployment and production support.
As a Solution Architect, I actively design solutions for system architecture, scalability, performance, distributed systems and cloud-based solutions. I've worked with Azure services such as Azure Functions, Service Bus, API Management and Storage, along with CI/CD and DevOps practices.
I'm also comfortable working with design patterns, clean architecture, event-driven systems and API design, and I enjoy solving complex technical problems and helping other developers when needed.

More recently, I've also been exploring AI-assisted software engineering, LLMs, RAG, AI agents and MCP, because I believe these technologies are becoming increasingly important for modern engineering teams.
At this stage of my career, I'm looking for a Senior Software Engineer opportunity where I can remain hands-on while also contributing to architecture, technical decision-making, scalability and overall engineering quality. That's one of the reasons I'm particularly interested in this opportunity with Kaseya.

1. coding
2. craft presentation 
3. Manager.

Kesya company : knowledge of bussiness



What are the things when we want to move from monolithic to microservice architecture
Large paload handling on API
how do you design a system which is highly available and scalable 
chessboard design class


sql and nosql(cosmos)
Auth and OIDC SMAL

How do we manage the service bus message queue/topic to handle the consumer failure 
how do we setup api gateway

pros and cons of cqrs
what design pattern we have use in our app

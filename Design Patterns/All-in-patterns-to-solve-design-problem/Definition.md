## 1. What is a Design Pattern?

A design pattern is a reusable solution approach to a recurring software-design problem.

It is not a library, framework, or piece of code that you blindly copy.

## For example:

##Problem:
I need to create different payment providers.

Current code:
if (provider == "Stripe")
    ...
else if (provider == "PayPal")
    ...
else if (provider == "Razorpay")
    ...

Problem:
Every new provider requires modifying existing code.

Possible solution:
Strategy Pattern

##So the architect's thinking is:

Problem
   ↓
Identify what changes
   ↓
Separate changing behavior
   ↓
Choose appropriate abstraction
   ↓
Apply pattern
   ↓
Evaluate trade-offs

Problem
   ↓
Identify what changes
   ↓
Separate changing behavior
   ↓
Choose appropriate abstraction
   ↓
Apply pattern
   ↓
Evaluate trade-offs

## 2. Major Categories of Design Patterns

The classic Gang of Four (GoF) design patterns contain 23 patterns divided into three categories.

                 DESIGN PATTERNS
                       │
          ┌────────────┼────────────┐
          │            │            │
     Creational   Structural   Behavioral
          │            │            │
      5 patterns    7 patterns   11 patterns


  Creational Patterns

Concerned with:

How objects are created

There are 5 classic patterns:

Singleton
Factory Method
Abstract Factory
Builder
Prototype
Structural Patterns

Concerned with:

How objects/classes are composed

There are 7:

Adapter
Bridge
Composite
Decorator
Facade
Flyweight
Proxy
Behavioral Patterns

Concerned with:

How objects communicate and distribute responsibility

There are 11:

Chain of Responsibility
Command
Interpreter
Iterator
Mediator
Memento
Observer
State
Strategy
Template Method
Visitor
3. But an Architect Should Know More Than GoF

In modern .NET architecture, you'll encounter additional patterns that aren't part of the original 23 GoF patterns.

For example:

Application / Enterprise Patterns
Repository
Unit of Work
Specification
Dependency Injection
Service Layer
DTO
Data Mapper
Active Record
CQRS
Domain Events
Domain Model
Distributed-System Patterns
Outbox
Inbox
Saga
Retry
Circuit Breaker
Bulkhead
Rate Limiting
Idempotency
Event Sourcing
Strangler Fig
API Gateway
Backend for Frontend
Cache-Aside
Integration / Messaging Patterns
Publisher/Subscriber
Competing Consumers
Message Router
Dead Letter Queue
Request/Reply
Event-driven architecture

These become extremely important at Architect/Staff level.

4. How We Will Learn Each Pattern

For every pattern, use this framework:

1. What problem does it solve?
2. What happens without the pattern?
3. What is the pattern?
4. What is the structure?
5. How does it work?
6. C# implementation
7. Real-world usage
8. When should I use it?
9. When should I NOT use it?
10. Advantages
11. Disadvantages
12. Architect-level considerations
13. Interview explanation

This is much more useful than memorizing definitions.    

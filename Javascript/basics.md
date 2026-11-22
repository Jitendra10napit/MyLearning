# JavaScript – Complete Interview Preparation Guide

## Part 1: JavaScript Fundamentals

### 1. Variable Declaration – var, let and const

JavaScript provides three ways to declare variables:

- `var`: Function-scoped. Can be redeclared and reassigned.
- `let`: Block-scoped. Cannot be redeclared in the same scope but can be reassigned.
- `const`: Block-scoped. Cannot be redeclared or reassigned.

**Example:**

```javascript
var a = 10;
let b = 20;
const c = 30;

a = 15;
b = 25;

// c = 35; // Error: Assignment to constant variable

console.log(`${a}, ${b}, ${c}`);
```

**Interview Comparison:**

| Feature | var | let | const |
|---|---|---|---|
| Scope | Function | Block | Block |
| Redeclaration | Yes | No | No |
| Reassignment | Yes | Yes | No |
| Hoisting | Yes | Yes (TDZ) | Yes (TDZ) |
| Default value | undefined | undefined | Must initialize |

### 2. Block Scope

Block scope means a variable is accessible only inside the `{}` block where it is declared.

- `var` does not follow block scope.
- `let` and `const` follow block scope.

**Example:**

```javascript
if (true) {
    var x = 10;
    let y = 20;
    const z = 30;
}

console.log(x); // 10
// console.log(y); // ReferenceError
// console.log(z); // ReferenceError
```

**Interview Tip:** Prefer `let` and `const` over `var` in modern JavaScript.

### 3. JavaScript Data Types

JavaScript has two main categories of data types:

**A. Primitive Data Types**

1. String
2. Number
3. Boolean
4. Undefined
5. Null
6. BigInt
7. Symbol

**B. Non-Primitive Data Types**

1. Object
2. Array
3. Function

**Example:**

```javascript
let name = "Jitendra";
let age = 40;
let isActive = true;
let salary;

let employee = {
    id: 101,
    name: "Jitendra"
};

let skills = ["C#", "JavaScript", "Angular"];

console.log(typeof name);       // string
console.log(typeof age);        // number
console.log(typeof isActive);   // boolean
console.log(typeof salary);     // undefined
console.log(typeof employee);   // object
console.log(typeof skills);     // object
console.log(typeof null);       // object
```

**Important Interview Point:**

`typeof null` returns `"object"`. This is a historical JavaScript behavior and is considered a language quirk.

Arrays are also objects in JavaScript.

To check whether a variable is an array:

```javascript
console.log(Array.isArray(skills)); // true
```

### 4. Difference Between == and ===

**== (Loose Equality)**

- Compares values after performing type conversion when necessary.
- Also called abstract equality.

**=== (Strict Equality)**

- Compares both value and data type.
- Does not perform implicit type conversion.

**Example:**

```javascript
console.log(10 == "10");  // true
console.log(10 === "10"); // false

console.log(10 == 10);    // true
console.log(10 === 10);   // true

console.log(null == undefined);  // true
console.log(null === undefined); // false
```

**Interview Tip:** Always prefer `===` unless you intentionally need type coercion.

### 5. JavaScript Operators

#### A. Arithmetic Operators

```javascript
let a = 10;
let b = 5;

console.log(a + b); // 15
console.log(a - b); // 5
console.log(a * b); // 50
console.log(a / b); // 2
console.log(a % b); // 0
```

#### B. Comparison Operators

```javascript
console.log(a > b);  // true
console.log(a < b);  // false
console.log(a >= b); // true
console.log(a <= b); // false
```

#### C. Logical Operators

```javascript
console.log(a > 5 && b < 10);  // true
console.log(a > 20 || b < 10); // true
console.log(!(a > b));         // false
```

| Operator | Meaning |
|---|---|
| `&&` | Logical AND |
| `||` | Logical OR |
| `!` | Logical NOT |

### 6. Nullish Coalescing (??) vs OR (||)

Both operators provide fallback values, but they behave differently.

**OR (||)** uses the right-hand value when the left-hand value is falsy.

Falsy values include:
- `false`
- `0`
- `""`
- `null`
- `undefined`
- `NaN`

**Nullish Coalescing (??)** uses the right-hand value only when the left-hand value is `null` or `undefined`.

**Example:**

```javascript
let a = 0;

console.log(a || 100); // 100
console.log(a ?? 100); // 0

let name = "";

console.log(name || "Guest"); // Guest
console.log(name ?? "Guest"); // ""
```

**Interview Tip:**

Use `??` when zero, false, or an empty string are valid values and should not be replaced.

### 7. Conditional Statements

#### A. If-Else

```javascript
let marks = 75;

if (marks >= 90) {
    console.log("Grade A");
}
else if (marks >= 60) {
    console.log("Grade B");
}
else {
    console.log("Grade C");
}
```

#### B. Ternary Operator

A shorter alternative to simple if-else conditions.

```javascript
let age = 20;

let result = age >= 18 ? "Adult" : "Minor";

console.log(result);
```

#### C. Switch Statement

Useful when comparing one expression against multiple possible values.

```javascript
let role = "Admin";

switch (role) {
    case "Admin":
        console.log("Full Access");
        break;

    case "User":
        console.log("Limited Access");
        break;

    default:
        console.log("No Access");
}
```

### 8. Loops in JavaScript

#### A. For Loop

```javascript
for (let i = 0; i <= 5; i++) {
    console.log(i);
}
```

#### B. While Loop

```javascript
let k = 1;

while (k <= 5) {
    console.log(k);
    k++;
}
```

#### C. For...of Loop

Used to iterate over values of iterable objects such as arrays and strings.

```javascript
let skills = ["C#", "JavaScript", "Angular"];

for (const skill of skills) {
    console.log(skill);
}
```

#### D. For...in Loop

Used to iterate over enumerable property keys of an object.

```javascript
let employee = {
    id: 101,
    name: "Jitendra",
    role: "Developer"
};

for (const key in employee) {
    console.log(key, employee[key]);
}
```

**Interview Comparison:**

| Loop | Purpose |
|---|---|
| for | General iteration |
| while | Executes while condition is true |
| for...of | Iterates over values |
| for...in | Iterates over property keys |
| forEach() | Executes a callback for each array element |

### 9. Functions in JavaScript

Functions are reusable blocks of code that perform a particular task.

#### A. Normal Function Declaration

```javascript
function add(a, b) {
    return a + b;
}

console.log(add(10, 20)); // 30
```

#### B. Function Expression

A function assigned to a variable.

```javascript
const addc = function(a, b) {
    return a + b;
};

console.log(addc(10, 10)); // 20
```

#### C. Arrow Function

A shorter syntax introduced in ES6.

```javascript
const addA = (a, b) => a + b;

console.log(addA(10, 40)); // 50
```

**Important Differences:**

| Feature | Normal Function | Arrow Function |
|---|---|---|
| Syntax | function keyword | `=>` |
| Own this | Yes | No |
| Own arguments | Yes | No |
| Constructor | Yes | No |
| Hoisting | Function declarations are hoisted | Depends on variable declaration |

**Interview Tip:** Arrow functions do not have their own `this` or `arguments` object. They inherit `this` from their surrounding lexical scope.

### 10. Arrays and Important Array Methods

An array stores multiple values in a single variable.

```javascript
let numbers = [10, 20, 30, 40, 50];

console.log(numbers[0]); // 10
```

#### A. Basic Array Methods

```javascript
let numbers = [10, 20, 30, 40, 50];

numbers.push(60);    // Add at end
numbers.pop();       // Remove from end
numbers.unshift(5);  // Add at beginning
numbers.shift();     // Remove from beginning

console.log(numbers);
```

#### B. map() – Transform Each Element

Creates a new array by transforming each element.

```javascript
let numbers = [10, 20, 30, 40, 50];

let result = numbers.map(x => x / 10);

console.log(result);

// [1, 2, 3, 4, 5]
```

**Use Case:** Convert API response data into a UI-friendly format.

#### C. filter() – Select Matching Elements

Returns a new array containing elements that satisfy a condition.

```javascript
let numbers = [1, 2, 3, 4, 5, 6];

let result = numbers.filter(x => x % 2 === 0);

console.log(result);

// [2, 4, 6]
```

**Use Case:** Filter employees based on department or status.

#### D. reduce() – Calculate a Single Result

Reduces an array into a single accumulated value.

```javascript
let numbers = [10, 20, 30, 40, 50];

let total = numbers.reduce(
    (sum, current) => sum + current,
    0
);

console.log(total); // 150
```

**Real-World Example: Calculate Shopping Cart Total**

```javascript
const cart = [
    { item: "Coffee", price: 150 },
    { item: "Sandwich", price: 220 },
    { item: "Cookie", price: 150 }
];

let total = cart.reduce(
    (sum, current) => sum + current.price,
    0
);

console.log(total); // 520
```

#### E. find() – Return First Matching Element

Returns the first element that satisfies the condition.

```javascript
let numbers = [10, 20, 30, 40, 50];

let result = numbers.find(x => x > 30);

console.log(result); // 40
```

#### F. some() – Check Whether Any Element Matches

Returns true if at least one element satisfies the condition.

```javascript
let numbers = [2, 4, 6, 8, 10];

console.log(numbers.some(x => x > 5)); // true
```

#### G. every() – Check Whether All Elements Match

Returns true only if every element satisfies the condition.

```javascript
let numbers = [2, 4, 6, 8, 10];

console.log(numbers.every(x => x % 2 === 0)); // true
```

**Important Array Methods Comparison:**

| Method | Purpose | Return Value |
|---|---|---|
| map() | Transform elements | New array |
| filter() | Select elements | New array |
| reduce() | Accumulate values | Single result |
| find() | First matching element | Element or undefined |
| some() | Check at least one match | Boolean |
| every() | Check all elements | Boolean |
| forEach() | Execute callback | undefined |

### 11. JavaScript Objects

Objects store data in key-value pairs.

**Example:**

```javascript
const employeeData = {
    id: 101,
    name: "Jitendra",
    salary: 500000,

    display: function() {
        console.log(this.name);
    }
};

console.log(employeeData.name);
console.log(employeeData["salary"]);

employeeData.display();
```

#### Object Destructuring

Destructuring extracts properties from objects into variables.

```javascript
const { id, name, salary } = employeeData;

console.log(name); // Jitendra
```

### 12. Spread and Rest Operators

Both use the same `...` syntax, but their purpose depends on where they are used.

#### A. Spread Operator

Expands elements of an array or properties of an object.

**Object Example:**

```javascript
const emp = {
    id: 101,
    name: "Jitendra"
};

const updateEmp = {
    ...emp,
    city: "Pune"
};

console.log(updateEmp);
```

**Array Example:**

```javascript
const first = [10, 20];
const second = [30, 40];

const combined = [...first, ...second];

console.log(combined);

// [10, 20, 30, 40]
```

#### B. Rest Operator

Collects multiple arguments into a single array.

```javascript
function sum(...numbers) {
    return numbers.reduce((a, b) => a + b, 0);
}

console.log(sum(10, 20, 30, 40)); // 100
```

**Interview Tip:**

- Spread: Expands values.
- Rest: Collects values.

---

# Part 2: Intermediate JavaScript

### 13. Scope in JavaScript

Scope determines where a variable can be accessed.

There are three commonly discussed scopes:

1. Global Scope
2. Function Scope
3. Block Scope

**Example:**

```javascript
let globalValue = 100;

function test() {
    let functionValue = 200;

    if (true) {
        let blockValue = 300;

        console.log(globalValue);   // 100
        console.log(functionValue); // 200
        console.log(blockValue);    // 300
    }

    // console.log(blockValue); // ReferenceError
}

test();
```

#### Lexical Scope

JavaScript uses lexical scope. This means an inner function can access variables from its surrounding scope.

```javascript
function outer() {
    let message = "Hello from outer function";

    function inner() {
        console.log(message);
    }

    inner();
}

outer();
```

### 14. Hoisting

Hoisting means JavaScript processes declarations before executing code within their scope.

#### A. var Hoisting

```javascript
console.log(a); // undefined

var a = 10;
```

JavaScript behaves conceptually as:

```javascript
var a;

console.log(a);

a = 10;
```

#### B. let and const Hoisting

`let` and `const` are also hoisted, but they remain inaccessible before initialization because of the Temporal Dead Zone (TDZ).

```javascript
// console.log(x); // ReferenceError

let x = 10;
```

#### C. Function Hoisting

Function declarations can be called before their declaration.

```javascript
sayHello();

function sayHello() {
    console.log("Hello!");
}

// Output: Hello!
```

**Interview Comparison:**

| Declaration | Hoisted | Accessible Before Initialization |
|---|---|---|
| var | Yes | Yes, returns undefined |
| let | Yes | No, ReferenceError |
| const | Yes | No, ReferenceError |
| Function declaration | Yes | Yes |

### 15. Callback Functions

A callback is a function passed as an argument to another function and executed by that function.

**Example:**

```javascript
function processUser(name, callback) {
    console.log(`Processing ${name}`);

    callback();
}

function completed() {
    console.log("Processing Completed!");
}

processUser("Jitendra", completed);
```

**Output:**

```text
Processing Jitendra
Processing Completed!
```

**Real-World Use Cases:**

- Event handling
- API calls
- Timers
- Array methods such as map(), filter() and reduce()

### 16. Synchronous and Asynchronous JavaScript

#### A. Synchronous Execution

Code executes one statement at a time. The next statement waits until the current statement finishes.

```javascript
console.log("First");
console.log("Second");
console.log("Third");
```

**Output:**

```text
First
Second
Third
```

#### B. Asynchronous Execution

Asynchronous operations allow JavaScript to continue executing other code while waiting for an operation to complete.

```javascript
console.log("First");

setTimeout(() => {
    console.log("Second");
}, 3000);

console.log("Third");
```

**Output:**

```text
First
Third
Second
```

**Interview Tip:** JavaScript uses the event loop, task queues and microtask queues to coordinate asynchronous operations.

### 17. DOM – Document Object Model

DOM represents an HTML document as a tree of objects.

JavaScript uses the DOM to access, modify, add and remove HTML elements.

**Example:**

HTML:

```html
<h1 id="heading">Hello</h1>
<button id="btn">Click Me</button>
```

JavaScript:

```javascript
const heading = document.getElementById("heading");

heading.textContent = "Welcome to JavaScript";

document.getElementById("btn").addEventListener("click", function() {
    console.log("Button clicked!");
});
```

**Common DOM Methods:**

| Method | Purpose |
|---|---|
| getElementById() | Select element by ID |
| getElementsByClassName() | Select elements by class |
| getElementsByTagName() | Select elements by tag |
| querySelector() | Select first matching element |
| querySelectorAll() | Select all matching elements |
| createElement() | Create HTML element |
| appendChild() | Add child element |
| remove() | Remove element |

### 18. Event Bubbling and Event Capturing

JavaScript events propagate through the DOM in three phases:

1. Capturing Phase – Event travels from outer elements towards the target.
2. Target Phase – Event reaches the target element.
3. Bubbling Phase – Event travels from the target towards outer elements.

#### A. Event Bubbling

Event moves from the target element to its parent elements.

#### B. Event Capturing

Event moves from the outer parent elements towards the target.

**Example:**

```javascript
parent.addEventListener("click", () => {
    console.log("Parent clicked");
});

child.addEventListener("click", () => {
    console.log("Child clicked");
});
```

By default, event listeners use bubbling.

To enable capturing:

```javascript
parent.addEventListener("click", () => {
    console.log("Parent capturing");
}, true);
```

#### C. stopPropagation()

Stops the event from propagating further through the DOM.

**Real-World Example:**

A clickable product card opens a product details page, but clicking the favorite button inside it should not open the product page.

```javascript
favoriteButton.addEventListener("click", (event) => {
    event.stopPropagation();

    console.log("Added to favorites");
});
```

#### D. Event Delegation

Event delegation means attaching one event listener to a parent instead of attaching separate listeners to multiple child elements.

It uses event bubbling.

**Real-World Example: Dynamic To-Do List**

HTML:

```html
<ul id="todo-list">
    <li>Buy groceries</li>
    <li>Walk the dog</li>
    <li>Read a book</li>
</ul>
```

JavaScript:

```javascript
const list = document.getElementById("todo-list");

list.addEventListener("click", (event) => {

    if (event.target.matches("li")) {
        event.target.style.textDecoration = "line-through";

        console.log("Completed:", event.target.textContent);
    }

});
```

**Benefits of Event Delegation:**

- Reduces the number of event listeners.
- Improves memory efficiency.
- Works with dynamically added elements.
- Makes event management easier.

---

# Part 3: Advanced JavaScript – Most Important Interview Topics

The following topics should be covered next for senior-level JavaScript interviews.

### 19. Closures
- What is a closure?
- Lexical environment.
- Practical use cases.
- Data privacy using closures.

### 20. Promises
- Promise states.
- Pending, Fulfilled and Rejected.
- then(), catch() and finally().
- Promise chaining.
- Promise.all(), Promise.race(), Promise.allSettled() and Promise.any().

### 21. Async/Await
- What is async/await?
- Difference between Promise and async/await.
- Error handling using try/catch.
- Parallel vs sequential execution.
- Promise.all() with async/await.

### 22. Event Loop
- Call Stack.
- Web APIs.
- Callback Queue (Task Queue).
- Microtask Queue.
- Event Loop execution order.
- setTimeout() vs Promise.

### 23. this Keyword
- this in global scope.
- this inside normal functions.
- this inside arrow functions.
- this inside objects.
- call(), apply() and bind().

### 24. Prototypes and Inheritance
- Prototype chain.
- Prototype inheritance.
- Object.create().
- Constructor functions.
- ES6 classes and inheritance.
- extends and super keywords.

### 25. Advanced Array and Object Concepts
- Shallow copy vs deep copy.
- Object.assign().
- structuredClone().
- Object.keys(), Object.values() and Object.entries().
- Array sorting and chaining.
- Immutable operations.

### 26. Memory Management
- Stack vs Heap.
- Garbage Collection.
- Memory leaks.
- WeakMap and WeakSet.
- Common memory leak scenarios.

### 27. JavaScript Error Handling
- try, catch and finally.
- throw keyword.
- Custom error classes.
- Promise error handling.
- Global error handling.

### 28. ES6+ Features
- Template literals.
- Destructuring.
- Default parameters.
- Spread and rest operators.
- Optional chaining (?.).
- Nullish coalescing (??).
- Modules (import/export).
- let and const.

### 29. JavaScript Performance
- Debouncing.
- Throttling.
- Lazy loading.
- Code splitting.
- Memoization.
- Web Workers.
- Avoiding unnecessary DOM updates.

### 30. Browser Storage and Security
- LocalStorage.
- SessionStorage.
- Cookies.
- IndexedDB.
- XSS.
- CSRF.
- Secure token handling.

---

# Part 4: Important JavaScript Interview Preparation Strategy

For a senior .NET developer preparing for full-stack interviews, focus on understanding JavaScript execution, asynchronous programming, array methods, closures, event loop and browser fundamentals.

| Priority | Topic | Importance |
|---|---|---|
| 1 | var, let, const and scope | High |
| 2 | == vs === | High |
| 3 | map, filter and reduce | High |
| 4 | Closures | Very High |
| 5 | Promise and Async/Await | Very High |
| 6 | Event Loop | Very High |
| 7 | this, call, apply and bind | Very High |
| 8 | Event Bubbling and Delegation | High |
| 9 | Prototype and Inheritance | High |
| 10 | Debouncing and Throttling | High |
| 11 | Shallow Copy and Deep Copy | High |
| 12 | DOM Manipulation | High |

**Important:** Do not just memorize definitions. For each topic, understand:

1. What is it?
2. Why do we need it?
3. How does it work internally?
4. What is the real-world use case?
5. What is the difference between similar concepts?
6. Can you write a practical coding example?

This approach will help you explain JavaScript concepts confidently in technical interviews.

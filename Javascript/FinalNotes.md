# JavaScript Interview Preparation – Complete Guide (Basics to Advanced)

## Table of Contents

1. JavaScript Variable Declaration
2. Block Scope
3. JavaScript Data Types
4. == vs ===
5. JavaScript Operators
6. Nullish Coalescing vs OR
7. Conditional Statements
8. Loops
9. Functions
10. Arrays and Array Methods
11. JavaScript Objects
12. Spread and Rest Operators
13. Scope and Lexical Scope
14. Hoisting and Temporal Dead Zone
15. Callback Functions
16. Synchronous vs Asynchronous JavaScript
17. DOM Manipulation
18. Event Bubbling and Capturing
19. Closures
20. this Keyword
21. call(), apply() and bind()
22. Prototype and Prototypal Inheritance
23. Classes and Inheritance
24. Promises
25. Async/Await
26. Promise.all() vs allSettled() vs race() vs any()
27. JavaScript Event Loop
28. Shallow Copy vs Deep Copy
29. Debouncing
30. Throttling
31. Higher-Order Functions
32. Memoization
33. Map and Set
34. Optional Chaining and Nullish Coalescing
35. ES6 Modules
36. Error Handling
37. Event Delegation
38. Generator Functions
39. JavaScript Memory Management
40. Lazy Loading
41. Web Workers
42. Microtasks vs Macrotasks
43. Object.freeze() vs Object.seal()
44. preventDefault() vs stopPropagation()

---

# Part 1: JavaScript Fundamentals

## 1. Variable Declaration – var, let and const

JavaScript provides three keywords for declaring variables.

- `var`: Function-scoped variable.
- `let`: Block-scoped variable that can be reassigned.
- `const`: Block-scoped variable that cannot be reassigned.

**Example:**

```javascript
var a = 10;
let b = 20;
const c = 30;

a = 15;
b = 25;

// c = 35; // TypeError

console.log(`${a}, ${b}, ${c}`);
```

**Comparison:**

| Feature | var | let | const |
|---|---|---|---|
| Scope | Function | Block | Block |
| Redeclaration | Allowed | Not allowed in same scope | Not allowed in same scope |
| Reassignment | Allowed | Allowed | Not allowed |
| Hoisting | Yes | Yes | Yes |
| Initial value before assignment | undefined | TDZ | TDZ |

**Important Interview Points:**

- Prefer `const` by default.
- Use `let` when reassignment is required.
- Avoid `var` in modern JavaScript.
- `const` prevents reassignment, not modification of an object's properties.

## 2. Block Scope

Block scope means variables are accessible only within the block `{}` where they are declared.

`let` and `const` follow block scope, whereas `var` does not.

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

**Real-world use case:**

Block scope helps prevent accidental variable modification and naming conflicts inside loops, conditions and other code blocks.

## 3. JavaScript Data Types

JavaScript data types are divided into two categories.

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

console.log(Array.isArray(skills)); // true
```

**Important Interview Points:**

- JavaScript is dynamically typed.
- `typeof null` returns `"object"` due to historical language behavior.
- Arrays are objects.
- Use `Array.isArray()` to identify arrays.
- Primitive values are immutable.

## 4. Difference Between == and ===

**Loose Equality (==)**

Compares values after performing type conversion when necessary.

**Strict Equality (===)**

Compares both value and type without performing implicit type conversion.

**Example:**

```javascript
console.log(10 == "10");  // true
console.log(10 === "10"); // false

console.log(10 == 10);    // true
console.log(10 === 10);   // true

console.log(null == undefined);  // true
console.log(null === undefined); // false

console.log(false == 0);  // true
console.log(false === 0); // false
```

**Comparison:**

| Scenario | == | === |
|---|---|---|
| 10 and "10" | true | false |
| 10 and 10 | true | true |
| null and undefined | true | false |
| false and 0 | true | false |

**Interview Tip:** Prefer strict equality (`===`) to avoid unexpected type coercion.

## 5. JavaScript Operators

### Arithmetic Operators

```javascript
let a = 10;
let b = 5;

console.log(a + b); // 15
console.log(a - b); // 5
console.log(a * b); // 50
console.log(a / b); // 2
console.log(a % b); // 0
```

### Comparison Operators

```javascript
console.log(a > b);  // true
console.log(a < b);  // false
console.log(a >= b); // true
console.log(a <= b); // false
```

### Logical Operators

```javascript
console.log(a > 5 && b < 10);  // true
console.log(a > 20 || b < 10); // true
console.log(!(a > b));         // false
```

| Operator | Meaning |
|---|---|
| `&&` | AND |
| `||` | OR |
| `!` | NOT |

## 6. Nullish Coalescing (??) vs OR (||)

Both operators provide fallback values, but their conditions are different.

**OR (||)** returns the right-hand operand when the left-hand operand is falsy.

Falsy values include:

- false
- 0
- ""
- null
- undefined
- NaN

**Nullish Coalescing (??)** returns the right-hand operand only when the left-hand operand is `null` or `undefined`.

**Example:**

```javascript
let a = 0;

console.log(a || 100); // 100
console.log(a ?? 100); // 0

let name = "";

console.log(name || "Guest"); // Guest
console.log(name ?? "Guest"); // ""
```

**Comparison:**

| Value | || 100 | ?? 100 |
|---|---|---|
| 0 | 100 | 0 |
| false | 100 | false |
| "" | 100 | "" |
| null | 100 | 100 |
| undefined | 100 | 100 |

**Real-world use case:** When receiving API data, a value of `0` or `false` may be valid. Nullish coalescing prevents these values from being unnecessarily replaced.

## 7. Conditional Statements

Conditional statements execute different blocks of code depending on a condition.

### If-Else

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

### Ternary Operator

A shorter alternative to simple if-else statements.

```javascript
let age = 20;

let result = age >= 18 ? "Adult" : "Minor";

console.log(result);
```

### Switch Statement

Useful when checking one expression against multiple possible values.

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

## 8. Loops in JavaScript

Loops execute a block of code repeatedly.

### For Loop

```javascript
for (let i = 0; i <= 5; i++) {
    console.log(i);
}
```

### While Loop

```javascript
let k = 1;

while (k <= 5) {
    console.log(k);
    k++;
}
```

### For...of Loop

Iterates over values of iterable objects such as arrays and strings.

```javascript
let skills = ["C#", "JavaScript", "Angular"];

for (const skill of skills) {
    console.log(skill);
}
```

### For...in Loop

Iterates over enumerable property keys of an object.

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

**Comparison:**

| Loop | Purpose |
|---|---|
| for | General iteration |
| while | Executes while a condition is true |
| for...of | Iterates over values |
| for...in | Iterates over property keys |
| forEach() | Executes a callback for each array element |

## 9. Functions in JavaScript

Functions are reusable blocks of code designed to perform specific tasks.

### A. Function Declaration

```javascript
function add(a, b) {
    return a + b;
}

console.log(add(10, 20)); // 30
```

### B. Function Expression

A function assigned to a variable.

```javascript
const addc = function(a, b) {
    return a + b;
};

console.log(addc(10, 10)); // 20
```

### C. Arrow Function

A shorter function syntax introduced in ES6.

```javascript
const addA = (a, b) => a + b;

console.log(addA(10, 40)); // 50
```

**Important Differences:**

| Feature | Normal Function | Arrow Function |
|---|---|---|
| Own this | Yes | No |
| Own arguments | Yes | No |
| Constructor | Yes, if constructible | No |
| Syntax | function keyword | Arrow syntax |
| this binding | Depends on invocation | Lexical |

**Interview Tip:** Arrow functions inherit `this` from their surrounding lexical scope. They are useful for callbacks but are not suitable as object methods when dynamic `this` binding is required.

## 10. Arrays and Important Array Methods

An array stores multiple values in a single variable.

```javascript
let numbers = [10, 20, 30, 40, 50];

console.log(numbers[0]); // 10
```

### Basic Array Methods

```javascript
let numbers = [10, 20, 30, 40, 50];

numbers.push(60);    // Add at the end
numbers.pop();       // Remove from the end
numbers.unshift(5);  // Add at the beginning
numbers.shift();     // Remove from the beginning

console.log(numbers);
```

### map() – Transform Elements

Returns a new array after applying a transformation to each element.

```javascript
let numbers = [10, 20, 30, 40, 50];

let result = numbers.map(x => x / 10);

console.log(result);
// [1, 2, 3, 4, 5]
```

**Use case:** Transform API response data into a format suitable for displaying in an Angular or JavaScript UI.

### filter() – Select Matching Elements

Returns a new array containing elements that satisfy a condition.

```javascript
let numbers = [1, 2, 3, 4, 5, 6];

let result = numbers.filter(x => x % 2 === 0);

console.log(result);
// [2, 4, 6]
```

**Use case:** Filter employees by department, status or salary.

### reduce() – Calculate a Single Result

Reduces an array into a single accumulated value.

```javascript
let numbers = [10, 20, 30, 40, 50];

let total = numbers.reduce(
    (sum, current) => sum + current,
    0
);

console.log(total); // 150
```

**Real-world example: Calculate Shopping Cart Total**

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

### find() – Return First Matching Element

```javascript
let numbers = [10, 20, 30, 40, 50];

let result = numbers.find(x => x > 30);

console.log(result); // 40
```

Returns the first matching element or `undefined`.

### some() – Check Whether Any Element Matches

```javascript
let numbers = [2, 4, 6, 8, 10];

console.log(numbers.some(x => x > 5)); // true
```

Returns `true` if at least one element satisfies the condition.

### every() – Check Whether All Elements Match

```javascript
let numbers = [2, 4, 6, 8, 10];

console.log(numbers.every(x => x % 2 === 0)); // true
```

Returns `true` only if every element satisfies the condition.

**Array Methods Comparison:**

| Method | Purpose | Return Value |
|---|---|---|
| map() | Transform elements | New array |
| filter() | Select elements | New array |
| reduce() | Accumulate values | Single result |
| find() | First matching element | Element or undefined |
| some() | At least one match | Boolean |
| every() | All elements match | Boolean |
| forEach() | Execute callback | undefined |

## 11. JavaScript Objects and Destructuring

Objects store data in key-value pairs.

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

### Object Destructuring

Destructuring extracts object properties into individual variables.

```javascript
const { id, name, salary } = employeeData;

console.log(name); // Jitendra
```

**Use case:** Extract properties from API responses without repeatedly accessing the object.

## 12. Spread and Rest Operators

Both use the `...` syntax, but their purpose depends on where they are used.

### Spread Operator

Expands the elements of an array or properties of an object.

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

### Rest Operator

Collects multiple arguments into a single array.

```javascript
function sum(...numbers) {
    return numbers.reduce((a, b) => a + b, 0);
}

console.log(sum(10, 20, 30, 40)); // 100
```

**Interview Tip:**

- Spread expands values.
- Rest collects values.
- Object and array spread create shallow copies.

---

# Part 2: Intermediate JavaScript

## 13. Scope and Lexical Scope

Scope determines where a variable can be accessed.

JavaScript commonly uses three types of scope:

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

### Lexical Scope

Lexical scope means an inner function can access variables declared in its surrounding scope.

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

**Important:** Lexical scope is determined by where a function is defined, not where it is called.

## 14. Hoisting and Temporal Dead Zone

Hoisting refers to how JavaScript processes declarations before executing statements in their scope.

### var Hoisting

```javascript
console.log(a); // undefined

var a = 10;
```

Conceptually, JavaScript processes the declaration before execution:

```javascript
var a;

console.log(a);

a = 10;
```

### let and const Hoisting

`let` and `const` are also hoisted, but they cannot be accessed before initialization.

This period is called the Temporal Dead Zone (TDZ).

```javascript
// console.log(x); // ReferenceError

let x = 10;
```

### Function Hoisting

Function declarations can be called before their declaration.

```javascript
sayHello();

function sayHello() {
    console.log("Hello!");
}
```

**Output:**

```text
Hello!
```

**Comparison:**

| Declaration | Hoisted | Accessible before initialization |
|---|---|---|
| var | Yes | Yes, returns undefined |
| let | Yes | No, ReferenceError |
| const | Yes | No, ReferenceError |
| Function declaration | Yes | Yes |

**Interview Tip:** Function expressions and arrow functions follow the hoisting behavior of the variable used to store them.

## 15. Callback Functions

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

**Real-world use cases:**

- Event handling
- API operations
- Timer operations
- Array methods such as map(), filter() and reduce()

## 16. Synchronous vs Asynchronous JavaScript

### Synchronous Execution

Statements execute sequentially. Each operation completes before the next statement executes.

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

### Asynchronous Execution

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

**Interview Tip:** JavaScript executes application code on a main thread in typical browser environments. The event loop coordinates asynchronous callbacks and tasks. Some work is handled by browser APIs or other runtime facilities.

## 17. DOM – Document Object Model

The Document Object Model (DOM) represents an HTML document as a tree of objects.

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
| createElement() | Create an HTML element |
| appendChild() | Add a child element |
| remove() | Remove an element |

## 18. Event Bubbling and Event Capturing

DOM events propagate through three phases:

1. Capturing Phase – Event travels from outer elements towards the target.
2. Target Phase – Event reaches the target element.
3. Bubbling Phase – Event travels from the target towards outer elements.

### Event Bubbling

Events move from the target element towards its parent elements.

### Event Capturing

Events move from the outer parent elements towards the target element.

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

### stopPropagation()

Stops an event from propagating further through the DOM.

**Real-world example:** A clickable product card opens the product details page, but clicking the favorite button inside the card should not trigger the parent card's click event.

```javascript
favoriteButton.addEventListener("click", (event) => {
    event.stopPropagation();

    console.log("Added to favorites");
});
```

---

# Part 3: Advanced JavaScript

## 19. Closures

A closure occurs when an inner function retains access to variables from its outer function, even after the outer function has finished executing.

**Example:**

```javascript
function outer() {
    let count = 0;

    function inner() {
        count++;
        console.log(count);
    }

    return inner;
}

const counter = outer();

counter(); // 1
counter(); // 2
counter(); // 3
```

**How does it work?**

1. The outer function creates the `count` variable.
2. The inner function references `count`.
3. The outer function returns the inner function.
4. The returned function retains access to its lexical environment.
5. Each call updates the same `count` variable.

### Real-world Example: Bank Account

Closures can help encapsulate private data.

```javascript
function createBankAccount(initialBalance) {
    let balance = initialBalance;

    return {
        deposit(amount) {
            balance += amount;
            return balance;
        },

        getBalance() {
            return balance;
        }
    };
}

const account = createBankAccount(1000);

console.log(account.deposit(500)); // 1500
console.log(account.getBalance()); // 1500
```

**Use cases:**

- Data privacy
- Function factories
- Counters
- Maintaining state
- Memoization
- Event handlers

**Interview Tip:** Closures retain access to their lexical environment, which can keep referenced data alive longer than expected.

## 20. this Keyword

The `this` keyword refers to the execution context of a function. Its value depends on how the function is invoked.

### A. this Inside an Object Method

```javascript
const employee = {
    name: "Jitendra",

    display() {
        console.log(this.name);
    }
};

employee.display();
```

**Output:**

```text
Jitendra
```

Here, `this` refers to the employee object.

### B. this Inside a Normal Function

```javascript
function display() {
    console.log(this);
}

display();
```

- In non-strict browser scripts, `this` generally refers to the global object.
- In strict mode, `this` is `undefined`.
- In JavaScript modules, strict mode applies automatically.

### C. this Inside an Arrow Function

Arrow functions do not have their own `this`. They inherit it from the surrounding lexical scope.

```javascript
const emp1 = {
    name: "Jitendra",

    display: () => {
        console.log(this.name);
    }
};

emp1.display();
```

This does not behave like a normal object method. The arrow function does not bind `this` to `emp1`.

**Interview Tip:** Use normal methods when you need `this` to refer to the object invoking the method.

## 21. call(), apply() and bind()

These methods allow a function to execute with an explicitly specified `this` value.

**Example:**

```javascript
const emp1 = {
    name: "Jitendra"
};

const emp2 = {
    name: "Rahul"
};

function greet(city, country) {
    console.log(`${this.name} ${city} ${country}`);
}
```

### call()

Invokes a function immediately and passes arguments individually.

```javascript
greet.call(emp1, "Pune", "India");
```

### apply()

Invokes a function immediately and passes arguments as an array.

```javascript
greet.apply(emp2, ["Bhopal", "India"]);
```

### bind()

Returns a new function with a specified `this` value and optionally preconfigured arguments.

```javascript
const greetEmployee = greet.bind(emp1, "Pune", "India");

greetEmployee();
```

**Comparison:**

| Method | Execution | Arguments |
|---|---|---|
| call() | Immediate | Individually |
| apply() | Immediate | Array |
| bind() | Returns new function | Individually |

## 22. Prototype and Prototypal Inheritance

JavaScript uses prototype-based inheritance.

An object can inherit properties and methods from another object through its prototype chain.

### A. Object.create()

```javascript
const person = {
    greet() {
        console.log("Hello");
    }
};

const emp1 = Object.create(person);

emp1.name = "Jitendra";

emp1.greet();
```

The `emp1` object inherits the `greet()` method from `person`.

### B. Constructor Function

```javascript
function Employee(name) {
    this.name = name;
}

Employee.prototype.display = function() {
    console.log(this.name);
};

const e1 = new Employee("Jitendra");
const e2 = new Employee("Rahul");

e1.display();
e2.display();
```

Both objects share the same `display()` method through the prototype.

**Why is this useful?**

Instead of creating duplicate methods for every object, JavaScript allows methods to be shared through prototypes.

**Interview Points:**

- Every ordinary object has an internal prototype reference.
- Prototype chains enable inheritance.
- `Object.create()` creates an object with a specified prototype.
- `Object.getPrototypeOf()` returns an object's prototype.

## 23. Classes and Inheritance

ES6 introduced class syntax to make object-oriented programming easier to write and understand.

JavaScript classes use prototype-based inheritance internally.

### A. Class Example

```javascript
class Employee {
    constructor(name) {
        this.name = name;
    }

    display() {
        console.log(this.name);
    }
}

const emp = new Employee("Jitendra");

emp.display();
```

### B. Inheritance Using extends and super

```javascript
class Employee {
    constructor(name) {
        this.name = name;
    }

    display() {
        console.log(this.name);
    }
}

class Developer extends Employee {
    constructor(name, technology) {
        super(name);

        this.technology = technology;
    }

    showTechnology() {
        console.log(this.technology);
    }
}

const dev = new Developer("Jitendra", "Dotnet");

dev.display();
dev.showTechnology();
```

- `extends` establishes inheritance.
- `super()` invokes the parent constructor.
- `super.method()` can invoke a parent class method.

### C. Static Methods

Static methods belong to the class itself rather than individual instances.

```javascript
class MathHelper {
    static add(a, b) {
        return a + b;
    }
}

console.log(MathHelper.add(10, 20)); // 30
```

**Interview Tip:** Static methods are useful for utility operations that do not require instance-specific data.

## 24. Promises

A Promise represents the eventual completion or failure of an asynchronous operation.

A Promise has three states:

1. Pending – The operation is still running.
2. Fulfilled – The operation completed successfully.
3. Rejected – The operation failed.

Once fulfilled or rejected, a Promise is settled and cannot change its state again.

**Example:**

```javascript
const promise = new Promise((resolve, reject) => {

    let success = true;

    if (success) {
        resolve("Data fetched successfully");
    }
    else {
        reject("Something went wrong");
    }

});

promise
    .then(result => console.log(result))
    .catch(error => console.log(error))
    .finally(() => console.log("Completed"));
```

### Important Promise Methods

| Method | Purpose |
|---|---|
| then() | Handles fulfillment |
| catch() | Handles rejection |
| finally() | Executes after settlement |
| Promise.all() | Waits for all promises; rejects if one rejects |
| Promise.allSettled() | Waits for every promise to settle |
| Promise.race() | First settled promise determines the result |
| Promise.any() | First fulfilled promise determines the result |

**Real-world use cases:**

- Fetching API data
- Database operations through asynchronous services
- File operations
- Background processing
- Multiple service calls

## 25. Async/Await

Async/await provides a cleaner syntax for writing asynchronous JavaScript code.

It is built on top of Promises.

### Without Async/Await

```javascript
fetch("https://jsonplaceholder.typicode.com/users")
    .then(response => response.json())
    .then(data => console.log(data))
    .catch(error => console.log(error));
```

### With Async/Await

```javascript
async function getUser() {

    try {
        const response = await fetch(
            "https://jsonplaceholder.typicode.com/users"
        );

        if (!response.ok) {
            throw new Error("API failed");
        }

        const data = await response.json();

        console.log(data);
    }
    catch (error) {
        console.log(error.message);
    }
}

getUser();
```

**Important Interview Points:**

- An async function always returns a Promise.
- `await` pauses execution of its surrounding async function until the Promise settles.
- It does not block the entire JavaScript thread.
- Use try/catch for error handling.
- Multiple independent operations can be executed in parallel using Promise.all().

## 26. Promise.all() vs Promise.allSettled() vs Promise.race() vs Promise.any()

These methods are used when working with multiple asynchronous operations.

Suppose an application needs to call three APIs:

```javascript
const api1 = fetch("/api/employees");
const api2 = fetch("/api/departments");
const api3 = fetch("/api/projects");
```

### A. Promise.all()

Waits for all Promises to fulfill.

If any Promise rejects, the returned Promise rejects.

```javascript
const results = await Promise.all([
    api1,
    api2,
    api3
]);
```

**Use case:** When all API responses are mandatory.

Note: A rejection does not automatically cancel the other operations.

### B. Promise.allSettled()

Waits for all Promises to settle, regardless of whether they succeed or fail.

```javascript
const results = await Promise.allSettled([
    api1,
    api2,
    api3
]);
```

**Use case:** When the application needs to process successful and failed API responses independently.

### C. Promise.race()

Returns the result of the first Promise to settle, whether fulfilled or rejected.

```javascript
const result = await Promise.race([
    api1,
    api2,
    api3
]);
```

**Use case:** Implementing a timeout or selecting the fastest response.

### D. Promise.any()

Returns the first fulfilled Promise.

It rejects with an AggregateError if all Promises reject.

```javascript
const result = await Promise.any([
    api1,
    api2,
    api3
]);
```

**Use case:** Calling multiple fallback services and accepting the first successful response.

### Comparison

| Method | Behavior | Use case |
|---|---|---|
| all() | All must fulfill | Mandatory APIs |
| allSettled() | Waits for all outcomes | Independent API calls |
| race() | First settled Promise wins | Timeout or fastest response |
| any() | First fulfilled Promise wins | Fallback services |

## 27. JavaScript Event Loop

The Event Loop coordinates synchronous JavaScript execution and asynchronous callbacks.

JavaScript generally executes application code on a single main thread in browser environments.

### Main Components

1. Call Stack
2. Web APIs / Runtime APIs
3. Callback Queue (Task Queue)
4. Microtask Queue
5. Event Loop

### Understanding the Components

**Call Stack**

Executes synchronous JavaScript code using a Last-In-First-Out (LIFO) structure.

**Web APIs**

Browser-provided capabilities such as timers, DOM events and network requests.

**Callback Queue**

Stores callbacks waiting for execution, such as timer callbacks.

**Microtask Queue**

Stores microtasks such as Promise handlers and queueMicrotask callbacks.

**Event Loop**

Checks whether the call stack is empty and coordinates the execution of queued tasks.

### Real-world Analogy: Restaurant Kitchen

Think of the Event Loop as a restaurant kitchen.

| JavaScript Component | Restaurant Analogy |
|---|---|
| Call Stack | Chef preparing a dish |
| Web APIs | Oven or timer working in the background |
| Callback Queue | Counter where completed orders wait |
| Microtask Queue | Priority order counter |
| Event Loop | Manager checking whether the chef is free |

### Example: Understanding Execution Order

```javascript
console.log("Start");

setTimeout(() => {
    console.log("Timeout");
}, 0);

Promise.resolve().then(() => {
    console.log("Promise");
});

console.log("End");
```

**Output:**

```text
Start
End
Promise
Timeout
```

### Why does this happen?

1. `Start` executes synchronously.
2. `setTimeout()` schedules a timer callback.
3. Promise `.then()` schedules a microtask.
4. `End` executes synchronously.
5. The microtask queue is processed.
6. The timer callback executes when its task is eligible.

**Important Interview Point:** A zero-millisecond timeout does not mean immediate execution. Its callback runs in a later task.

## 28. Shallow Copy vs Deep Copy

This is an important concept when working with nested objects and API responses.

### A. Shallow Copy

A shallow copy creates a new outer object, but nested objects remain shared references.

**Example:**

```javascript
const employee = {
    name: "Jitendra",
    address: {
        city: "Pune"
    }
};

const copy = { ...employee };

copy.address.city = "Mumbai";

console.log(employee.address.city);
```

**Output:**

```text
Mumbai
```

**Why?**

The outer object is copied, but both objects reference the same nested `address` object.

### B. Deep Copy

A deep copy creates independent copies of nested objects.

```javascript
const employeeDC = {
    name: "Jitendra",
    address: {
        city: "Pune"
    }
};

const copy = structuredClone(employeeDC);

copy.address.city = "Mumbai";

console.log(employeeDC.address.city);
```

**Output:**

```text
Pune
```

`structuredClone()` supports deep cloning of many JavaScript data types, but not every possible value, such as functions.

### Comparison

| Feature | Shallow Copy | Deep Copy |
|---|---|---|
| Outer object | New object | New object |
| Nested objects | Shared references | Independently cloned |
| Performance | Generally faster | Generally more expensive |
| Use case | Flat objects | Nested objects requiring independence |

**Interview Tip:** Object spread and `Object.assign()` create shallow copies, not deep copies.

## 29. Debouncing

Debouncing means executing a function only after a specified period has passed without another call.

If another call occurs during the waiting period, the timer is reset.

### Implementation

```javascript
function debounce(callback, delay) {

    let timer;

    return function(...args) {

        clearTimeout(timer);

        timer = setTimeout(() => {
            callback.apply(this, args);
        }, delay);

    };
}
```

**Example: Search API**

```javascript
const search = debounce(
    (value) => {
        console.log("API called:", value);
    },
    500
);

search("J");
search("Ji");
search("Jit");
search("Jite");
search("Jiten");
```

Only the final value is processed if no further call occurs during the 500 ms delay.

**Real-world use cases:**

- Search suggestions
- Auto-save
- Form validation
- Window resize events
- API calls triggered by user input

**Benefits:**

- Reduces unnecessary API requests.
- Improves application performance.
- Prevents excessive event handling.

## 30. Throttling

Throttling means executing a function at most once within a specified time interval.

Unlike debouncing, repeated calls do not continually reset the execution timer.

### Implementation

```javascript
function throttle(callback, delay) {

    let lastExecution = 0;

    return function(...args) {

        const now = Date.now();

        if (now - lastExecution >= delay) {

            lastExecution = now;

            callback.apply(this, args);
        }
    };
}
```

**Example: Scroll Event**

```javascript
const handleScroll = throttle(() => {
    console.log("Scroll event executed");
}, 1000);

window.addEventListener("scroll", handleScroll);
```

**Real-world use cases:**

- Scroll events
- Mouse movement
- Window resizing
- Frequent UI events
- Limiting repeated processing

### Debouncing vs Throttling

| Debouncing | Throttling |
|---|---|
| Executes after activity stops | Executes at controlled intervals |
| Resets timer on every call | Limits execution frequency |
| Useful for search | Useful for scrolling |
| Waits for inactivity | Controls execution rate |

## 31. Higher-Order Functions

A higher-order function is a function that either:

1. Accepts another function as an argument.
2. Returns another function.

**Example:**

```javascript
function calculate(operation, a, b) {
    return operation(a, b);
}

function add(a, b) {
    return a + b;
}

function multiply(a, b) {
    return a * b;
}

console.log(calculate(add, 10, 20));      // 30
console.log(calculate(multiply, 10, 20)); // 200
```

**Common examples:**

- map()
- filter()
- reduce()
- forEach()

**Real-world use cases:**

- Reusable business logic
- Custom validation
- Event handling
- Function composition
- Middleware and callback-based processing

## 32. Memoization

Memoization is an optimization technique that stores previously calculated results so repeated calls with the same inputs can return cached results.

**Example:**

```javascript
function memoize(fn) {

    const cache = new Map();

    return function(...args) {

        const key = JSON.stringify(args);

        if (cache.has(key)) {
            return cache.get(key);
        }

        const result = fn.apply(this, args);

        cache.set(key, result);

        return result;
    };
}

function add(a, b) {
    console.log("Calculating...");
    return a + b;
}

const memoizedAdd = memoize(add);

console.log(memoizedAdd(10, 20));
console.log(memoizedAdd(10, 20));
```

**Output:**

```text
Calculating...
30
30
```

The second call uses the cached result.

**Use cases:**

- Expensive calculations
- Repeated data transformations
- Recursive algorithms
- Performance optimization

**Interview Tip:** A practical memoization implementation must consider cache invalidation, memory usage and how arguments are represented as cache keys. JSON.stringify is not suitable for every possible argument type.

## 33. Map and Set

JavaScript provides Map and Set as built-in collection types.

### A. Map

A Map stores key-value pairs. Keys can be of any data type.

**Example:**

```javascript
const employees = new Map();

employees.set(1, "Jitendra");
employees.set(2, "Rahul");

console.log(employees.get(1)); // Jitendra
console.log(employees.has(2)); // true

employees.delete(2);

console.log(employees.has(2)); // false
```

**Important Map Methods:**

| Method | Purpose |
|---|---|
| set() | Add or update an entry |
| get() | Retrieve a value |
| has() | Check whether a key exists |
| delete() | Remove an entry |
| clear() | Remove all entries |

### B. Set

A Set stores unique values. Duplicate values are automatically removed.

```javascript
const numbers1 = new Set([1, 2, 2, 3, 4, 4, 5]);

console.log(numbers1);
// Set(5) {1, 2, 3, 4, 5}
```

**Remove Duplicates from an Array:**

```javascript
const numbers2 = [1, 2, 2, 3, 3, 4];

const unique = [...new Set(numbers2)];

console.log(unique);
// [1, 2, 3, 4]
```

### Map vs Set

| Feature | Map | Set |
|---|---|---|
| Stores | Key-value pairs | Unique values |
| Duplicate keys | No | Not applicable |
| Duplicate values | Allowed | Not allowed |
| Access | By key | By membership |
| Common use | Lookup and caching | Removing duplicates |

## 34. Optional Chaining and Nullish Coalescing

These operators are especially useful when working with nested API responses.

### A. Optional Chaining (?.)

Optional chaining safely accesses nested properties.

If an intermediate property is `null` or `undefined`, the expression returns `undefined` instead of throwing an error.

```javascript
const employee = {
    name: "Jitendra",
    address: {
        city: "Pune"
    }
};

console.log(employee.address?.city); // Pune

console.log(employee.department?.name); // undefined
```

Without optional chaining, accessing a property on `null` or `undefined` can throw a TypeError.

### B. Nullish Coalescing (??)

Provides a default value only when the left-hand value is `null` or `undefined`.

```javascript
const employee = {
    name: "Jitendra",
    salary: 0
};

const salary = employee.salary ?? 10000;

console.log(salary); // 0
```

**Use case:** Avoid replacing valid values such as `0`, `false` or an empty string.

## 35. ES6 Modules – Import and Export

Modules allow JavaScript code to be divided into multiple files, making applications easier to maintain and reuse.

### A. Named Export

**employee.js**

```javascript
export function getEmployee() {
    return {
        id: 101,
        name: "Jitendra"
    };
}

export const company = "MyCompany";
```

**app.js**

```javascript
import { getEmployee, company } from "./employee.js";

console.log(getEmployee());
console.log(company);
```

### B. Default Export

**employee.js**

```javascript
export default function getEmployee() {
    return {
        id: 101,
        name: "Jitendra"
    };
}
```

**app.js**

```javascript
import getEmployee from "./employee.js";

console.log(getEmployee());
```

**Comparison:**

| Named Export | Default Export |
|---|---|
| Multiple exports per file | One default export per module |
| Import using the exported name | Import can use a chosen local name |
| Curly braces required for named imports | No curly braces for default import |

## 36. Error Handling

JavaScript provides several mechanisms to handle errors and prevent unexpected application failures.

Important keywords:

- try
- catch
- finally
- throw

**Example:**

```javascript
function divide(a, b) {

    try {

        if (b === 0) {
            throw new Error("Division by zero is not allowed");
        }

        return a / b;

    }
    catch (error) {
        console.log(error.message);
    }
    finally {
        console.log("Operation completed");
    }
}

divide(10, 0);
```

**Explanation:**

- `try`: Contains code that might throw an error.
- `catch`: Handles the error.
- `finally`: Executes whether an error occurs or not.
- `throw`: Creates and throws an error.

**Real-world use cases:**

- API error handling
- Form validation
- File processing
- Business rule validation
- Unexpected runtime exceptions

---

# Part 4: Advanced JavaScript Concepts

## 37. Event Delegation

Event delegation means attaching a single event listener to a parent element instead of attaching individual listeners to every child element.

It works because of event bubbling.

**HTML:**

```html
<ul id="employeeList">
    <li data-id="1">Jitendra</li>
    <li data-id="2">Rahul</li>
    <li data-id="3">Amit</li>
</ul>
```

**JavaScript:**

```javascript
document.getElementById("employeeList")
    .addEventListener("click", function(event) {

        const item = event.target.closest("li");

        if (item && this.contains(item)) {

            console.log(item.dataset.id);
            console.log(item.textContent);

        }
    });
```

**Benefits:**

- Fewer event listeners.
- Better memory efficiency for large lists.
- Works with dynamically added child elements.
- Easier event management.

**Real-world example:** A dynamic employee list where users can add or remove employee records without attaching new event listeners to every row.

## 38. Generator Functions

A generator function can pause its execution and resume it later.

Generator functions use the `function*` syntax and the `yield` keyword.

**Example:**

```javascript
function* generateNumbers() {
    yield 10;
    yield 20;
    yield 30;
}

const generator = generateNumbers();

console.log(generator.next());
console.log(generator.next());
console.log(generator.next());
console.log(generator.next());
```

**Output:**

```javascript
{ value: 10, done: false }
{ value: 20, done: false }
{ value: 30, done: false }
{ value: undefined, done: true }
```

**How does it work?**

1. Calling the generator function returns an iterator.
2. Execution starts when `next()` is called.
3. Execution pauses at each `yield`.
4. The next `next()` call resumes execution.
5. After completion, `done` becomes `true`.

**Use cases:**

- Lazy evaluation
- Iterators
- Processing large sequences
- Custom iteration logic
- Controlling execution flow

## 39. JavaScript Memory Management

JavaScript automatically manages memory through garbage collection.

### General Memory Management Process

1. Memory is allocated when objects and variables are created.
2. JavaScript tracks whether objects remain reachable.
3. Objects that are no longer reachable can be garbage-collected.

### Example of a Potential Memory Leak

```javascript
let employeeData = [];

function loadData() {

    employeeData.push(
        new Array(100000).fill("Employee")
    );

}
```

If this function keeps adding data that is no longer needed, memory consumption can continue increasing.

### Event Listener Memory Leak

```javascript
function handleClick() {
    console.log("Clicked");
}

window.addEventListener("resize", handleClick);

// When no longer needed:
window.removeEventListener("resize", handleClick);
```

### Common Causes of Memory Leaks

- Event listeners that are never removed.
- Unnecessary global variables.
- Unbounded caches.
- Timers that are never cleared.
- References to objects that are no longer needed.
- Closures retaining large objects unnecessarily.

**Interview Tip:** JavaScript garbage collection commonly uses reachability-based techniques. Developers should avoid retaining unnecessary references to objects.

## 40. Lazy Loading

Lazy loading means loading resources only when they are required rather than loading everything when the application starts.

### Example: Dynamic Module Import

```javascript
async function loadDashboard() {

    const module = await import("./dashboard.js");

    module.showDashboard();
}

loadDashboard();
```

Instead of loading the dashboard module initially, JavaScript loads it when the function executes.

**Benefits:**

- Reduces initial application loading time.
- Reduces unnecessary resource downloads.
- Improves application performance.
- Supports code splitting.

**Real-world use cases:**

- Loading dashboard components on demand.
- Loading images when they enter the viewport.
- Loading large modules only when users navigate to a feature.

## 41. Web Workers

Web Workers allow JavaScript to execute certain tasks in a separate background thread.

Normally, JavaScript application code runs on the main thread. Expensive calculations can make the UI unresponsive.

Web Workers help move CPU-intensive calculations away from the main thread.

### Main File – main.js

```javascript
const worker = new Worker("worker.js");

worker.postMessage(1000000);

worker.onmessage = function(event) {
    console.log("Result:", event.data);
};
```

### Worker File – worker.js

```javascript
self.onmessage = function(event) {

    let total = 0;

    for (let i = 0; i < event.data; i++) {
        total += i;
    }

    self.postMessage(total);
};
```

**How does it work?**

1. The main thread creates a Web Worker.
2. The main thread sends data using `postMessage()`.
3. The worker performs the calculation.
4. The worker sends the result back.
5. The main thread receives the result through `onmessage`.

**Use cases:**

- Large calculations.
- Data processing.
- Image processing.
- CPU-intensive operations.

**Important Interview Points:**

- Workers run separately from the main thread.
- Workers cannot directly manipulate the DOM.
- They communicate through messages.
- They can be terminated using `worker.terminate()`.

## 42. Microtasks vs Macrotasks

This is an important topic related to the JavaScript Event Loop.

JavaScript processes synchronous code first, then microtasks, and subsequently eligible tasks from the task queue.

### Microtasks

Examples:

- Promise.then()
- Promise.catch()
- Promise.finally()
- queueMicrotask()

### Tasks (Often Called Macrotasks)

Examples:

- setTimeout()
- setInterval()
- DOM event callbacks

### Example

```javascript
console.log("A");

setTimeout(() => {
    console.log("B");
}, 0);

Promise.resolve().then(() => {
    console.log("C");
});

queueMicrotask(() => {
    console.log("D");
});

console.log("E");
```

**Output:**

```text
A
E
C
D
B
```

### Explanation

1. A executes synchronously.
2. setTimeout schedules a timer task.
3. Promise.then() schedules a microtask.
4. queueMicrotask() schedules another microtask.
5. E executes synchronously.
6. Microtasks execute in their queue order: C, then D.
7. The timer callback executes afterward.

**Important Interview Point:** Microtasks are processed before the next task is selected. Newly scheduled microtasks are also processed before moving to the next task.

## 43. Object.freeze() vs Object.seal()

Both methods restrict modifications to JavaScript objects.

### A. Object.freeze()

Prevents adding, deleting and modifying existing properties of an object.

```javascript
const employee = {
    name: "Jitendra",
    salary: 100000
};

Object.freeze(employee);

employee.name = "Rahul";

console.log(employee.name); // Jitendra
```

### B. Object.seal()

Prevents adding or deleting properties but allows modification of existing writable properties.

```javascript
const employee = {
    name: "Jitendra",
    salary: 100000
};

Object.seal(employee);

employee.name = "Rahul"; // Allowed

employee.city = "Pune"; // Not allowed

delete employee.salary; // Not allowed

console.log(employee.name); // Rahul
```

### Comparison

| Feature | Object.freeze() | Object.seal() |
|---|---|---|
| Add properties | No | No |
| Delete properties | No | No |
| Modify existing properties | No | Yes, if writable |
| Restriction level | Stronger | Less restrictive |

**Important Interview Point:** Both are shallow operations. Nested objects are not automatically frozen or sealed.

For nested objects, additional recursive handling is needed if deep immutability is required.

## 44. preventDefault() vs stopPropagation()

These methods are frequently asked in JavaScript interviews, but they solve different problems.

### A. event.preventDefault()

Prevents the browser's default action for an event.

**Common examples:**

- Clicking an anchor normally navigates to another page.
- Submitting a form normally triggers its default submission behavior.
- Certain keyboard actions have browser-defined behavior.

**HTML:**

```html
<form id="signup-form">
    <input type="text" placeholder="Enter your email" />
    <button type="submit">Sign Up</button>
</form>

<a id="my-link" href="https://google.com">
    Go to Google
</a>
```

**JavaScript:**

```javascript
// Prevent form submission's default behavior
document.getElementById("signup-form")
    .addEventListener("submit", (event) => {

        event.preventDefault();

        console.log("Form handled using JavaScript");
    });

// Prevent link navigation
document.getElementById("my-link")
    .addEventListener("click", (event) => {

        event.preventDefault();

        console.log("Navigation prevented");
    });
```

### B. event.stopPropagation()

Stops an event from propagating further through the DOM event path.

**HTML:**

```html
<div id="outer-box">
    Outer Box

    <button id="inner-btn">
        Click Me
    </button>
</div>
```

**JavaScript:**

```javascript
document.getElementById("outer-box")
    .addEventListener("click", () => {

        console.log("Outer box was clicked!");
    });

document.getElementById("inner-btn")
    .addEventListener("click", (event) => {

        event.stopPropagation();

        console.log("Button was clicked!");
    });
```

When the button is clicked, the button handler executes, but the bubbling event does not reach the outer box handler.

### Comparison

| Feature | preventDefault() | stopPropagation() |
|---|---|---|
| Purpose | Prevent browser default action | Stop event propagation |
| Stops navigation | Yes, when cancelable | No |
| Stops form default submission | Yes, when cancelable | No |
| Stops parent event handlers | No | Yes, along the propagation path |
| Prevents event execution entirely | No | No |

**Important:** `stopPropagation()` does not prevent the browser's default action. Similarly, `preventDefault()` does not stop event bubbling.

---

# Part 5: Senior-Level Interview Revision

## Important Comparison Questions

| Question | Key Concept |
|---|---|
| var vs let vs const | Scope and reassignment |
| == vs === | Type coercion |
| null vs undefined | Missing and empty values |
| || vs ?? | Falsy vs nullish values |
| Function declaration vs expression | Hoisting |
| Normal function vs arrow function | this binding |
| map() vs filter() | Transformation vs filtering |
| find() vs filter() | First match vs all matches |
| forEach() vs map() | Iteration vs transformation |
| Spread vs Rest | Expand vs collect |
| Shallow copy vs deep copy | Reference sharing |
| Callback vs Promise | Asynchronous programming |
| Promise vs async/await | Asynchronous syntax |
| Promise.all() vs allSettled() | Failure handling |
| Promise.race() vs any() | First settled vs first fulfilled |
| Call vs apply vs bind | Function invocation |
| Closure vs scope | Lexical environment |
| Debouncing vs throttling | Execution frequency |
| Map vs Set | Key-value vs unique values |
| Object.freeze() vs seal() | Object modification |
| Event bubbling vs capturing | Event propagation |
| preventDefault vs stopPropagation | Browser behavior vs event propagation |
| Microtask vs task | Event loop execution order |
| Web Worker vs main thread | Background processing |

## Practical Interview Preparation Checklist

### JavaScript Fundamentals

- [ ] var, let and const
- [ ] Primitive and non-primitive types
- [ ] Type coercion
- [ ] Equality operators
- [ ] Logical and comparison operators
- [ ] Nullish coalescing
- [ ] Conditional statements
- [ ] Loops and functions

### Intermediate JavaScript

- [ ] Array methods
- [ ] Object manipulation
- [ ] Destructuring
- [ ] Spread and rest operators
- [ ] Scope and lexical scope
- [ ] Hoisting and TDZ
- [ ] Callback functions
- [ ] DOM manipulation
- [ ] Event bubbling and capturing

### Advanced JavaScript

- [ ] Closures
- [ ] this keyword
- [ ] call(), apply() and bind()
- [ ] Prototypes and inheritance
- [ ] Classes and static methods
- [ ] Promises
- [ ] Async/await
- [ ] Promise combinators
- [ ] Event Loop
- [ ] Microtasks and tasks
- [ ] Shallow and deep copy
- [ ] Debouncing and throttling
- [ ] Higher-order functions
- [ ] Memoization
- [ ] Map and Set
- [ ] Optional chaining
- [ ] ES6 modules
- [ ] Error handling
- [ ] Generator functions
- [ ] Memory management
- [ ] Lazy loading
- [ ] Web Workers

## Final Interview Preparation Advice

For senior software engineering interviews, understanding JavaScript syntax is not enough. Focus on the internal execution model and practical application.

For every topic, prepare answers to these six questions:

1. What is it?
2. Why do we need it?
3. How does it work internally?
4. What is the difference between similar concepts?
5. Where would you use it in a real-world application?
6. Can you implement it without using built-in shortcuts?

**Recommended focus areas:** Closures, Event Loop, Promises, Async/Await, `this`, Prototypes, Array Methods, Debouncing, Throttling and Memory Management.

These concepts are particularly important when working with modern frontend frameworks such as Angular and when integrating JavaScript applications with .NET Web APIs.

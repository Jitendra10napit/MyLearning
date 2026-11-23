// Online JavaScript compiler (editor)
// Write and run JavaScript online using this JS editor.
//Variable Declaration
/*
var a = 10;
let b = 20;
const c = 30;

a = 15;
b = 25;*/
//c = 35;

//console.log(`${a}, ${b}, ${c}`)  
//console.log("Try clicking the Run button.");

//Block scope
/*
if(true){
  var x= 10;
  let y = 20;
  const z = 30;
  //console.log(y);
}

*/

//var x = 30;
//let y = 50;
//console.log(x);
//c=scope problem
//console.log(z);


//Primitive types:
/*
let name = "Jitendra";
let age = 40;
let isActive = true;
let salary;

let employee = {
  id: 101,
  name: "Jitendra"
}


let skills = ["c#", "Javascript", "Angular"];
*/
/*
console.log(typeof name);
console.log(typeof age);
console.log(typeof isActive);
console.log(typeof salary);

console.log(typeof employee);
console.log(typeof skills);
console.log(typeof null);

*/

//Difference Between == and ===

// == (compares values after performing type conversation when necessary)
// === (compares both value and types without implicit type conversion)

/*
let a = 10;
let b = 5;

//Arthmetic
console.log(a+b);
console.log(a-b);
console.log(a*b);
console.log(a/b);
console.log(a%b);

//Comparison

console.log(a>b);
console.log(a<b);

//Logical

console.log(a>5 && b<10);
console.log(a>20 || b<10);
console.log(!(a>b));
*/
//Nullish Coaleascing vs OR


//let a = 0;

//console.log(a || 100);

//a = 0
//console.log(a ?? 100);
 

// || uses the right value whebn the left value is falsy;
// ?? uses the right value only whebn the left value is null or undefined


//6. Conditional Statements

/*
let marks = 75;

if(marks>=90)
  console.log("Grade A");
else if(marks>=60)
  console.log("Grade B");
else
  console.log("Grade C");
*/

//Ternary operator:
/*
let age = 20;

let result = age>=18 ? "Adult" : "Minor";

console.log(result);
*/

//Switch
/*
let role = "Admin";

switch(role){
  case "Admin":
    console.log("Full Access");
    break;

  case "User":
    console.log("Limited access");
    break;

  default:
    console.log("No access");
}


//Looping
console.log("for loop");

for(let i = 0; i<=5; i++){
  console.log(i); 
}

console.log("While loop");
let k = 1; 
//While loop

while(k<=5){
  console.log(k);
  k++;
}

// for of iterates over values
console.log("Iterate for values in loop");

let skills = ["C#", "Javascript", "Angular"];

for(const skill of skills){
  console.log(skill);
}

console.log("Iterate for employee object in loop");

let employee = {
  id:101,
  name:"Jitendra",
  role:"Developer"
}

for(const key in employee)
{
  console.log(key, employee[key]);
}

*/

//Normal function

function  add(a, b){
  return a + b;
}

//console.log(add(10, 20));


// function expression
const addc = function(a, b){
  return a + b;
}

//console.log(addc(10,10));


// Arrow function
//Important difference: Arrow functions do not have their own this or arguments object and cannot be used as //constructors.
const addA = (a, b)=> a + b;

//console.log(addA(10, 40));

//Array and important array methods

let numbers = [10, 20, 30, 40 ,50];

/*
console.log(numbers[0]);

numbers.push(60); //add at end
console.log(numbers);
numbers.pop(); // Remove from end
console.log(numbers);
numbers.unshift(5); // add at beginning
console.log(numbers);
numbers.shift(); // remove from beginning
console.log(numbers); 
*/


//following methods are especially important for interviews

// A. map() - transform each element

//let result = numbers.map(x=> x/10);

//console.log(result);

//B. Filters()- Select matching elements 
/*
let num = [1, 2, 3, 4, 5, 6];

let result = num.filter(x=>x%2==0);

//console.log(result);



// C reduce() - calcualte a single result

let total = numbers.reduce(
  (s, c)=> s + c, 0
);

//console.log(total);

const cart = [
  {item: "Coffee", price: 150},
  {item: "Sandwich", price: 220},
  {item: "Cookie", price: 150}
]

let tb = cart.reduce(
  (sum, current)=> sum+ current.price, 0
);

//console.log(tb);


// find() // return first matching element

let res = numbers.find(s=>s>30);

console.log(res);


let evenNum = [2, 4, 6, 8, 10];

console.log(evenNum.some(s=>s>5));

console.log(evenNum.every(s=>s%2==0))


const employeeData = {
  id: 101,
  name:"jitendra",
  salary: 500000,

  display: function() {
    console.log(this.name);
  }
}

console.log(employeeData.name);
console.log(employeeData["salary"]);

employeeData.display();

// Destructuring:

const {id, name, salary} = employeeData;
console.log(name);

*/

//Spread Operator

const emp = {
  id: 101,
  name: "jitendra" 
};

const updateEmp = {
  ...emp,
  city:"pune"
}

//console.log(updateEmp);

//Rest operator

let nums = [10,20,30,40,50];
function sum(...numbers){
  return numbers.reduce((a, b) => a + b, 0);
}

//console.log(sum(...nums));

//Note: Spread and rest use the same ... syntax, but their purpose depends on where they are used.




// Part2: Intermediate Javascript

//11. scope in javascript

// Scope determines where a variable can be accessed

//There are three commonly discussed scopes:

//Global scope, Function scope, Block scope

//Example

let globalValue = 100;

function test(){
  let functionValue = 200;

  if(true){
    let blockScope = 300;

    console.log(globalValue);
    console.log(functionValue);
    console.log(blockScope);
  }

  // console.log(blockScope); //error blockScope is not definded
}

// Javascript also uses lexical scope, meaning a function can access variables from its surrouding scope

//test();

//12. Hoisting:: Hoisting means Javascript processess declarations before executing code in their scope.

var j;
//console.log(j);

j = 10;

let x;
//console.log(x);

//Function hoisting

//sayHello(); // Works! Outputs: "Hello!"

function sayHello() {
  console.log("Hello!");
}

//13. CallBack function

// A callback is a function passed as an argument to another function.

/*
function processUser(name, callback){
  console.log(`processing ${name}`)
  callback();
}

function completed(){
  console.log("Processing Completed!");
}

processUser("Jitendra", completed);




//syncronous and asyncronous function

console.log("First");

setTimeout(()=>{
  console.log("Second");
}, 3000);

console.log("Third");
*/

//DOM: Document object model: it allows javascript to access the html elements

//document.getElementById("id");

/*
  element.addEventListenner("click", function(){
  console.log("clicked");
  });
*/


//16 Event Bubbling and Event Capturing

//Event bubbling: Event moves from the target element to its parent elements.

//Event capturing: Event moves from the outer parent elements towards the target.

/*
event.stopPropagation() (Stopping the Spread)

Use this when you click an inner element and do not want the parent element's click event to fire.
Real-world use case: A clickable product card that opens a product page, with a "Favorite" heart button inside it. Clicking the heart shouldn't open the product page.
*/


/*
2. Event Delegation (Using Bubbling to Save Memory)

Instead of attaching separate event listeners to dozens of child items, attach one single listener to their parent and let events bubble up to it.
Real-world use case: A dynamic to-do list where items are constantly added or removed.

html
<ul id="todo-list">
  <li>Buy groceries</li>
  <li>Walk the dog</li>
  <li>Read a book</li>
</ul>
Use code with caution.
javascript
const list = document.getElementById("todo-list");

// Only ONE listener on the parent <ul>
list.addEventListener("click", (event) => {
  // event.target is the exact <li> that was clicked
  if (event.target.tagName === "LI") {
    event.target.style.textDecoration = "line-through";
    console.log("Completed:", event.target.textContent);
  }
});

*/

//Part 3: Advanced JavaScript – Most Important Interview Topics 

//17 closure:A closure happens when an inner function remembers and accesses variables from its outer function, even // after the outer function has completed execution.

/*
function outer(){
  let count = 0;

    function inner(){
      count++;
      console.log(count);
    }

  return inner;
}

const counter = outer();

counter();
counter();
counter();


function createBankAccount(initialBalance){
  let balance = initialBalance;

    return {

        deposit(amount){
          balance += amount;
          return balance;
        },

        getBalance(){
          return balance;
        }
    };
}

const account = createBankAccount(1000);

console.log(account.deposit(500));
console.log(account.getBalance());
*/

//this keyword

//this referes to the execution context of a funtion. its value depends on how the function is called.

//Object Method:

const employee = {
  name:"Jitendra",

  display(){
    console.log(this.name);
  }
}

//employee.display();


//Normal function

function display(){
  console.log(this);
}

//display(); // global object in non strict mode but in strict mode its undefined


//Example: Arrow function:: An Arrow function does not have own this and it inherits this from its surrounding lexical scope.
/*
const emp1 = {
  name:"jitendra",

  display: ()=>{
    console.log(this.name); 
  }
}

//emp1.display();

*/

/*
// 19 call(), apply(), bind()

const emp1 = {
  name:"jitendra"
};

const emp2 = {
  name:"Rahul"
};

function greet(city, country){
  console.log(`${this.name} ${city} ${country}`);
}


//call() Pass arguments individually

greet.call(emp1, "Pune", "India");

//apply(): Pass arguments as an array

greet.apply(emp2, ["Bhopal", "India"]);


//bind: Returns a new functions

const greetEmployee = greet.bind(emp1, "Pune", "India");

greetEmployee();

*/

//20 Prototype and prototypal inheritance

//Javascript uses prototype inheritance

//An object can inherit properties and methods from another object through its prototype chain

/*

const person = {

  greet(){
    console.log("hello");
  }
}

const emp1 = Object.create(person);

emp1.name = "Jitendra";

emp1.greet();


// Constructor function example

function Employee(name){
  this.name = name;
}


Employee.prototype.display = function(){
  console.log(this.name);
}

const e1 = new Employee("Jitendra");
const e2 = new Employee("Rahul");

e1.display();
e2.display();
*/
//Both object share the same display method through the prototype

//why is this useful
// Instead of creating duplicate methods for every object, javascript can share methods through prototypes.

//21. Class and inheritance
/*
class Employee{
  constructor(name){
    this.name = name;
  }

  display(){
    console.log(this.name);
  }
}

class Developer extends Employee{

  constructor(name, technology){
    super(name)
    this.technology = technology;
  }

  showTechnology(){
    console.log(this.technology);
  }
}

const dev = new Developer("Jitendra", "Dotnet");


dev.display();
dev.showTechnology(); 

class MathHelper{

  static add(a, b){
    return a + b;
  }
}

console.log(add(10, 20));

*/


//22. Promise :: A promise represents the eventual result of an asyncronouse operation 

// A promise has three states :
// 1. Pending: Initial state that says Operation is still running
// 2. FulFilled: OPeration successful
// 3. Reject: Operation

/*
const promise = new Promise((resolve, reject) =>{

  let success = true;

  if(success){
    resolve("data fetched successfully");
  }
  else{
    reject("Something went wrong");
  }
  
});

promise
  .then(result=> console.log(result))
  .catch(error=> console.log(error))
  .finally(()=> console.log("Complete"))

mportant methods:
Method	Purpose
then()	Handle success
catch()	Handle errors
finally()	Execute regardless of outcome
Promise.all()	Wait for all promises; reject on first rejection
Promise.allSettled()	Wait for all outcomes
Promise.race()	First settled promise wins
Promise.any()	First fulfilled promise wins
*/

// 23. async and await: it make asyncronus code easier to read and maintain.

//Without async and await
/*
fetch("https://jsonplaceholder.typicode.com/users")
    .then(response => response.json())
    .then(data => console.log(data))
    .catch(error => console.log(error));

   

//With async and await

async function getUser(){

    try{
      const response =  await fetch(
        "https://jsonplaceholder.typicode.com/users"
      )

      if(!response.ok){
        throw new Error("API failed");
      }

      const data = await response.json();
      console.log(data);
    }
  catch(error){
    console.log(error.message);
  }
}

getUser();

Important interview points:
- An async function always returns a Promise.
- await pauses the execution of its surrounding async function until the Promise settles.
- It does not block the entire JavaScript thread.
- Use try/catch for error handling.

*/

//24. Promise.all() vs Promise.allSettled() vs Promise.race() vs Promise.any()
/*
Suppose we need to call three APIs.

const api1 = fetch("/api/employees");
const api2 = fetch("/api/departments");
const api3 = fetch("/api/projects");

const results = await promise.all([
  api1, api2, api3
])

//All must fulfill for the overall Promise to fulfill. If one rejects, the overall Promise rejects immediately, although //other operations are not automatically cancelled.

//Promise.allSettled():
const results = await Promise.allSettled([
    api1,
    api2,
    api3
]);


//Waits for all promises to settle, whether fulfilled or rejected.

// Promise.race():
const result = await Promise.race([
    api1,
    api2,
    api3
]);


// Returns the outcome of the first Promise to settle, success or failure.


//Promise.any():
const result = await Promise.any([
    api1,
    api2,
    api3
]);


// Returns the first fulfilled Promise. It rejects with AggregateError if all promises reject.

Method	    Behavior	                 Use case
all	        All must succeed	          Multiple mandatory APIs
allSettled	Wait for all results	      Independent API calls
race	    First settled wins	          Timeout or fastest response
any	        First successful wins	      Multiple fallback services

*/

/*
// Event Loop: 
    The JavaScript Event Loop manages synchronous code and asynchronous callbacks.

Main components:
1. Call Stack.
2. Web APIs / Runtime APIs.
3. Callback Queue (Task Queue).
4. Microtask Queue.
5. Event Loop.


The Event Loop is the mechanism that lets JavaScript run asynchronous tasks (like timers or network requests) without freezing your app, even though JavaScript can only execute one line of code at a time (single-threaded).
Think of it like a restaurant kitchen:
1. Call Stack: The chef cooking one dish at a time (synchronous code).
2. Web APIs: The oven or timer running in the background while the chef keeps cooking (setTimeout, fetch).
3. Callback / Task Queue: A counter where finished background tasks wait in line.
4. Event Loop: A manager constantly checking: "Is the chef free? If yes, bring the next waiting task from the counter to the chef."

console.log("Start");

setTimeout(() => {
    console.log("Timeout");
}, 0);

Promise.resolve().then(() => {
    console.log("Promise");
});

console.log("End");

Start
End
Promise
Timeout


Why?
1. Start executes synchronously.
2. setTimeout schedules a timer callback.
3. Promise .then() schedules a microtask.
4. End executes synchronously.
5. Microtasks execute before the next timer task.
6. Timeout callback executes.
Important distinction:
- Promise callbacks are microtasks.
- setTimeout callbacks are tasks (often called macrotasks).
- Microtasks are processed before the next task is selected.



Bonus: Microtasks vs. Macrotasks (Interview Favorite!)

Not all waiting queues have equal priority.
Promises go into the Microtask Queue, which has higher priority than setTimeout (Macrotask Queue).
javascript
console.log("1. Sync");

setTimeout(() => console.log("2. setTimeout (Macrotask)"), 0);

Promise.resolve().then(() => console.log("3. Promise (Microtask)"));

console.log("4. Sync");

// Output order:
// 1. Sync
// 4. Sync
// 3. Promise (Microtask)  <-- Runs BEFORE setTimeout!
// 2. setTimeout (Macrotask) 

console.log("1. Sync");

setTimeout(() => console.log("2. setTimeout (Macrotask)"), 0);

Promise.resolve().then(() => console.log("3. Promise (Microtask)"));

console.log("4. Sync");
*/

/*
26. Shallow Copy vs Deep Copy
This is very important when handling objects received from APIs.

Shallow copy:
const employee = {
    name: "Jitendra",
    address: {
        city: "Pune"
    }
};

const copy = { ...employee };

copy.address.city = "Mumbai";

console.log(employee.address.city);


Output:
Mumbai


Why?

Because the outer object is copied, but the nested address object is shared.

//Deep copy:
const employeeDC = {
    name: "Jitendra",
    address: {
        city: "Pune"
    }
};

const copy = structuredClone(employeeDC);

copy.address.city = "Mumbai";

console.log(employeeDC.address.city); // Pune


//structuredClone() supports deep cloning of many JavaScript data types, but not every possible value (for example, functions).

*/

/*
//27. Debouncing
//Debounding means executing a function only after a specified period has passed without another call.


function debounce(callback, delay){
  let timer;

  return function(...args){
    clearTimeout(timer);
    timer = setTimeout(()=>{
      callback.apply(this, args);
    }, delay);
  };
}

const search = debounce((value) =>{console.log("api called", value)}, 500);

search("J");
search("Ji");
search("Jit");
search("Jite");
search("Jiten");

Only the last value is processed if no further call occurs during the 500 ms delay.

Use cases:
- Search suggestions.
- Auto-save.
- Form validation.
- Resize events.

*/

/*
//28. Throttling

//Throttling means executing a function at most once within a specified time interval.


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

const handleScroll = throttle(() => {
    console.log("Scroll event executed");
}, 1000);

handleScroll();

Use cases:
- Scroll events.
- Mouse movement.
- Window resizing.
- Frequent UI events.
Debouncing vs Throttling: 

Debouncing vs Throttling:
Debouncing	                         Throttling
Executes after activity stops	     Executes at controlled intervals
Useful for search	                 Useful for scrolling
Resets timer on each call	         Limits execution frequency
*/

/*
//29. Higher Order functions

//A higher-order function is a function that accepts another function as an argument or returns a function.

function calculate(operation, a, b){
  return operation(a, b);
}

function add(a, b){
  return a + b;
}

function multiply(a, b){
  return a*b;
}

console.log(calculate(add, 10, 20)); //30

console.log(calculate(multiply, 10, 20)); //200
*/


//31. Memoization
//Memoization stores previously calculated results so that repeated calls can return cached results instead of recalculating.


//32. Map and set


//Map => Stores key value pairs, keys can be of any type

/*
const employees = new Map();

employees.set(1, "abc");
employees.set(2, "xyz");

console.log(employees.get(1)); //Jitendra
console.log(employees.has(2)); // true

 employees.delete(2);
console.log(employees.has(2)); // true


//Set: Stores unique values.

const numbers1 = new Set([1,2,2,3,4,4,5])

console.log(numbers1);

  
const numbers2 = [1, 2, 2, 3, 3, 4];

const unique = [...new Set(numbers2)];

console.log(unique); // [1, 2, 3, 4]


//33. Optional Chaining and Nullish Coalescing
      // Note: Very useful when accessing nested API responses.
const employee = {
    name: "Jitendra",
    address: {
        city: "Pune"
    }
};

//console.log(employee.address?.city); // Pune

//console.log(employee.department?.name); // undefined


//Without optional chaining, accessing a property on undefined or null can throw an error.

Nullish coalescing:
const employee = {
    name: "Jitendra",
    salary: 0
};

const salary = employee.salary ?? 10000;

//console.log(salary); // 0


//This avoids replacing valid values such as 0, false or an empty string.

*/

//34. ES6 Modules – import and export

//Note: Modules allow us to split JavaScript code into multiple files.

  //employee.js
/*
export function getEmployee(){
  return{
    id:101,
    name:"Jitendra"
  }
}

export const company = "MyComapny";

//app.js

import {getEmployee, company} from "./employee.js";


consol.log(getEmoloyee());
consol.log(company);
*/

//35. Error handling

//Javascript provides try, catch, finally and throw.


//Part 4: Advanced Javascript 

  //36. Event Delegation: 

  //Event delegation means ading a single event listener to a parent element instead of adding listeners to every child element.
  
/*
HTML CODE

<ul id="employeeList">
    <li data-id="1">Jitendra</li>
    <li data-id="2">Rahul</li>
    <li data-id="3">Amit</li>
</ul>
*/
/*
//Javascript code

  document.getElementById("employeeList")
          .addEventListener("click", function(event){

            if(event.target.matches("li")){
              console.log(event.target.dataset.id);
              console.log(event.target.textContent); 
            }
        });

Benefits:
- Fewer event listeners.
- Better memory efficiency for large lists.
- Works for dynamically added child elements.
- Easier event management.
*/

/*
//Generator functions
  //A generator function can pause execution and resume it later

  function* generateNumners(){
    yield 10;
    yield 20;
    yield 30; 
  }

const gentr = generateNumners();

console.log(gentr.next());
console.log(gentr.next());
console.log(gentr.next());
console.log(gentr.next());

Use cases:
- Lazy evaluation.
- Iterators.
- Processing large sequences.
- Controlling execution fl
*/


/*
38. JavaScript Memory Management
JavaScript automatically manages memory using garbage collection.

The general process is:
1. Memory is allocated when objects and variables are created.
2. JavaScript tracks whether objects remain reachable.
3. Unreachable objects can be garbage-collected.
Example of a potential memory leak:

let employeeData = [];

function loadData() {
    employeeData.push(
        new Array(100000).fill("Employee")
    );
}

If this function keeps adding data that is no longer needed, memory usage can grow.

Another common example is an event listener that is never removed.

function handleClick() {
    console.log("Clicked");
}

window.addEventListener("resize", handleClick);

// When no longer needed:
window.removeEventListener("resize", handleClick);

Common memory leak causes:
- Unremoved event listeners.
- Unnecessary global variables.
- Unbounded caches.
- Timers that are never cleared.
- References to objects that are no longer needed.

//
Example: Lazy loading a module
async function loadDashboard() {

    const module = await import("./dashboard.js");

    module.showDashboard();
}

loadDashboard();


Instead of loading everything initially, the dashboard module is loaded when needed.

*/

/*
//40 Web worker:

// JavaScript normally executes application code on the main thread.
// If we perform expensive calculations on the main thread, the UI can become unresponsive.
// Web Workers execute JavaScript in a separate background thread.

main.js:
const worker = new Worker("worker.js");

worker.postMessage(1000000);

worker.onmessage = function(event) {
    console.log("Result:", event.data);
};

worker.js:
self.onmessage = function(event) {

    let total = 0;

    for (let i = 0; i < event.data; i++) {
        total += i;
    }

    self.postMessage(total);
};

Use cases:
- Large calculations.
- Data processing.
- Image processing.
- CPU-intensive operations.
Important: Workers cannot directly manipulate the DOM. They communicate with the main thread through messages.


41. Microtasks vs Macrotasks
This is another common event-loop question.

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


Output:
A
E
C
D
B


Explanation:
- A and E are synchronous.
- C and D are microtasks.
- B is a timer task.
- Microtasks execute before the next timer task.


42. Object.freeze() vs Object.seal()
Both methods restrict modifications to objects.

Object.freeze():
const employee = {
    name: "Jitendra",
    salary: 100000
};

Object.freeze(employee);

employee.name = "Rahul";

console.log(employee.name); // Jitendra


Object.seal():
const employee = {
    name: "Jitendra",
    salary: 100000
};

Object.seal(employee);

employee.name = "Rahul"; // Allowed

employee.city = "Pune"; // Not allowed

delete employee.salary; // Not allowed

console.log(employee.name); // Rahul


Freeze	Seal
Cannot add properties	Cannot add properties
Cannot delete properties	Cannot delete properties
Cannot modify existing properties	Can modify existing properties
More restrictive	Less restrictive

Both are shallow operations. Nested objects are not automatically frozen or sealed.


43. Event Handling: preventDefault vs stopPropagation
This is a commonly asked interview question.

preventDefault(): Prevents the browser's default action.
document.querySelector("a").addEventListener(
    "click",
    function(event) {
        event.preventDefault();
        console.log("Navigation prevented");
    }
);


stopPropagation(): Stops the event from moving further through the DOM propagation path.
document.querySelector("button").addEventListener(
    "click",
    function(event) {
        event.stopPropagation();
        console.log("Button clicked");
    }
);


They solve different problems.


1. event.preventDefault() (Stops Browser Behavior)

Certain HTML elements have built-in jobs. Calling preventDefault() tells the browser: "Don't do your automatic job—let my JavaScript handle this instead."

Common Examples:

• Clicking an <a> tag normally opens a new webpage.
• Submitting a <form> normally reloads the entire page.

Code Example:

html
<form id="signup-form">
  <input type="text" placeholder="Enter your email" />
  <button type="submit">Sign Up</button>
</form>

<a id="my-link" href="https://google.com">Go to Google</a>
Use code with caution.
javascript
// 1. Stop a form from reloading the page
document.getElementById("signup-form").addEventListener("submit", (event) => {
  event.preventDefault(); // Page stays put!
  console.log("Form submitted using JavaScript instead of reloading!");
});

// 2. Stop a link from navigating away
document.getElementById("my-link").addEventListener("click", (event) => {
  event.preventDefault(); // Won't open Google!
  console.log("Link clicked, but stayed on the same page.");
});



2. event.stopPropagation() (Stops Event Travel)

When you click an element inside another element, the click travels up to all parent elements (bubbling). Calling stopPropagation() tells the event: "Stop right here. Do not notify any parent elements."

Code Example:

html
<div id="outer-box" style="padding: 20px; background: lightblue;">
  Outer Box
  <button id="inner-btn">Click Me</button>
</div>
Use code with caution.
javascript
document.getElementById("outer-box").addEventListener("click", () => {
  console.log("Outer box was clicked!");
});

document.getElementById("inner-btn").addEventListener("click", (event) => {
  event.stopPropagation(); // Stops the click from reaching #outer-box
  console.log("Button was clicked!");
});
*/

console.log(null === undefined);




format for wiki page:
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
console.log(j);

j = 10;

let x;
console.log(x);

//Function hoisting

sayHello(); // Works! Outputs: "Hello!"

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

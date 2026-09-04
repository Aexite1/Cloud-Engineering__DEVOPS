# JavaScript Fundamentals: Syntax and Core Constructs

## 1. Why JavaScript Matters

JavaScript is a programming language widely used to make web pages respond to user actions and perform dynamic operations. In a typical browser-based application, it can execute on the client side and control behaviour that would otherwise require a static page.

This note introduces the main syntax building blocks needed before moving into more advanced JavaScript development.

## 2. Writing Notes Inside JavaScript

Comments allow developers to explain code without affecting program execution.

**Single-line comment**
```javascript
// This explains the statement below
```

**Multi-line comment**
```javascript
/*
This section can contain
several lines of explanation.
*/
```

## 3. Storing Values

A variable gives a program a name through which a value can be accessed. JavaScript provides three declaration keywords:

| Keyword | Main characteristic |
|---|---|
| `var` | Function-scoped and can be redeclared |
| `let` | Block-scoped and can be reassigned |
| `const` | Block-scoped and cannot be reassigned after initialization |

Example:

```javascript
var a = 5;
let b = "Hello";
const c = true;
```

For modern JavaScript, `let` and `const` are generally preferred because their block-scoping behaviour is easier to reason about.

## 4. JavaScript Data

JavaScript works with several kinds of values.

### Primitive values

Common primitive types include:

- `Number`
- `String`
- `Boolean`
- `Undefined`
- `Null`

### Composite values

Objects and arrays allow multiple values or related properties to be grouped together.

### Functions

Functions are also first-class values in JavaScript and can be stored in variables or passed to other functions.

Example:

```javascript
let num = 10;
let str = "JavaScript";
let bool = true;
let arr = [1, 2, 3];

let obj = {
    name: "John",
    age: 30
};

let func = function() {
    console.log("Hello!");
};
```

## 5. Operators

Operators allow expressions to calculate values, compare values, assign data, and combine conditions.

### Common categories

- **Arithmetic:** `+`, `-`, `*`, `/`, `%`
- **Assignment:** `=`, `+=`, `-=`, `*=`, `/=`
- **Comparison:** `==`, `===`, `!=`, `!==`, `>`, `<`, `>=`, `<=`
- **Logical:** `&&`, `||`, `!`
- **String concatenation:** `+`
- **Ternary:** `condition ? value1 : value2`

Example:

```javascript
let x = 10;
let y = 5;

let z = x + y;

let result = (x > y)
    ? "x is greater than y"
    : "x is less than or equal to y";
```

## 6. Controlling Program Flow

Control structures determine which statements execute and how many times they execute.

### Conditions

`if`, `else if`, and `else` can be used when the program needs to make a decision.

```javascript
let num = 10;

if (num > 0) {
    console.log("Positive number");
} else if (num < 0) {
    console.log("Negative number");
} else {
    console.log("Zero");
}
```

JavaScript also provides `switch` statements for selecting between multiple cases.

### Loops

Common looping structures include:

- `for`
- `while`
- `do...while`

Example:

```javascript
let i = 0;

while (i < 5) {
    console.log(i);
    i++;
}
```

## 7. Reusable Logic with Functions

A function groups instructions that can be executed whenever the function is called.

### Function declaration

```javascript
function greet(name) {
    return "Hello, " + name + "!";
}
```

### Function expression

```javascript
let greet = function(name) {
    return "Hello, " + name + "!";
};

console.log(greet("John"));
```

A function declaration can be called before its definition because of JavaScript's handling of function declarations, while a function expression assigned to a variable follows the variable's initialization rules.

## Final Notes

The concepts above form the basic syntax needed to read and write JavaScript programs. Variables and data types provide the data, operators manipulate it, control structures determine execution flow, and functions package reusable behaviour. Further practice builds on these fundamentals to support interactive web development.

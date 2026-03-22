

# Part 02 — JavaScript & TypeScript
## 500 Interview Questions for React Native Developer | 10–12 LPA
> **Structure:** 50 topics × 10 questions | **Difficulty:** 10% Easy · 50% Medium · 40% Hard
> **Each question includes:** Answer · Code Snippet · Interview Trap/Gotcha · Resume Link (where applicable)

---

## TABLE OF CONTENTS
1. var / let / const & Scoping
2. Hoisting
3. Closures
4. Execution Context & Call Stack
5. Event Loop & Concurrency Model
6. Microtasks vs Macrotasks
7. Callbacks & Callback Hell
8. Promises
9. async / await
10. Error Handling in Async Code
11. Prototype & Prototype Chain
12. `this` Keyword
13. Arrow Functions vs Regular Functions
14. Currying & Partial Application
15. Higher-Order Functions
16. Array Methods (map/filter/reduce/flat)
17. Object Methods & Destructuring
18. Spread & Rest Operators
19. Modules (CommonJS vs ES Modules)
20. Classes & Inheritance
21. Symbols & Iterators
22. Generators
23. WeakMap / WeakSet / WeakRef
24. Memory Management & Garbage Collection
25. Immutability Patterns
26. TypeScript — Types vs Interfaces
27. TypeScript — Generics
28. TypeScript — Utility Types
29. TypeScript — Decorators
30. TypeScript — Strict Mode & Type Guards
31. TypeScript — Enums
32. TypeScript — Mapped & Conditional Types
33. TypeScript — Declaration Files (.d.ts)
34. TypeScript — Module Augmentation
35. TypeScript — with React Native (Props, Hooks, Redux)
36. Functional Programming Concepts
37. Design Patterns in JS
38. Event Delegation & DOM Events
39. Debounce & Throttle
40. Deep Clone / Shallow Clone
41. Coercion & Type Conversion
42. Truthy / Falsy & Short-circuit Evaluation
43. Regular Expressions in JS
44. Date & Time in JS
45. Error Types & Custom Errors
46. JSON — parse, stringify, edge cases
47. ES2020–ES2024 New Features
48. Performance Optimisation in JS
49. Testing — Jest (Unit / Integration)
50. Secure Coding Practices in JS

---

---
## Topic 01 — var / let / const & Scoping

### Q1 · Easy
**What is the difference between `var`, `let`, and `const` in JavaScript?**

**Answer:**
`var` is function-scoped (or globally scoped), can be re-declared, and is hoisted with an `undefined` value. `let` and `const` are block-scoped, not re-declarable in the same scope, and are hoisted but kept in a **Temporal Dead Zone (TDZ)** until their declaration line. `const` additionally prevents re-assignment (though the object it points to can still be mutated).

```js
var x = 1;
{
  var x = 2; // same variable — re-declares
}
console.log(x); // 2

let y = 1;
{
  let y = 2; // new block-scoped variable
}
console.log(y); // 1

const obj = { a: 1 };
obj.a = 99; // ✅ mutation allowed
obj = {};   // ❌ TypeError: Assignment to constant variable
```

**Trap:** Many candidates say "`const` is immutable." It is not — only the binding is constant, not the value.

**Resume Link:** Every Redux reducer you wrote uses `const` for state — understanding const mutation is essential for correct immutable updates.

---

### Q2 · Medium
**What is the Temporal Dead Zone (TDZ) and when does it occur?**

**Answer:**
TDZ is the period between when a `let`/`const` variable is hoisted and when it is initialised. Accessing the variable during TDZ throws a `ReferenceError`. `var` does not have TDZ — it initialises to `undefined` on hoist.

```js
console.log(a); // ✅ undefined (var is hoisted + initialised)
var a = 5;

console.log(b); // ❌ ReferenceError: Cannot access 'b' before initialization
let b = 5;
```

**Trap:** Interviewers often show code where `let` is used in a class field or block above its declaration — asking "what happens?" Answer: TDZ error, not `undefined`.

**Resume Link:** In ERP SKAVO, Redux selectors and initial state are const-declared at the top of each slice — TDZ awareness prevents subtle initialisation bugs.

---

### Q3 · Medium
**Explain how `var` inside a loop creates closure bugs, and how `let` fixes it.**

**Answer:**
`var` is function-scoped, so all loop iterations share the same variable. By the time callbacks run, the loop has completed and `i` holds the final value. `let` creates a new binding per iteration, capturing the correct value.

```js
// Bug with var
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Prints: 3, 3, 3

// Fixed with let
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Prints: 0, 1, 2
```

**Trap:** Interviewers ask the `var` version first to check if you know *why* it prints 3,3,3 — not just that it's wrong.

**Resume Link:** Redux-Saga event channels and async effect loops in your ERP app could surface this exact bug if `var` is mistakenly used.

---

### Q4 · Medium
**Can you re-declare a `let` variable in the same scope? What about a nested scope?**

**Answer:**
No, re-declaring `let` in the same scope throws a `SyntaxError`. However, a nested block creates a new scope where a same-named `let` is allowed (it shadows the outer variable).

```js
let x = 1;
let x = 2; // ❌ SyntaxError: Identifier 'x' has already been declared

{
  let x = 99; // ✅ shadows outer x — new block scope
  console.log(x); // 99
}
console.log(x); // 1
```

**Trap:** Candidates confuse "can't re-declare" with "can't shadow" — they are different things.

---

### Q5 · Hard
**What happens when you use `const` with arrays and objects inside React Native's `useReducer`?**

**Answer:**
`const` prevents re-assignment of the variable binding, but Redux/useReducer patterns work because reducers *return new objects/arrays* (new bindings) rather than mutating the `const`. If you mutate in place and return the same reference, React won't detect a change.

```js
// WRONG — mutation, same reference
const reducer = (state, action) => {
  state.count += 1; // mutates — const binding unchanged
  return state;     // same ref → React bails out, no re-render
};

// CORRECT — new reference
const reducer = (state, action) => {
  return { ...state, count: state.count + 1 }; // new obj → re-render triggers
};
```

**Trap:** Interviewers ask why a `useReducer`-powered component doesn't re-render — mutation of state is the most common culprit.

**Resume Link:** Direct parallel to every Redux reducer you wrote in your ERP and Debt Relief India apps.

---

### Q6 · Hard
**How does block scoping interact with `switch` statements? What is the common bug?**

**Answer:**
All `case` blocks in a `switch` share the same scope (the `switch` block). Declaring `let` or `const` with the same name in two different cases causes a `SyntaxError`. The fix is to wrap each `case` body in curly braces `{}` to create a new block scope.

```js
// BUG
switch (action.type) {
  case 'A':
    let result = 1; // declared here
    break;
  case 'B':
    let result = 2; // ❌ SyntaxError: already declared
    break;
}

// FIX
switch (action.type) {
  case 'A': {
    let result = 1; // own block scope
    break;
  }
  case 'B': {
    let result = 2; // ✅ separate scope
    break;
  }
}
```

**Trap:** This is very common in Redux reducers written as switch statements — a frequent real-world bug.

**Resume Link:** Directly relevant to your Redux reducers in all three of your production apps.

---

### Q7 · Hard
**What is the difference between global scope, module scope, and function scope in a React Native context?**

**Answer:**
- **Global scope** — variables accessible everywhere (e.g., `global.myVar = ...` in RN). Avoid polluting it.
- **Module scope** — variables declared at the top level of a JS module file. Only accessible within that module unless exported.
- **Function scope** — variables inside a function, inaccessible outside.

```js
// global.js — module scope
const API_BASE = 'https://api.example.com'; // module-scoped, not global

// Accidental global in RN
function setup() {
  leakyVar = 'oops'; // no const/let/var — becomes global (strict mode throws)
}

// RN uses strict mode via Hermes — leakyVar throws ReferenceError
```

**Trap:** In older React Native (pre-Hermes), accidental globals would silently work. With Hermes, they throw. Know the difference.

**Resume Link:** Your adoption of Hermes in the ERP app means strict mode is active — this matters.

---

### Q8 · Medium
**What is IIFE and why was it used before ES Modules?**

**Answer:**
An **Immediately Invoked Function Expression** creates a private scope to avoid polluting the global namespace, which was critical before ES Modules introduced module scope.

```js
(function () {
  var privateVar = 'only here';
  console.log(privateVar); // works
})();

console.log(privateVar); // ❌ ReferenceError
```

**Trap:** IIFEs are rare in modern RN code, but interviewers use them to test scope understanding. Know the pattern even if you don't write it daily.

---

### Q9 · Hard
**If you declare a variable with `var` inside an `if` block, where does it get hoisted to?**

**Answer:**
`var` hoists to the top of the *nearest enclosing function* (or global scope). The `if` block is not a function scope, so the `var` leaks out of the block.

```js
function test() {
  if (true) {
    var leaked = 'I escaped the if block';
  }
  console.log(leaked); // 'I escaped the if block'
}
test();
```

**Trap:** Candidates say "hoisted to the top of the block" — wrong. `var` does not respect block scope.

---

### Q10 · Hard
**What does "scope chain" mean and how does the JS engine resolve variable lookups?**

**Answer:**
When a variable is referenced, the JS engine first looks in the current scope, then moves up to the enclosing scope, and so on until the global scope. This chain of scopes is the **scope chain**. It is determined *lexically* (at write time), not dynamically.

```js
const global = 'global';

function outer() {
  const outerVar = 'outer';
  function inner() {
    const innerVar = 'inner';
    console.log(innerVar);  // found in current scope
    console.log(outerVar);  // found in outer scope
    console.log(global);    // found in global scope
    console.log(missing);   // ❌ ReferenceError — not in chain
  }
  inner();
}
outer();
```

**Trap:** Scope chain is *lexical* — it doesn't matter where the function is *called*, only where it is *defined*.

---

---
## Topic 02 — Hoisting

### Q11 · Easy
**What is hoisting in JavaScript?**

**Answer:**
Hoisting is the JS engine's behaviour of moving declarations (not initialisations) to the top of their scope before execution. `var` declarations are hoisted and initialised to `undefined`. Function declarations are fully hoisted (name + body). `let`/`const` are hoisted but not initialised (TDZ).

```js
console.log(greet()); // 'Hello' — function declaration fully hoisted
console.log(x);       // undefined — var hoisted, not initialised
console.log(y);       // ❌ ReferenceError — let in TDZ

function greet() { return 'Hello'; }
var x = 5;
let y = 10;
```

**Trap:** "Hoisting moves code to the top" — not literally. The engine processes declarations in a compilation phase before executing code.

---

### Q12 · Medium
**Are function expressions hoisted?**

**Answer:**
Function *expressions* assigned to `var` are hoisted as `undefined` — the function body is not available until the assignment runs. Function *declarations* are fully hoisted.

```js
console.log(fn1()); // ✅ 'declaration' — fully hoisted
console.log(fn2);   // undefined (var is hoisted, not the function)
console.log(fn2()); // ❌ TypeError: fn2 is not a function

function fn1() { return 'declaration'; }
var fn2 = function () { return 'expression'; };
```

**Trap:** Calling a `var` function expression before declaration gives `TypeError` (not `ReferenceError`) because `fn2` is `undefined`, and calling `undefined()` is a type error.

---

### Q13 · Medium
**What happens when a function and a `var` have the same name?**

**Answer:**
Function declarations take priority over `var` declarations during hoisting. The function wins the initial binding, but if the `var` is later assigned, that assignment overwrites the function.

```js
console.log(typeof foo); // 'function' — function wins
var foo = 'bar';
console.log(typeof foo); // 'string' — assignment overwrites
function foo() {}
```

**Trap:** Many candidates say the `var` shadows the function — the opposite is true for the hoisted phase.

---

### Q14 · Medium
**Does hoisting work inside arrow functions?**

**Answer:**
Arrow functions do not have their own hoisting behaviour — they follow the same rules as regular variable declarations. Arrow functions assigned to `const`/`let` are in TDZ; assigned to `var`, they're `undefined` until assigned.

```js
console.log(arrow); // undefined (var hoisting)
var arrow = () => 'hi';

console.log(arrow2); // ❌ ReferenceError (TDZ)
const arrow2 = () => 'hi';
```

**Trap:** Interviewers ask "are arrow functions hoisted?" — the variable binding is hoisted (by its declaration keyword), but the function body is not available early.

---

### Q15 · Hard
**How does hoisting behave in class declarations?**

**Answer:**
Class declarations are hoisted but *not initialised* — they remain in TDZ, unlike function declarations. This means you cannot use a class before its declaration.

```js
const obj = new Animal(); // ❌ ReferenceError: Cannot access 'Animal' before initialization
class Animal {}

// Compare with function:
const a = makeAnimal(); // ✅ works — function hoisted fully
function makeAnimal() { return {}; }
```

**Trap:** People assume classes behave like function declarations. They behave more like `let` — TDZ applies.

**Resume Link:** You use class-based patterns implicitly in React components (PureComponent) and TypeScript classes in your ERP app.

---

### Q16 · Hard
**What is the output of this code? Explain step by step.**

```js
var x = 1;
function test() {
  console.log(x);
  var x = 2;
  console.log(x);
}
test();
console.log(x);
```

**Answer:**
Output: `undefined`, `2`, `1`

- Inside `test()`, `var x` is hoisted to the top of the function. At `console.log(x)` it's `undefined` (hoisted but not yet assigned).
- After `var x = 2`, the local `x` is `2`.
- Outside, the global `x` remains `1` — the function's `var x` is local.

**Trap:** Candidates say the first log prints `1` (thinking it reads the global). It doesn't — the local `var x` shadows the global from the moment of hoisting.

---

### Q17 · Hard
**Can you explain hoisting in the context of named function expressions?**

**Answer:**
In a named function expression, the name is only available *inside* the function body, not in the outer scope. The outer variable binding follows normal `var`/`let`/`const` hoisting rules.

```js
var foo = function bar() {
  console.log(bar); // ✅ accessible inside
};
console.log(foo); // ✅ the function
console.log(bar); // ❌ ReferenceError — not in outer scope
```

**Trap:** The function name `bar` is scoped to the function itself — useful for recursion but invisible outside.

---

### Q18 · Medium
**Explain hoisting with `let` inside a block in a loop.**

**Answer:**
Each iteration of a `for` loop with `let` creates a fresh binding — this is the mechanism that fixes the classic closure-in-loop bug. The engine effectively re-runs the `let` declaration for each iteration.

```js
const funcs = [];
for (let i = 0; i < 3; i++) {
  funcs.push(() => i); // each closure captures its own `i`
}
console.log(funcs[0]()); // 0
console.log(funcs[1]()); // 1
console.log(funcs[2]()); // 2
```

**Trap:** This only works because of `let`'s per-iteration block scope. `var` would give `3, 3, 3`.

---

### Q19 · Hard
**How does the JavaScript engine's two-phase execution relate to hoisting?**

**Answer:**
JS engines use two phases: (1) **Compilation/Creation phase** — scans the code, creates the scope, and registers all declarations (sets up memory for `var` as `undefined`, registers function declarations fully, puts `let`/`const` in TDZ). (2) **Execution phase** — runs code line by line, assigns values. Hoisting is a side effect of the creation phase.

```js
// What the engine "sees" after creation phase:
// var a = undefined;
// function b() { return 1; }
// (let c stays in TDZ)

console.log(a); // undefined
console.log(b()); // 1
console.log(c); // ReferenceError

var a = 5;
function b() { return 1; }
let c = 10;
```

**Trap:** Hoisting is not code movement — it's a mental model for the creation phase. Real engines don't physically move code.

---

### Q20 · Hard
**What is the output and why?**

```js
function outer() {
  function inner() { return x; }
  var x = 10;
  return inner();
}
console.log(outer());
```

**Answer:**
Output: `10`. When `inner` is *defined*, it closes over the scope of `outer`. When `inner` is *called* (via `return inner()`), `x` has already been assigned `10`. So it returns `10`, not `undefined`. If `inner` were called before `var x = 10`, it would return `undefined` (hoisted but not yet assigned).

**Trap:** Candidates confuse when `inner` is defined vs when it is called. Closures capture the *variable binding*, not the value at definition time.

---

---
## Topic 03 — Closures

### Q21 · Easy
**What is a closure in JavaScript?**

**Answer:**
A closure is a function that retains access to its outer (enclosing) scope even after the outer function has returned. The inner function "closes over" the variables in the lexical environment where it was created.

```js
function counter() {
  let count = 0;
  return function increment() {
    count++;
    return count;
  };
}
const inc = counter();
console.log(inc()); // 1
console.log(inc()); // 2
console.log(inc()); // 3
// count is alive in memory because inc holds a reference to it
```

**Trap:** "Closures are functions inside functions" — partially right but incomplete. The key is the *retained scope*.

---

### Q22 · Medium
**How do closures enable the module pattern in JavaScript?**

**Answer:**
Closures let you create private state and expose only a public API — the foundation of the module pattern used before ES Modules.

```js
const BankAccount = (function () {
  let balance = 0; // private

  return {
    deposit(amount) { balance += amount; },
    withdraw(amount) { balance -= amount; },
    getBalance() { return balance; }
  };
})();

BankAccount.deposit(100);
console.log(BankAccount.getBalance()); // 100
console.log(balance); // ❌ ReferenceError — private
```

**Trap:** Interviewers ask "how would you replicate private class fields before they existed?" — closures are the answer.

---

### Q23 · Medium
**Explain a practical use of closures in React Native — for example, in event handlers.**

**Answer:**
In RN, closures appear in `useCallback`, `useMemo`, and plain function definitions inside components. A handler defined inside a component closes over its props and state.

```js
function PaymentButton({ userId, amount }) {
  const handlePay = () => {
    // closes over userId and amount from the outer scope
    processPayment(userId, amount);
  };

  return Pay;
}
```

**Trap:** Stale closures — if `useCallback` has an empty dependency array, the handler closes over the *initial* values of `userId` and `amount` and will never see updates.

**Resume Link:** Your Razorpay payment handler in Debt Relief India and Wink app both use this pattern.

---

### Q24 · Hard
**What is a "stale closure" in React hooks and how do you fix it?**

**Answer:**
A stale closure occurs when a function captures an old version of a state or prop value because it was memoised without including that value in its dependency array. The function runs with outdated data.

```js
// STALE CLOSURE BUG
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1); // always uses initial count=0
    }, 1000);
    return () => clearInterval(id);
  }, []); // missing count in deps — stale!

  return {count}; // always shows 1
}

// FIX — functional updater form
useEffect(() => {
  const id = setInterval(() => {
    setCount(prev => prev + 1); // uses latest state, no closure needed
  }, 1000);
  return () => clearInterval(id);
}, []);
```

**Trap:** Using functional updater form (`prev =>`) sidesteps stale closure for state. For props, you must add them to deps.

**Resume Link:** Real-time data flows in your ERP app (attendance, notifications) use `useEffect` — stale closures would cause incorrect data displayed.

---

### Q25 · Hard
**How do closures cause memory leaks and how do you prevent them?**

**Answer:**
If a closure holds a reference to a large object, that object cannot be garbage collected as long as the closure exists. In React Native, event listeners and timers that close over component state and are never cleaned up cause leaks.

```js
// Memory leak: closure holds reference to large data
function setup() {
  const bigData = new Array(1000000).fill('data');
  return function leaky() {
    console.log(bigData.length); // bigData can never be GC'd
  };
}
const fn = setup(); // bigData stays alive

// Fix: null out reference when done
let fn2 = setup();
fn2();
fn2 = null; // now bigData can be garbage collected
```

**Trap:** In React Native, forgetting to remove event listeners in `useEffect` cleanup is a closure-based memory leak.

**Resume Link:** You use push notification listeners and WebSocket subscriptions in your apps — cleanup is critical.

---

### Q26 · Hard
**Explain how `useCallback` and `useMemo` relate to closures.**

**Answer:**
Both hooks memoize a value/function across renders. They create a closure over the values listed in the dependency array. If a dep changes, the hook creates a new closure with the updated values; otherwise it returns the cached one.

```js
const memoizedFn = useCallback(() => {
  // This closure captures `userId` — a new fn is created only when userId changes
  fetchUser(userId);
}, [userId]);

const memoizedVal = useMemo(() => {
  return heavyComputation(data);
}, [data]);
```

**Trap:** `useCallback(() => fn, [])` with empty deps creates a closure over initial state — safe only if the function doesn't need fresh state.

**Resume Link:** You use Redux selectors + `useCallback` in your ERP 200+ API screen flows to avoid unnecessary re-renders.

---

### Q27 · Medium
**What is the difference between a closure and a pure function?**

**Answer:**
A **pure function** depends only on its arguments and produces no side effects — it has no hidden state. A **closure** captures external state from its lexical scope — it is inherently impure if that state can change, but useful for encapsulation.

```js
// Pure function — no external state
const add = (a, b) => a + b;

// Closure — captures external state
function makeAdder(x) {
  return (y) => x + y; // closes over x
}
const add5 = makeAdder(5);
add5(3); // 8 — result depends on captured x, not just argument
```

**Trap:** Interviewers test whether you know Redux reducers must be *pure* — they must not close over mutable external state.

---

### Q28 · Hard
**How do closures work with asynchronous code — specifically with setTimeout in a loop?**

**Answer:**
`var` in a loop creates one shared binding; all async callbacks (setTimeout) close over the same variable and see its final value. `let` creates a new binding per iteration, so each callback gets the correct value.

```js
// Classic trap
for (var i = 0; i < 5; i++) {
  setTimeout(() => console.log(i), i * 100); // 5,5,5,5,5
}

// Fix 1: let
for (let i = 0; i < 5; i++) {
  setTimeout(() => console.log(i), i * 100); // 0,1,2,3,4
}

// Fix 2: IIFE to capture i per iteration (legacy)
for (var i = 0; i < 5; i++) {
  (function (j) {
    setTimeout(() => console.log(j), j * 100);
  })(i); // 0,1,2,3,4
}
```

**Trap:** The IIFE fix is asked to test deep understanding — know both the cause and both fix strategies.

---

### Q29 · Hard
**What is a closure-based memoization function? Write one from scratch.**

**Answer:**
Memoization uses a closure to maintain a private cache. The returned function checks the cache before computing.

```js
function memoize(fn) {
  const cache = {}; // private cache — closure

  return function (...args) {
    const key = JSON.stringify(args);
    if (key in cache) {
      console.log('cache hit');
      return cache[key];
    }
    cache[key] = fn(...args);
    return cache[key];
  };
}

const factorial = memoize(function f(n) {
  return n <= 1 ? 1 : n * f(n - 1);
});

console.log(factorial(5)); // computed
console.log(factorial(5)); // cache hit
```

**Trap:** Using `args.toString()` as a key is fragile — `JSON.stringify` handles objects/arrays but not circular structures.

**Resume Link:** Performance-sensitive ERP screens with heavy computation can benefit from this pattern.

---

### Q30 · Hard
**Explain how Redux-Saga uses closures internally in its channel/buffer implementation.**

**Answer:**
Redux-Saga's `channel` and `eventChannel` use closure to hold internal buffers and subscriber lists. When you call `take(channel)`, the saga suspends and a callback closes over the continuation (next step), stored inside the channel's closure until `put` is called.

```js
// Simplified mental model of eventChannel
function createChannel() {
  let subscribers = []; // captured in closure

  return {
    take(cb) {
      subscribers.push(cb); // stores continuation
    },
    emit(value) {
      const sub = subscribers.shift();
      if (sub) sub(value); // resumes saga
    }
  };
}
```

**Trap:** Understanding this helps debug why sagas sometimes miss events when a channel buffer overflows — the continuation is simply dropped.

**Resume Link:** You use Redux-Saga in all three production apps — understanding channels explains real-time data flow behaviour.

---

---
## Topic 04 — Execution Context & Call Stack

### Q31 · Easy
**What is an execution context in JavaScript?**

**Answer:**
An execution context is the environment in which JavaScript code is evaluated and executed. It contains: the **Variable Environment** (bindings for vars/functions), the **Lexical Environment** (scope chain), and the value of **`this`**. Three types exist: Global, Function, and Eval execution contexts.

```js
// Global context is created first
var globalVar = 'global';

function foo() {
  // New function execution context created when foo() is called
  var localVar = 'local';
  console.log(globalVar); // accesses global via scope chain
}
foo();
```

**Trap:** People conflate execution context with scope. Scope is the *visibility of variables*; execution context is the broader environment (includes `this`, LexEnv, VarEnv).

---

### Q32 · Medium
**What is the call stack and what happens when it overflows?**

**Answer:**
The call stack is a LIFO data structure that tracks active function calls. Each call pushes a frame; each return pops it. Infinite recursion causes a **stack overflow** (Maximum call stack size exceeded).

```js
function recurse() {
  return recurse(); // no base case
}
recurse(); // ❌ RangeError: Maximum call stack size exceeded

// Fix: add base case
function safeRecurse(n) {
  if (n === 0) return 'done';
  return safeRecurse(n - 1);
}
```

**Trap:** Stack overflow is a `RangeError`, not a `TypeError` or `ReferenceError` — know the error type.

---

### Q33 · Medium
**What is the difference between the creation phase and execution phase of an execution context?**

**Answer:**
During the **creation phase**, the engine sets up the Variable Environment (hoists `var`, registers function declarations, puts `let`/`const` in TDZ), sets up the scope chain, and determines `this`. During the **execution phase**, code runs line by line and variables get their assigned values.

```js
// Creation phase sets up:
// this = window/global
// x → undefined (var)
// greet → function (declaration fully hoisted)
// y → TDZ (let)

// Execution phase runs:
console.log(x);       // undefined
console.log(greet()); // 'hi'
console.log(y);       // ReferenceError

var x = 10;
function greet() { return 'hi'; }
let y = 20;
```

**Trap:** The creation phase is conceptual — the engine doesn't literally run code twice, it processes declarations in a preliminary scan.

---

### Q34 · Hard
**What happens to the call stack when async code (setTimeout) is encountered?**

**Answer:**
When `setTimeout` is encountered, the callback is handed off to the Web API / Timer API (outside the JS engine). The call stack continues without blocking. When the timer expires, the callback is placed in the **macrotask queue**. The event loop picks it up only when the call stack is empty.

```js
console.log('1');          // pushed to stack, runs
setTimeout(() => {
  console.log('3');        // goes to timer API, callback queued later
}, 0);
console.log('2');          // pushed to stack, runs
// Call stack empties → event loop pushes setTimeout callback
// Output: 1, 2, 3
```

**Trap:** `setTimeout(..., 0)` does not run immediately — it runs after the current call stack clears and all microtasks are flushed.

---

### Q35 · Hard
**Explain tail call optimisation (TCO) — does JavaScript support it?**

**Answer:**
TCO allows the engine to reuse the current stack frame for a tail call (a function call as the last operation), preventing stack growth. ES6 specifies TCO, but only **Safari/JavaScriptCore** implements it fully. V8 (used in Node.js/Chrome) removed TCO; Hermes (used in React Native) does not implement it either.

```js
// Tail call — last operation is the recursive call
function factorial(n, acc = 1) {
  if (n <= 1) return acc;
  return factorial(n - 1, n * acc); // tail position
}
// In Safari: no stack overflow even for large n
// In Hermes/V8: still limited by stack size
```

**Trap:** Interviewers may ask if RN/Hermes supports TCO. The answer is no — Hermes focuses on startup performance, not TCO.

**Resume Link:** Relevant to your Hermes adoption in the ERP app.

---

### Q36 · Medium
**What is the difference between stack frames for function calls vs arrow functions?**

**Answer:**
Both push frames onto the call stack when invoked. The difference is not in stack frame handling but in context: arrow functions don't create their own `this`, `arguments`, or `super` bindings — they inherit from the enclosing context. Stack overflow behaviour is identical.

```js
function regular() {
  console.log(this);       // depends on call site
  console.log(arguments);  // own arguments object
}

const arrow = () => {
  console.log(this);       // inherits outer this
  // arguments not available — ReferenceError if accessed
};
```

**Trap:** Interviewers ask if arrow functions are "lighter" on the stack — they aren't meaningfully different in terms of stack frame size.

---

### Q37 · Hard
**What does the execution context look like for a Redux-Saga generator function when it is suspended?**

**Answer:**
When a generator yields (e.g., `yield take(ACTION)`), the generator's execution context is *paused but not popped from the stack*. The generator's local variables, scope, and position are preserved in the generator object's internal state. The saga middleware resumes it by calling `.next(value)`.

```js
function* watchLogin() {
  while (true) {
    const action = yield take('LOGIN'); // paused here — context preserved
    yield call(loginSaga, action.payload); // resumed when action dispatched
  }
}
// The while(true) loop is safe because the generator suspends at each yield
```

**Trap:** Generators do not block the event loop — they suspend, which is fundamentally different from a blocking while loop.

**Resume Link:** Direct to your Redux-Saga usage in ERP SKAVO and Debt Relief India.

---

### Q38 · Medium
**How many execution contexts exist when this runs?**

```js
const a = () => { const b = () => {}; b(); };
a();
```

**Answer:**
Three: (1) Global execution context (always exists), (2) `a`'s execution context when `a()` is called, (3) `b`'s execution context when `b()` is called inside `a`. At peak (inside `b`), the stack has: Global → a → b.

**Trap:** Arrow functions still create execution contexts (stack frames) even though they don't create their own `this`.

---

### Q39 · Hard
**What is the difference between the Lexical Environment and the Variable Environment in an execution context?**

**Answer:**
Both are components of an execution context. In ES6+:
- **Variable Environment** stores `var` bindings and function declarations (function-scoped).
- **Lexical Environment** stores `let`, `const`, and function scope for block-scoped identifiers. It also holds the reference to the outer environment (enabling the scope chain).

In practice, the distinction matters for how block scoping works — a new Lexical Environment is created for each block `{}`, but the Variable Environment is tied to the function.

```js
function foo() {
  var x = 1; // Variable Environment
  let y = 2; // Lexical Environment
  {
    let z = 3; // New Lexical Environment for this block
    console.log(x, y, z); // 1, 2, 3
  }
  console.log(z); // ❌ ReferenceError
}
```

**Trap:** Interviewers ask this to see if you understand *why* `let`/`const` are block-scoped at the spec level.

---

### Q40 · Hard
**What is the "this" value in the global execution context in Node.js vs a browser vs React Native?**

**Answer:**
- **Browser global context:** `this === window`
- **Node.js module scope:** `this === module.exports` (empty object `{}`)
- **Node.js global scope:** `this === global`
- **React Native (Hermes):** `this === globalThis` (similar to browser `window`, but is the RN global object)

```js
// In React Native (Hermes)
console.log(this === globalThis); // true in global scope
console.log(typeof global);       // 'object'

// Common RN pattern
global.myConfig = { apiUrl: 'https://...' }; // sets on RN global
```

**Trap:** RN developers sometimes use `global.X` for cross-module singletons — this works but is an anti-pattern; use a module or Redux store instead.

**Resume Link:** Your ERP app's global error boundary and JWT token management may use global-level utilities.

---

---
## Topic 05 — Event Loop & Concurrency Model

### Q41 · Easy
**What is the JavaScript event loop?**

**Answer:**
JavaScript is single-threaded. The event loop is the mechanism that enables non-blocking async behaviour. It continuously checks: if the **call stack** is empty, take the next task from the **task queues** (microtask queue first, then macrotask queue) and push it onto the stack.

```
Call Stack → empty?
  Yes → flush all Microtasks (Promises, queueMicrotask)
      → pick one Macrotask (setTimeout, setInterval, I/O)
      → repeat
```

**Trap:** "JavaScript is async" — false. JS is single-threaded and synchronous by default. The async illusion comes from the event loop + browser/Node APIs.

---

### Q42 · Medium
**What is the difference between the task queue and the job queue (microtask queue)?**

**Answer:**
- **Macrotask queue (Task Queue):** `setTimeout`, `setInterval`, `setImmediate`, I/O, UI rendering.
- **Microtask queue (Job Queue):** Resolved Promises (`.then`/`.catch`), `async/await` continuations, `queueMicrotask`, `MutationObserver`.

**Priority:** After each macrotask, ALL microtasks are flushed before the next macrotask runs.

```js
console.log('1');
setTimeout(() => console.log('4'), 0); // macrotask
Promise.resolve().then(() => console.log('3')); // microtask
console.log('2');
// Output: 1, 2, 3, 4
```

**Trap:** Many candidates say "3, 4" — wrong. Microtasks flush entirely before any macrotask runs.

---

### Q43 · Hard
**What is the output of this code? Explain in detail.**

```js
async function main() {
  console.log('A');
  await Promise.resolve();
  console.log('B');
}
console.log('1');
main();
console.log('2');
```

**Answer:**
Output: `1`, `A`, `2`, `B`

- `console.log('1')` runs synchronously.
- `main()` is called: `console.log('A')` runs synchronously. `await Promise.resolve()` suspends `main` and schedules the continuation (`console.log('B')`) as a microtask.
- Back in the outer context: `console.log('2')` runs.
- Call stack empties → microtask queue flushes → `console.log('B')` runs.

**Trap:** People say `1, A, B, 2` — they forget `await` immediately suspends the async function, returning control to the caller.

---

### Q44 · Hard
**How does the React Native JS thread relate to the event loop?**

**Answer:**
RN runs JavaScript on a single JS thread (the JavaScriptCore or Hermes runtime). The event loop runs on this thread. UI rendering, native module calls, and network requests happen on *separate native threads*. The JS thread receives callbacks/events from native threads via the bridge (or JSI in the new arch), which are processed through the event loop.

```
Native Thread (UI/Network)
    ↓ callback via Bridge/JSI
JS Thread Event Loop
    ↓ processes callback
JS Code runs (setState, dispatch, etc.)
    ↓
Bridge/JSI sends update back to native
```

**Trap:** Heavy synchronous JS (complex reducers, large JSON parsing) blocks the JS thread and causes frame drops — the UI thread is separate but depends on JS for logic.

**Resume Link:** Your 200+ API app benefits from non-blocking Redux-Saga effects — blocking the JS thread would freeze the ERP screens.

---

### Q45 · Hard
**What happens when you have nested Promises inside a setTimeout?**

```js
setTimeout(() => {
  console.log('timeout');
  Promise.resolve().then(() => console.log('promise inside timeout'));
}, 0);
Promise.resolve().then(() => console.log('promise outside'));
```

**Answer:**
Output: `promise outside`, `timeout`, `promise inside timeout`

1. `setTimeout` callback goes to macrotask queue.
2. `Promise.resolve().then(...)` (outside) goes to microtask queue.
3. Stack empty → flush microtasks → `promise outside`.
4. Pick next macrotask → `timeout` logs.
5. During that macrotask, a new microtask is queued.
6. After macrotask finishes, flush microtasks → `promise inside timeout`.

**Trap:** Microtasks created *during* a macrotask are flushed before the *next* macrotask — not deferred until next iteration.

---

### Q46 · Hard
**What is `queueMicrotask` and when would you use it in React Native?**

**Answer:**
`queueMicrotask(fn)` explicitly schedules a function as a microtask — runs after the current synchronous code but before any macrotasks. Useful when you want something to run "as soon as possible" without a setTimeout(0) delay.

```js
console.log('sync start');

queueMicrotask(() => {
  console.log('microtask'); // runs before any setTimeout
});

setTimeout(() => console.log('macrotask'), 0);

console.log('sync end');
// Output: sync start, sync end, microtask, macrotask
```

**Trap:** `queueMicrotask` is available in Hermes (RN ≥ 0.64). Before that, `Promise.resolve().then(fn)` was the microtask equivalent.

**Resume Link:** For latency-sensitive operations in your fintech app (e.g., EMI calculation feedback), microtask scheduling ensures UI feedback happens before the next render cycle.

---

### Q47 · Medium
**What is "render-blocking" in the context of React Native's JS thread?**

**Answer:**
RN's JS thread handles both JavaScript logic and the bridge communication. Heavy synchronous JS (large loops, complex computations) blocks the thread, delaying processing of UI events and native callbacks. This manifests as dropped frames or unresponsive touch events.

```js
// Render-blocking — freezes JS thread
function heavySync() {
  let result = 0;
  for (let i = 0; i < 10_000_000; i++) result += i;
  return result;
}

// Better — defer heavy work
setTimeout(() => {
  // runs in next macrotask, allowing UI events to be processed between
  heavySync();
}, 0);

// Best — use InteractionManager in RN
InteractionManager.runAfterInteractions(() => heavySync());
```

**Trap:** `setTimeout(fn, 0)` helps but is not perfect — `InteractionManager.runAfterInteractions` is the RN-idiomatic way.

**Resume Link:** Your ERP app's 100+ screen app with attendance/payroll data — heavy data processing on the JS thread would hurt UX.

---

### Q48 · Hard
**What is the difference between `process.nextTick` (Node) and `Promise.then` in the microtask queue?**

**Answer:**
Both are microtasks, but `process.nextTick` callbacks are processed in a dedicated "nextTick queue" that runs *before* the Promise microtask queue. This is a Node.js-specific ordering.

```js
// Node.js only
Promise.resolve().then(() => console.log('Promise'));
process.nextTick(() => console.log('nextTick'));
// Output: nextTick, Promise (nextTick queue drains first)
```

**Trap:** This is Node.js only — React Native (Hermes) does not have `process.nextTick`. Mentioning it in an RN context shows breadth of knowledge.

---

### Q49 · Hard
**How does the event loop handle long-running Promises in React Native?**

**Answer:**
A long-running Promise (e.g., a saga waiting for a network response) does not block the event loop — it's just a pending microtask continuation. The `.then` callback enters the queue only when the Promise resolves. During the wait, the JS thread is free to handle other events.

```js
async function fetchData() {
  const data = await fetch('https://api.example.com/data');
  // While fetch is pending, the JS thread processes other events
  // The continuation (code after await) runs when fetch resolves
  return data.json();
}
```

**Trap:** The *fetch* itself is handled by a native thread — the JS side just has a Promise pending. The JS thread is not blocked.

**Resume Link:** Your 35% faster API performance (from your resume) is partly thanks to non-blocking async patterns in Redux-Saga.

---

### Q50 · Hard
**What is the relationship between the event loop and React Native's `InteractionManager`?**

**Answer:**
`InteractionManager` defers work until animations and interactions have completed. It uses the macrotask queue (via `setTimeout`) internally to ensure that heavy work doesn't interfere with in-flight animations. It's a higher-level API built on top of the event loop.

```js
import { InteractionManager } from 'react-native';

function MyComponent() {
  useEffect(() => {
    const task = InteractionManager.runAfterInteractions(() => {
      // Heavy data processing — runs after current animation/interaction
      processLargeDataset();
    });
    return () => task.cancel();
  }, []);
}
```

**Trap:** `InteractionManager` does not use microtasks — it schedules work as a macrotask after interactions. Don't use it for time-critical microtask-level operations.

**Resume Link:** Relevant to your ERP app's screen transitions with complex data loading.

---

---
## Topic 06 — Microtasks vs Macrotasks

### Q51 · Easy
**Name three microtasks and three macrotasks in JavaScript.**

**Answer:**
- **Microtasks:** `Promise.then/catch/finally`, `async/await` continuations, `queueMicrotask`, `MutationObserver`
- **Macrotasks:** `setTimeout`, `setInterval`, `setImmediate` (Node.js), I/O callbacks, `MessageChannel`

```js
// Microtask
Promise.resolve().then(() => console.log('micro'));

// Macrotask
setTimeout(() => console.log('macro'), 0);

// Output: micro, macro (microtasks always before macrotasks)
```

**Trap:** UI rendering (in browsers) is a macrotask — it runs between macrotask iterations, not between microtasks.

---

### Q52 · Medium
**Can microtasks starve the event loop?**

**Answer:**
Yes. If microtasks continuously queue new microtasks, the macrotask queue (and UI updates) never get a turn — the event loop is starved.

```js
// Infinite microtask loop — starves macrotasks
function infiniteMicro() {
  Promise.resolve().then(infiniteMicro);
}
infiniteMicro();
// setTimeout callbacks never run — UI freezes in browser
```

**Trap:** Interviewers ask "can microtasks block the event loop?" — yes, if they infinitely re-queue. A `while(true)` blocks the stack; infinite microtasks starve the event loop.

---

### Q53 · Hard
**What is the output of this complex event loop puzzle?**

```js
console.log('start');
setTimeout(() => console.log('timeout 1'), 0);
Promise.resolve()
  .then(() => {
    console.log('promise 1');
    setTimeout(() => console.log('timeout 2'), 0);
  })
  .then(() => console.log('promise 2'));
console.log('end');
```

**Answer:**
`start`, `end`, `promise 1`, `promise 2`, `timeout 1`, `timeout 2`

1. Sync: `start`, `end`
2. Microtasks: `promise 1` (queues timeout 2), then `promise 2`
3. Macrotasks: `timeout 1`, `timeout 2`

**Trap:** `timeout 2` is queued *during* the microtask phase — it goes to the macrotask queue and runs after `timeout 1` (which was already queued).

---

### Q54 · Hard
**How does `async/await` interact with the microtask queue — specifically after each `await`?**

**Answer:**
Each `await` suspends the async function and schedules its continuation as a microtask. Multiple `await`s create a chain of microtask continuations, with each continuation flushed after the previous one resolves.

```js
async function chain() {
  console.log('A');
  await step1(); // queues continuation as microtask after step1 resolves
  console.log('B');
  await step2(); // queues another microtask
  console.log('C');
}
// Each 'await' is a microtask checkpoint — other microtasks can run between them
```

**Trap:** Two consecutive `await`s introduce *two* microtask "ticks" — relevant for testing race conditions.

---

### Q55 · Medium
**What is `MessageChannel` and is it a microtask or macrotask?**

**Answer:**
`MessageChannel` is a Web API for creating a two-way communication channel. Messages sent via `port.postMessage()` are delivered as **macrotasks**, not microtasks — they go through the task queue.

```js
const channel = new MessageChannel();
channel.port1.onmessage = () => console.log('message channel');
channel.port2.postMessage('ping');

Promise.resolve().then(() => console.log('promise'));
// Output: promise, message channel (promise is microtask, channel is macrotask)
```

**Trap:** `MessageChannel` is occasionally used as a `setTimeout(0)` alternative — knowing it's a macrotask prevents ordering bugs.

---

### Q56 · Hard
**How does React's scheduler (used in React Native) interact with the event loop?**

**Answer:**
React's scheduler uses `MessageChannel` to yield control back to the browser/native thread between render work slices. This ensures that high-priority events (user input, animations) are processed between chunks of React work, instead of React blocking the thread for a full tree reconciliation.

```
React scheduler:
  - chunk of work → yield via MessageChannel (macrotask)
  - browser processes UI events, animations
  - next MessageChannel callback → next chunk of work
```

**Trap:** In React Native, this is why long lists rendered with default `ScrollView` can block rendering but `FlatList` with `windowSize` does not — FlatList batches render work cooperatively with the scheduler.

**Resume Link:** Your FlatList usage in ERP SKAVO's large data screens directly leverages this.

---

### Q57 · Hard
**What is the difference in microtask handling between Hermes and V8?**

**Answer:**
Both Hermes and V8 follow the ECMAScript spec for microtask queuing (Promise continuations flush after each macrotask). Hermes is optimised for mobile startup performance and bytecode pre-compilation. V8 has more aggressive JIT compilation. Their microtask ordering behaviour is spec-compliant and effectively identical for most use cases, but Hermes lacks some V8-specific optimisations (like `process.nextTick`).

**Trap:** Interviewers ask this to gauge your understanding of the RN runtime environment — the answer is "spec-compliant, effectively the same for Promises."

**Resume Link:** Your app runs on Hermes — knowing the engine helps you reason about performance.

---

### Q58 · Medium
**Why is `setTimeout(fn, 0)` not truly "immediate"?**

**Answer:**
`setTimeout(fn, 0)` schedules `fn` as a macrotask. It runs only after the current synchronous code finishes AND all microtasks (Promises) are flushed. The "0ms" delay is a minimum, not exact — the actual delay depends on the event loop iteration time.

```js
console.log('sync');
setTimeout(() => console.log('timeout'), 0);
Promise.resolve().then(() => console.log('micro'));
// Output: sync, micro, timeout
// timeout is NOT immediate despite 0ms
```

**Trap:** In browsers, the minimum setTimeout delay is clamped to 4ms in nested contexts. In RN/Node.js, it's closer to 1ms but still not zero.

---

### Q59 · Hard
**What happens when you `await` a non-Promise value in JavaScript?**

**Answer:**
`await` wraps non-Promise values in `Promise.resolve(value)`, making the continuation a microtask. The function still suspends for at least one microtask tick.

```js
async function test() {
  console.log('before await');
  await 42; // wraps in Promise.resolve(42) — still suspends
  console.log('after await');
}
console.log('start');
test();
console.log('end');
// Output: start, before await, end, after await
```

**Trap:** `await 42` (non-Promise) still causes a suspension — it's not like calling a sync function. This surprises many candidates.

---

### Q60 · Hard
**How can understanding microtasks help debug a Redux-Saga race condition?**

**Answer:**
Redux-Saga effects like `race` and `take` work at the level of JS event processing. If two actions are dispatched in rapid succession (same synchronous call), they're both in the macrotask or microtask queue. Understanding that Promise-based effects resolve as microtasks helps predict which saga resumes first.

```js
// If two actions dispatched synchronously:
dispatch({ type: 'A' });
dispatch({ type: 'B' });

// Saga using race:
const { a, b } = yield race({
  a: take('A'),
  b: take('B'),
});
// 'a' wins because 'A' was dispatched first — dispatches processed in order
```

**Trap:** Concurrent dispatches from different async sources (e.g., WebSocket + user tap) are non-deterministic — use saga's `race` or `take` with `flush` semantics carefully.

**Resume Link:** Your WebSocket + Redux-Saga integration in the matrimonial app and ERP real-time features.

---

---
## Topic 07 — Callbacks & Callback Hell

### Q61 · Easy
**What is a callback function in JavaScript?**

**Answer:**
A callback is a function passed as an argument to another function, to be invoked after some operation completes. Callbacks are the foundational async pattern in JavaScript before Promises.

```js
function fetchData(url, callback) {
  // Simulate async
  setTimeout(() => {
    const data = { user: 'Devesh' };
    callback(null, data); // Node-style: error first
  }, 500);
}

fetchData('/api/user', (err, data) => {
  if (err) return console.error(err);
  console.log(data.user); // 'Devesh'
});
```

**Trap:** "Callbacks are async" — false. A callback can be synchronous (`Array.map` takes a callback that runs synchronously).

---

### Q62 · Medium
**What is callback hell and what are its downsides?**

**Answer:**
Callback hell (pyramid of doom) occurs when multiple nested callbacks make code hard to read, debug, and maintain. Downsides: poor error handling, difficult to reason about control flow, no easy way to run things in parallel.

```js
// Callback hell — deeply nested
getUser(id, (err, user) => {
  if (err) handle(err);
  getOrders(user.id, (err, orders) => {
    if (err) handle(err);
    getInvoice(orders[0].id, (err, invoice) => {
      if (err) handle(err);
      // pyramid continues...
    });
  });
});

// Fixed with async/await
async function getInvoiceForUser(id) {
  const user = await getUser(id);
  const orders = await getOrders(user.id);
  const invoice = await getInvoice(orders[0].id);
  return invoice;
}
```

**Trap:** Interviewers sometimes present callback hell code and ask "how would you refactor this?" — have the async/await version ready.

**Resume Link:** Your 200+ API ERP app would be unmaintainable with raw callbacks — Redux-Saga/async-await is why it's scalable.

---

### Q63 · Medium
**What is error-first callback convention and why does it exist?**

**Answer:**
Node.js popularised the convention of `callback(error, result)` — the first argument is an error (or `null` if success), the second is the result. This ensures errors are always explicitly handled and not silently swallowed.

```js
fs.readFile('config.json', 'utf8', (err, data) => {
  if (err) {
    console.error('Failed:', err.message);
    return; // early exit on error
  }
  console.log(JSON.parse(data));
});
```

**Trap:** Forgetting the `return` after handling `err` is a common bug — execution continues into the success branch with `undefined` data.

---

### Q64 · Hard
**How would you convert a callback-based function to a Promise?**

**Answer:**
Wrap it in `new Promise`. Use Node's `util.promisify` for Node-style error-first callbacks. Manually wrap non-standard callbacks.

```js
// Manual promisification
function readFilePromise(path) {
  return new Promise((resolve, reject) => {
    fs.readFile(path, 'utf8', (err, data) => {
      if (err) reject(err);
      else resolve(data);
    });
  });
}

// Node.js utility
const { promisify } = require('util');
const readFileAsync = promisify(fs.readFile);

// Usage
const data = await readFileAsync('config.json', 'utf8');
```

**Trap:** If the callback is called multiple times (like an event emitter), `Promise` is wrong — use `Observable` or a stream instead.

---

### Q65 · Hard
**What is `util.callbackify` and when would you use it in React Native?**

**Answer:**
`util.callbackify` converts an async function into a Node-style callback function. Useful when you need to integrate with callback-based APIs from a Promise-based codebase. In React Native, this is rare but can appear in native module bridges.

```js
const { callbackify } = require('util');

async function getUser(id) {
  return fetch(`/api/users/${id}`).then(r => r.json());
}

const getUserCb = callbackify(getUser);
getUserCb(42, (err, user) => {
  if (err) return console.error(err);
  console.log(user);
});
```

**Trap:** `callbackify` always produces error-first callbacks. If the async function throws, the first argument to the callback is the error.

---

### Q66 · Hard
**How do you handle parallel async callbacks without Promises?**

**Answer:**
Use a counter or `async` library. Track completion of N callbacks; when all complete, invoke the final callback.

```js
function parallel(tasks, done) {
  const results = [];
  let completed = 0;

  tasks.forEach((task, i) => {
    task((err, result) => {
      if (err) return done(err);
      results[i] = result;
      if (++completed === tasks.length) done(null, results);
    });
  });
}

// Usage
parallel([fetchA, fetchB, fetchC], (err, [a, b, c]) => {
  console.log(a, b, c);
});
```

**Trap:** The counter approach has a subtle bug if one callback fires multiple times — production code should use Promises or a library.

---

### Q67 · Medium
**What is the "inversion of control" problem with callbacks?**

**Answer:**
When you pass a callback to a third-party library, you give it control over: when your callback is called, how many times, with what arguments, and what `this` is. This "inversion of control" is a trust problem — Promises solve it by putting control back in your hands.

```js
// You trust trackPurchase to call analytics() once, at the right time
trackPurchase(order, analytics); // ← inversion of control

// With Promise, you control the continuation
trackPurchase(order)
  .then(analytics) // you decide when analytics runs
  .catch(handleError);
```

**Trap:** This is a key argument *for* Promises — understanding it shows you know *why* Promises were introduced, not just *how* to use them.

---

### Q68 · Hard
**How does React Native's `NativeModules` bridge use callbacks, and what are the pitfalls?**

**Answer:**
Native modules expose methods that accept JS callbacks (or return Promises). The callback is invoked from the native thread and scheduled on the JS thread. Pitfalls: callbacks can only be called once; if the component unmounts before the callback fires, you may set state on an unmounted component.

```js
import { NativeModules } from 'react-native';

// Callback style (older native modules)
NativeModules.BiometricModule.authenticate((error, success) => {
  if (error) return handleError(error);
  // risk: component may be unmounted by the time this runs
  setAuthenticated(success);
});

// Better: use a ref to track mounted state
const isMounted = useRef(true);
useEffect(() => () => { isMounted.current = false; }, []);
NativeModules.BiometricModule.authenticate((error, success) => {
  if (isMounted.current) setAuthenticated(success);
});
```

**Trap:** "Can't call setState on an unmounted component" — classic RN warning caused by async callbacks.

**Resume Link:** Your native module integrations (Razorpay, Google Maps, push notifications) all use callback/Promise bridges.

---

### Q69 · Medium
**What is the difference between `setTimeout(fn, 0)` and a resolved Promise callback?**

**Answer:**
Both defer execution, but resolved Promise callbacks are **microtasks** (run before the next macrotask), while `setTimeout(fn, 0)` is a **macrotask** (runs after all microtasks). Promises run sooner.

```js
setTimeout(() => console.log('timeout'), 0);
Promise.resolve().then(() => console.log('promise'));
console.log('sync');
// Output: sync, promise, timeout
```

**Trap:** Use `Promise.resolve().then()` when you need to defer work but want it to run as soon as possible (before setTimeout). Use setTimeout when you want to yield to the rendering/UI layer.

---

### Q70 · Hard
**How would you implement a `series` function that runs callbacks sequentially?**

**Answer:**
```js
function series(tasks, done) {
  const results = [];

  function next(index) {
    if (index === tasks.length) return done(null, results);

    tasks[index]((err, result) => {
      if (err) return done(err);
      results.push(result);
      next(index + 1); // tail-recursive pattern
    });
  }

  next(0);
}

series([fetchA, fetchB, fetchC], (err, [a, b, c]) => {
  console.log(a, b, c);
});
```

**Trap:** Each task waits for the previous to complete — unlike `parallel`, which starts all at once. Interviewers ask you to implement both.

---

---
## Topic 08 — Promises

### Q71 · Easy
**What is a Promise in JavaScript and what are its three states?**

**Answer:**
A Promise is an object representing the eventual completion or failure of an async operation. States: **Pending** (initial), **Fulfilled** (success — `.then` runs), **Rejected** (failure — `.catch` runs). Once settled (fulfilled or rejected), a Promise's state is immutable.

```js
const p = new Promise((resolve, reject) => {
  setTimeout(() => resolve('done'), 1000);
});

p.then(val => console.log(val))  // 'done'
 .catch(err => console.error(err));
```

**Trap:** "Resolved = fulfilled" — wrong. A Promise can be "resolved" to another Promise (it *follows* that Promise) without being fulfilled. The spec distinguishes resolved from fulfilled.

---

### Q72 · Medium
**What is the difference between `Promise.all`, `Promise.allSettled`, `Promise.race`, and `Promise.any`?**

**Answer:**
- **`Promise.all`** — waits for all; rejects immediately on any rejection.
- **`Promise.allSettled`** — waits for all; always fulfills, returns array of `{status, value/reason}`.
- **`Promise.race`** — settles with the first settled Promise (fulfilled OR rejected).
- **`Promise.any`** — fulfills with the first fulfilled; rejects only if ALL reject (AggregateError).

```js
const promises = [fetchA(), fetchB(), fetchC()];

// Fails fast if any rejects
await Promise.all(promises);

// Always resolves — useful for parallel calls where partial failure is ok
const results = await Promise.allSettled(promises);
results.forEach(r => {
  if (r.status === 'fulfilled') console.log(r.value);
  else console.error(r.reason);
});
```

**Trap:** Interviewers ask "which one would you use to fetch user + orders + notifications in parallel but not fail if notifications fail?" → `Promise.allSettled`.

**Resume Link:** Your ERP dashboard loads CRM + HRMS + Payroll simultaneously — `Promise.allSettled` is the right choice.

---

### Q73 · Hard
**What is Promise chaining and how does error propagation work?**

**Answer:**
Each `.then`/`.catch` returns a new Promise, enabling chaining. Errors propagate down the chain, skipping `.then` callbacks, until caught by a `.catch`. After a `.catch`, execution continues normally.

```js
fetchUser()
  .then(user => fetchOrders(user.id)) // skipped if fetchUser rejects
  .then(orders => processOrders(orders)) // skipped if either above rejects
  .catch(err => {
    console.error('caught:', err);
    return []; // recover — next .then receives []
  })
  .then(data => console.log('final:', data)); // runs with [] if catch recovered
```

**Trap:** If `.catch` returns a value (or nothing), the chain *resumes in fulfilled state*. If `.catch` throws or returns a rejected Promise, the next `.catch` handles it.

---

### Q74 · Hard
**What is the difference between `.catch(fn)` and `.then(null, fn)`?**

**Answer:**
Both handle rejections, but there's a subtle difference: `.then(onFulfilled, onRejected)` — the `onRejected` will NOT catch errors thrown by `onFulfilled` in the same `.then`. `.catch(fn)` is equivalent to `.then(null, fn)` and catches errors from the *previous* step, not the current `.then`.

```js
// .then(success, error) — error handler doesn't catch success handler errors
promise.then(
  () => { throw new Error('in success'); }, // ← NOT caught by below
  err => console.error('rejection:', err)
);

// .catch() — catches errors from previous .then
promise
  .then(() => { throw new Error('in success'); })
  .catch(err => console.error('caught:', err)); // ✅ catches the error above
```

**Trap:** Most candidates don't know this distinction. It's a genuine footgun.

---

### Q75 · Medium
**What is a "Promise chain anti-pattern" (callback hell with Promises)?**

**Answer:**
Nesting `.then` inside `.then` instead of chaining recreates callback hell. Each nested `.then` creates a new Promise that isn't part of the outer chain, so errors don't propagate correctly.

```js
// Anti-pattern — nested then
fetchUser().then(user => {
  fetchOrders(user.id).then(orders => { // ← not in outer chain!
    fetchInvoice(orders[0].id).then(invoice => {
      console.log(invoice);
    });
  });
});

// Correct — flat chain (return the inner Promise)
fetchUser()
  .then(user => fetchOrders(user.id))    // return Promise
  .then(orders => fetchInvoice(orders[0].id))
  .then(invoice => console.log(invoice))
  .catch(handleError);
```

**Trap:** Forgetting to `return` the inner Promise is the most common bug — without `return`, the outer chain doesn't wait for the inner operation.

---

### Q76 · Hard
**How does `Promise.resolve()` behave when passed another Promise?**

**Answer:**
`Promise.resolve(anotherPromise)` returns `anotherPromise` itself (if it's a native Promise). It does not wrap it in a new Promise. This is called "thenable assimilation" — if the value has a `.then` method, it's treated as a Promise.

```js
const p1 = Promise.resolve('value');
const p2 = Promise.resolve(p1);
console.log(p1 === p2); // true — same Promise returned

// Thenable
const thenable = { then: (resolve) => resolve(42) };
const p3 = Promise.resolve(thenable);
p3.then(v => console.log(v)); // 42
```

**Trap:** `new Promise(resolve => resolve(anotherPromise))` is different — it *does* follow `anotherPromise` and waits for it to settle.

---

### Q77 · Hard
**What are the risks of unhandled Promise rejections in React Native?**

**Answer:**
In RN, unhandled Promise rejections cause a yellow warning in dev and can silently fail in production with older setups. In Node.js (and newer RN), unhandled rejections crash the process. Always attach `.catch()` or wrap with `try/catch`.

```js
// BAD — unhandled rejection
async function loadData() {
  const data = await fetch('/api/data'); // if this rejects, no handler
}

// GOOD — always handle
async function loadData() {
  try {
    const data = await fetch('/api/data');
    return data.json();
  } catch (err) {
    crashReporting.log(err); // log to Sentry/Crashlytics
    throw err; // re-throw if needed
  }
}
```

**Trap:** In Redux-Saga, unhandled errors in worker sagas terminate the saga silently — always wrap saga bodies in try/catch.

**Resume Link:** Your 99% API reliability in Debt Relief India came from robust error-boundary handling.

---

### Q78 · Medium
**What is `Promise.resolve` vs `new Promise(resolve => resolve(...))`?**

**Answer:**
`Promise.resolve(val)` is a shorthand that creates an already-resolved Promise. `new Promise(...)` creates a new Promise and runs the executor synchronously. For non-Promise values, both produce equivalent results. For thenable values, `Promise.resolve` assimilates them; `new Promise` follows them.

```js
// Equivalent for non-thenable values
const p1 = Promise.resolve(42);
const p2 = new Promise(res => res(42));

// Not equivalent for thenables
const t = { then: res => setTimeout(() => res(99), 100) };
const p3 = Promise.resolve(t);      // assimilates thenable
const p4 = new Promise(res => res(t)); // also follows thenable — same result here
```

**Trap:** The distinction matters in edge cases — interviewers ask this to test depth.

---

### Q79 · Hard
**How would you implement a retry mechanism for a failed Promise?**

**Answer:**
```js
async function withRetry(fn, retries = 3, delay = 500) {
  for (let attempt = 1; attempt <= retries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt === retries) throw err;
      await new Promise(res => setTimeout(res, delay * attempt)); // exponential backoff
    }
  }
}

// Usage
const data = await withRetry(() => fetch('/api/payments').then(r => r.json()), 3, 300);
```

**Trap:** Don't retry on non-recoverable errors (4xx). Check `err.status` and only retry on 5xx or network errors.

**Resume Link:** Your Razorpay payment flow and KYC API calls in Debt Relief India would benefit from retry logic.

---

### Q80 · Hard
**What is the difference between `async function` and a function returning a Promise?**

**Answer:**
`async function` always returns a Promise. If the function returns a value, it's wrapped in `Promise.resolve(value)`. If it throws, it returns a rejected Promise. A regular function returning a Promise is functionally similar but has important differences: `async` functions handle thrown errors automatically; regular functions need explicit try/catch or reject calls.

```js
async function asyncFn() {
  throw new Error('oops');
}
asyncFn().catch(e => console.log(e.message)); // 'oops' — auto-wrapped in rejected Promise

function regularFn() {
  throw new Error('oops'); // ← NOT wrapped — crashes synchronously if not caught
}
// vs
function regularFnPromise() {
  return new Promise((_, reject) => reject(new Error('oops'))); // explicit
}
```

**Trap:** A regular function that throws synchronously behaves differently from an async function that throws — the error isn't captured as a rejection.

---

---
## Topic 09 — async / await

### Q81 · Easy
**What does `async/await` do and what does it compile to?**

**Answer:**
`async/await` is syntactic sugar over Promises. An `async` function implicitly returns a Promise. `await` pauses execution of the async function until the awaited Promise settles, then resumes with the resolved value. Under the hood, it's equivalent to `.then()` chains.

```js
// async/await
async function getData() {
  const user = await fetchUser();
  const orders = await fetchOrders(user.id);
  return orders;
}

// Equivalent Promise chain
function getData() {
  return fetchUser()
    .then(user => fetchOrders(user.id));
}
```

**Trap:** "`await` blocks the thread" — false. It suspends only the *async function*, yielding control to the event loop. The thread is not blocked.

---

### Q82 · Medium
**What is the problem with `await` inside `forEach`?**

**Answer:**
`Array.forEach` does not understand Promises — it runs the callback, starts the async operation, and immediately moves to the next iteration without waiting. The `await` inside has no effect on the loop's sequencing.

```js
// BUG — forEach doesn't await
const ids = [1, 2, 3];
ids.forEach(async (id) => {
  const user = await fetchUser(id); // awaited inside the callback, but forEach doesn't wait
  console.log(user);
});
console.log('done'); // prints BEFORE any user is logged

// FIX — for...of (sequential)
for (const id of ids) {
  const user = await fetchUser(id);
  console.log(user);
}

// FIX — Promise.all (parallel)
const users = await Promise.all(ids.map(id => fetchUser(id)));
```

**Trap:** This is one of the most common async bugs in production React Native code. Know both fixes.

**Resume Link:** Your ERP app's batch operations (payroll processing, bulk attendance updates) need this pattern.

---

### Q83 · Medium
**How do you run multiple async operations in parallel with `async/await`?**

**Answer:**
Start all Promises before awaiting any of them, or use `Promise.all`.

```js
// Sequential (slow — waits for each before starting next)
const user = await fetchUser(id);
const orders = await fetchOrders(id);
const profile = await fetchProfile(id);

// Parallel (fast — all start at once)
const [user, orders, profile] = await Promise.all([
  fetchUser(id),
  fetchOrders(id),
  fetchProfile(id),
]);

// Also correct — start then await
const userP = fetchUser(id);
const ordersP = fetchOrders(id);
const user = await userP;
const orders = await ordersP;
```

**Trap:** The "start then await" pattern is less readable but demonstrates understanding that `fetchUser()` starts immediately even before `await`.

**Resume Link:** Your ERP dashboard loading CRM + HRMS + Payroll data simultaneously.

---

### Q84 · Hard
**What is "await waterfall" and how does it hurt performance?**

**Answer:**
Await waterfall is the sequential chaining of independent async operations that could run in parallel. Each operation waits for the previous one to complete, adding total latency.

```js
// Waterfall — unnecessary sequential awaits
// Total time: t(fetchUser) + t(fetchOrders) + t(fetchProfile)
const user = await fetchUser(id);    // 200ms
const orders = await fetchOrders(id); // 300ms (waits for user first)
const profile = await fetchProfile(id); // 150ms (waits for orders first)
// Total: ~650ms

// Parallel — all run at once
// Total time: max(200, 300, 150) = 300ms
const [user, orders, profile] = await Promise.all([...]);
```

**Trap:** Only use sequential awaits when operations are *dependent* (need result of previous). Independent calls should always be parallelised.

**Resume Link:** 35% faster API performance on your resume — this is exactly the kind of optimisation that achieves it.

---

### Q85 · Hard
**What is `top-level await` and does it work in React Native?**

**Answer:**
Top-level `await` (ES2022) allows using `await` at the top level of ES modules without wrapping in an async function. As of React Native with Hermes, top-level `await` has limited support — it works in some bundler configurations (Metro with proper settings) but is not universally available.

```js
// Top-level await (ES2022)
const config = await fetch('/config.json').then(r => r.json());
export { config };

// RN workaround — IIFE or async init
(async () => {
  const config = await loadConfig();
  AppRegistry.registerComponent('App', () => AppWithConfig(config));
})();
```

**Trap:** Don't rely on top-level await in RN without checking your Metro/Hermes version compatibility.

**Resume Link:** Your app initialisation (JWT token loading, app config) is a top-level async concern.

---

### Q86 · Hard
**How do you properly cancel an async operation in React Native?**

**Answer:**
Promises are not natively cancellable. Use an `AbortController` for fetch requests, or a mounted-state ref for component-level cancellation.

```js
function useUserData(userId) {
  const [data, setData] = useState(null);

  useEffect(() => {
    const controller = new AbortController(); // AbortController supported in RN ≥ 0.60

    async function load() {
      try {
        const res = await fetch(`/api/users/${userId}`, { signal: controller.signal });
        const json = await res.json();
        setData(json);
      } catch (err) {
        if (err.name === 'AbortError') return; // intentionally cancelled
        throw err;
      }
    }

    load();
    return () => controller.abort(); // cleanup on unmount or userId change
  }, [userId]);

  return data;
}
```

**Trap:** `AbortController` only cancels the *fetch* — any code after `await fetch(...)` is already running. Check for `AbortError` after each await if needed.

**Resume Link:** User navigation in your ERP app should cancel in-flight API calls to prevent state updates on unmounted screens.

---

### Q87 · Medium
**What is the difference between `throw` inside async and rejecting a Promise explicitly?**

**Answer:**
Inside an `async` function, `throw` is equivalent to `return Promise.reject(error)`. Both result in a rejected Promise. Outside an async function, `throw` is synchronous and not caught by `.catch()`.

```js
// These are equivalent inside async function
async function fn1() { throw new Error('e'); }
async function fn2() { return Promise.reject(new Error('e')); }

fn1().catch(e => console.log(e.message)); // 'e'
fn2().catch(e => console.log(e.message)); // 'e'

// Outside async — synchronous throw, NOT a Promise rejection
function fn3() { throw new Error('e'); }
fn3().catch(...); // ❌ TypeError: fn3(...).catch is not a function
```

**Trap:** Regular functions that throw are not Promises — you can't chain `.catch()` on them unless they explicitly return a Promise.

---

### Q88 · Hard
**How do you handle timeout for an async operation?**

**Answer:**
Use `Promise.race` between the async operation and a timeout Promise.

```js
function withTimeout(promise, ms) {
  const timeout = new Promise((_, reject) =>
    setTimeout(() => reject(new Error(`Timed out after ${ms}ms`)), ms)
  );
  return Promise.race([promise, timeout]);
}

// Usage
const data = await withTimeout(fetchPaymentStatus(orderId), 5000);
```

**Trap:** `Promise.race` does not cancel the original Promise — it just stops waiting for it. The original fetch may still complete; its result is just ignored. Use `AbortController` for true cancellation.

**Resume Link:** Razorpay payment status checks in your Debt Relief India and Wink apps need timeout handling.

---

### Q89 · Hard
**What is the "async constructor" problem in JavaScript classes?**

**Answer:**
Class constructors cannot be `async` — they return the instance synchronously. To initialise a class asynchronously, use a static factory method.

```js
// WRONG
class UserService {
  constructor() {
    this.user = await fetchUser(); // ❌ SyntaxError
  }
}

// CORRECT — static factory
class UserService {
  static async create() {
    const service = new UserService();
    service.user = await fetchUser();
    return service;
  }
}
const service = await UserService.create();
```

**Trap:** This pattern is common in service layer classes in RN apps. Know the static factory workaround.

**Resume Link:** Your ERP app's service layer (JWT auth service, payment service) may use this pattern.

---

### Q90 · Hard
**What is `async generator` and can you give a practical example in React Native?**

**Answer:**
An async generator combines generators and async — it yields Promises and can be iterated with `for await...of`. Useful for streaming paginated API responses.

```js
async function* paginatedFetch(endpoint) {
  let page = 1;
  let hasMore = true;

  while (hasMore) {
    const res = await fetch(`${endpoint}?page=${page}`);
    const data = await res.json();
    yield data.items;
    hasMore = data.hasNextPage;
    page++;
  }
}

// Usage in RN
async function loadAllEmployees() {
  const all = [];
  for await (const batch of paginatedFetch('/api/employees')) {
    all.push(...batch);
    dispatch(appendEmployees(batch)); // update UI progressively
  }
}
```

**Trap:** Async generators are not commonly known — mentioning them signals advanced knowledge.

**Resume Link:** Your ERP HRMS module with large employee lists can use async generators for progressive loading.

---

---
## Topic 10 — Error Handling in Async Code

### Q91 · Medium
**What are the different ways to handle errors in async code?**

**Answer:**
1. `.catch()` on Promises
2. `try/catch` with `async/await`
3. Error-first callbacks
4. Global `unhandledrejection` event

```js
// 1. .catch()
fetch('/api').then(r => r.json()).catch(err => handle(err));

// 2. try/catch
async function load() {
  try {
    const data = await fetch('/api').then(r => r.json());
  } catch (err) {
    handle(err);
  }
}

// 3. Global handler (last resort)
process.on('unhandledRejection', (reason) => {
  crashReporter.log(reason);
});
```

**Trap:** `try/catch` only catches *awaited* rejections. A Promise that isn't awaited won't be caught.

---

### Q92 · Hard
**What happens when you forget `await` and add `try/catch`?**

**Answer:**
Without `await`, the `try` block starts the async operation and exits before it rejects. The rejection becomes unhandled — `catch` never fires.

```js
async function bad() {
  try {
    fetchData(); // ← no await!
  } catch (err) {
    console.log('caught'); // NEVER runs
  }
}

async function good() {
  try {
    await fetchData(); // ← awaited
  } catch (err) {
    console.log('caught'); // ✅ runs on rejection
  }
}
```

**Trap:** The linter rule `no-floating-promises` catches this. Enabling it in TypeScript projects prevents silent unhandled rejections.

**Resume Link:** Your TypeScript codebase in all three apps benefits from enabling strict TypeScript + no-floating-promises.

---

### Q93 · Hard
**How do you handle errors in Redux-Saga?**

**Answer:**
Use `try/catch` inside worker sagas. For non-critical errors, dispatch a failure action. For critical errors, use a `sagaMiddleware.run` error handler.

```js
function* fetchUserSaga(action) {
  try {
    const user = yield call(api.getUser, action.payload.id);
    yield put({ type: 'FETCH_USER_SUCCESS', payload: user });
  } catch (err) {
    yield put({ type: 'FETCH_USER_FAILURE', payload: err.message });
    // Optionally log to crash reporter
    yield call([crashReporter, 'log'], err);
  }
}

// Global saga error handler
const sagaMiddleware = createSagaMiddleware({
  onError: (err) => crashReporter.log(err)
});
```

**Trap:** If a saga throws without a try/catch, the saga *terminates silently* — subsequent dispatches are never handled. Always wrap saga bodies.

**Resume Link:** Your 99% API reliability in Debt Relief India is partly due to this pattern.

---

### Q94 · Hard
**What is an `AggregateError` and when does it occur?**

**Answer:**
`AggregateError` is thrown by `Promise.any()` when *all* provided Promises reject. It contains an `errors` array with all rejection reasons.

```js
const results = await Promise.any([
  fetch('/api/a'),
  fetch('/api/b'),
  fetch('/api/c'),
]).catch(err => {
  if (err instanceof AggregateError) {
    err.errors.forEach(e => console.error(e)); // all three errors
  }
});
```

**Trap:** `AggregateError` is also used by some internal engine operations. Know that `err.errors` is an array, not a single error.

---

### Q95 · Medium
**What is the difference between catching errors with `.catch()` vs `.finally()`?**

**Answer:**
`.catch()` handles errors — its callback receives the rejection reason. `.finally()` runs regardless of success or failure — it's for cleanup (hiding loaders, closing connections) and does not receive the error or value.

```js
fetchData()
  .then(data => setData(data))
  .catch(err => setError(err))  // only on failure
  .finally(() => setLoading(false)); // always runs — cleanup
```

**Trap:** If `.catch()` re-throws or returns a rejected Promise, `.finally()` still runs, and the rejection continues down the chain. `.finally()` does not swallow errors.

**Resume Link:** Every API call in your ERP app should use `.finally` to hide loading indicators.

---

### Q96 · Hard
**How do you implement global error handling for async errors in React Native?**

**Answer:**
Combine: (1) `ErrorBoundary` for render errors, (2) `unhandledrejection` / `global.ErrorUtils` for async errors, (3) Redux-Saga `onError` for saga errors.

```js
// RN global error handler
import { ErrorUtils } from 'react-native';

const originalHandler = ErrorUtils.getGlobalHandler();
ErrorUtils.setGlobalHandler((error, isFatal) => {
  crashReporter.log(error, { isFatal });
  originalHandler(error, isFatal);
});

// Unhandled promise rejections (Hermes)
global.onunhandledrejection = (event) => {
  crashReporter.log(event.reason);
};
```

**Trap:** `ErrorBoundary` does not catch async errors — it only catches errors thrown during render, lifecycle methods, and constructors.

**Resume Link:** Production-grade RN apps like your ERP and fintech apps need this setup.

---

### Q97 · Medium
**What is the "optional chaining" (`?.`) operator and how does it help with error handling?**

**Answer:**
`?.` safely accesses nested properties — returns `undefined` instead of throwing if any part of the chain is `null`/`undefined`. Reduces defensive null checks.

```js
// Without optional chaining
const city = user && user.address && user.address.city;

// With optional chaining
const city = user?.address?.city; // undefined if any is null/undefined

// With method calls
const length = user?.orders?.length ?? 0;
const name = response?.data?.user?.getName?.();
```

**Trap:** `?.` short-circuits the entire expression — `user?.orders?.filter(...)` won't throw if `user` is null, but also won't return an empty array — it returns `undefined`. Combine with `?? []` for safety.

**Resume Link:** Deep nested API responses in your 200+ API ERP app benefit greatly from `?.`.

---

### Q98 · Hard
**What is the difference between error boundaries and try/catch in React Native?**

**Answer:**
`try/catch` handles JS exceptions in synchronous and `async/await` code. `ErrorBoundary` is a React class component that catches errors thrown during *rendering, lifecycle methods, and constructors* of child components — it cannot catch async errors, event handler errors, or errors outside React's rendering.

```js
class ErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, info) {
    crashReporter.log(error, info);
  }

  render() {
    if (this.state.hasError) return ;
    return this.props.children;
  }
}
```

**Trap:** ErrorBoundary does NOT catch: async errors, event handlers, errors in the boundary itself, or SSR errors.

**Resume Link:** Your ERP app's 100+ screens need ErrorBoundary at the route level.

---

### Q99 · Hard
**How would you implement a typed error handling pattern in TypeScript for React Native API calls?**

**Answer:**
Use discriminated union types for responses instead of throwing — caller handles both cases explicitly.

```ts
type ApiResult =
  | { success: true; data: T }
  | { success: false; error: { code: string; message: string } };

async function fetchUser(id: string): Promise<ApiResult> {
  try {
    const res = await fetch(`/api/users/${id}`);
    if (!res.ok) return { success: false, error: { code: String(res.status), message: res.statusText } };
    const data: User = await res.json();
    return { success: true, data };
  } catch (err) {
    return { success: false, error: { code: 'NETWORK', message: (err as Error).message } };
  }
}

// Usage — TypeScript narrows the type
const result = await fetchUser('123');
if (result.success) {
  console.log(result.data.name); // TypeScript knows data exists
} else {
  console.error(result.error.message); // TypeScript knows error exists
}
```

**Trap:** This pattern is inspired by Rust's `Result<T, E>` — interviewers love seeing functional error handling over throw/catch.

**Resume Link:** Your TypeScript usage in all three production apps can benefit from this type-safe pattern.

---

### Q100 · Hard
**What is `cause` property on Error objects (ES2022) and how can it improve debugging?**

**Answer:**
ES2022 added `{ cause }` option to `Error` constructor, allowing you to chain errors and preserve the original error context when re-throwing with a higher-level message.

```js
async function loadPaymentData(userId) {
  try {
    return await fetchPayments(userId);
  } catch (err) {
    throw new Error('Failed to load payment data', { cause: err });
    // err.cause = original network error
  }
}

// Accessing cause
try {
  await loadPaymentData('u123');
} catch (err) {
  console.error(err.message);       // 'Failed to load payment data'
  console.error(err.cause.message); // original error
}
```

**Trap:** `err.cause` is a relatively new property — ensure your crash reporting tool logs `cause` recursively for full error chains.

**Resume Link:** Debugging failed Razorpay transactions or JWT auth issues in your fintech app — error chaining helps trace the root cause.

---

---
## Topic 11 — Prototype & Prototype Chain

### Q101 · Easy
**What is the prototype chain in JavaScript?**

**Answer:**
Every JavaScript object has an internal `[[Prototype]]` link to another object (its prototype). When you access a property, JS first looks at the object itself, then follows the prototype chain up until it finds the property or reaches `null` (end of chain).

```js
const animal = { breathe() { return true; } };
const dog = Object.create(animal);
dog.bark = function () { return 'Woof'; };

console.log(dog.bark());    // own property
console.log(dog.breathe()); // inherited from animal via prototype chain
console.log(dog.missing);   // undefined — not in chain
```

**Trap:** `dog.prototype` is `undefined` (only functions have `.prototype`). The prototype link is accessed via `Object.getPrototypeOf(dog)` or the deprecated `dog.__proto__`.

---

### Q102 · Medium
**What is the difference between `__proto__` and `prototype`?**

**Answer:**
- `prototype` is a property on **constructor functions** — it's the object assigned as `[[Prototype]]` to instances created with `new`.
- `__proto__` (deprecated) is a getter/setter on every object that gives access to the object's `[[Prototype]]`. Use `Object.getPrototypeOf()` instead.

```js
function Dog(name) { this.name = name; }
Dog.prototype.bark = function () { return 'Woof'; };

const d = new Dog('Rex');
console.log(d.__proto__ === Dog.prototype); // true
console.log(Object.getPrototypeOf(d) === Dog.prototype); // true (preferred)
```

**Trap:** `d.prototype` is `undefined` — only functions have `.prototype`. Instances have `[[Prototype]]` accessible via `__proto__` or `Object.getPrototypeOf`.

---

### Q103 · Hard
**How does `class` inheritance work under the hood in JavaScript (ES6)?**

**Answer:**
ES6 `class` is syntactic sugar over prototype-based inheritance. `extends` sets up the prototype chain. `super()` calls the parent constructor. Under the hood, `class Child extends Parent` does `Object.setPrototypeOf(Child.prototype, Parent.prototype)`.

```js
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} makes a sound`; }
}

class Dog extends Animal {
  speak() { return `${this.name} barks`; }
}

const d = new Dog('Rex');
// Dog.prototype.__proto__ === Animal.prototype → true
// d.__proto__ === Dog.prototype → true
// d.__proto__.__proto__ === Animal.prototype → true
```

**Trap:** `class` is not a new object model — it's prototype delegation. Performance-wise, they're equivalent.

---

### Q104 · Hard
**What is the difference between `Object.create(null)` and `{}`?**

**Answer:**
`{}` creates an object whose `[[Prototype]]` is `Object.prototype` (inherits `toString`, `hasOwnProperty`, etc.). `Object.create(null)` creates a truly empty object with *no prototype* — no inherited properties. Useful for pure key/value stores (like cache maps) where inherited keys could cause bugs.

```js
const normal = {};
console.log(normal.toString); // ƒ toString() — inherited

const clean = Object.create(null);
console.log(clean.toString);  // undefined — no prototype
console.log(Object.getPrototypeOf(clean)); // null

// Safe map — no collision with toString, hasOwnProperty etc.
const cache = Object.create(null);
cache['toString'] = 'cached value'; // no conflict with Object.prototype.toString
```

**Trap:** Trying to call `clean.hasOwnProperty(...)` throws because there's no prototype. Use `Object.prototype.hasOwnProperty.call(clean, key)` instead.

---

### Q105 · Medium
**How does `hasOwnProperty` work and when should you use it?**

**Answer:**
`hasOwnProperty` checks if a property is directly on the object (own property), not inherited through the prototype chain. Use it when iterating over object keys to exclude inherited properties.

```js
function Animal() {}
Animal.prototype.breathe = true;
const a = new Animal();
a.name = 'Rex';

console.log('name' in a);            // true — own property
console.log('breathe' in a);         // true — inherited
console.log(a.hasOwnProperty('name')); // true
console.log(a.hasOwnProperty('breathe')); // false

for (const key in a) {
  if (a.hasOwnProperty(key)) console.log(key); // only 'name'
}
```

**Trap:** Modern alternative: `Object.hasOwn(obj, key)` (ES2022) — safer than `obj.hasOwnProperty()` because `hasOwnProperty` can be overridden.

---

### Q106 · Hard
**What is prototypal inheritance vs classical inheritance?**

**Answer:**
- **Classical inheritance** (Java/C++) — classes are blueprints; objects are instances. Inheritance is a static class hierarchy.
- **Prototypal inheritance** (JavaScript) — objects inherit directly from other objects via the prototype chain. There are no true classes (ES6 `class` is syntactic sugar).

Prototypal is more flexible — objects can inherit from any object, and the prototype chain can be modified at runtime.

```js
// Prototypal — direct object inheritance
const vehicleProto = { start() { return 'vroom'; } };
const car = Object.create(vehicleProto);
car.brand = 'Toyota';
car.start(); // inherited

// Classical-style (ES6 class sugar)
class Vehicle { start() { return 'vroom'; } }
class Car extends Vehicle { }
new Car().start(); // works the same under the hood
```

**Trap:** Interviewers ask this to see if you understand JS's actual object model, not just the `class` syntax.

---

### Q107 · Hard
**What is `Object.setPrototypeOf` and why is it considered harmful?**

**Answer:**
`Object.setPrototypeOf(obj, proto)` changes an object's prototype at runtime. Engines optimise objects based on their shape and prototype chain at creation time — changing the prototype invalidates these optimisations, causing significant performance penalties (deoptimisation).

```js
const a = { x: 1 };
const b = { y: 2 };
Object.setPrototypeOf(a, b);
console.log(a.y); // 2 — b is now a's prototype

// ❌ Avoid in performance-critical code — deoptimises engine
// ✅ Use Object.create() at creation time instead
const a2 = Object.create(b);
a2.x = 1;
```

**Trap:** Only use `setPrototypeOf` in special cases (mixin patterns). Never in hot paths.

---

### Q108 · Medium
**What does `instanceof` check?**

**Answer:**
`instanceof` checks if an object's prototype chain contains the `prototype` property of the constructor function. It traverses the entire chain.

```js
class Animal {}
class Dog extends Animal {}

const d = new Dog();
console.log(d instanceof Dog);    // true
console.log(d instanceof Animal); // true — Animal.prototype is in the chain
console.log(d instanceof Object); // true — Object.prototype is always at the top

// Gotcha with different realms (iframes, Node.js vm)
// Array from different realm fails instanceof check
```

**Trap:** `instanceof` fails across different JavaScript realms (e.g., iframes). Use `Array.isArray()` or `Object.prototype.toString.call()` for reliable type checking.

---

### Q109 · Hard
**How do you implement a mixin pattern using prototypes in JavaScript?**

**Answer:**
Mixins copy methods from one object to another's prototype without formal inheritance. Useful for composing behaviour across class hierarchies.

```js
const Serializable = {
  serialize() { return JSON.stringify(this); },
  deserialize(json) { return Object.assign(this, JSON.parse(json)); }
};

const Validatable = {
  validate() { return Object.keys(this).every(k => this[k] !== null); }
};

class User {
  constructor(name, email) {
    this.name = name;
    this.email = email;
  }
}

// Apply mixins
Object.assign(User.prototype, Serializable, Validatable);

const u = new User('Devesh', 'devesh7664@gmail.com');
console.log(u.serialize());  // '{"name":"Devesh","email":"..."}'
console.log(u.validate());   // true
```

**Trap:** Mixins can cause naming collisions — last writer wins. Use `Object.assign` carefully.

---

### Q110 · Hard
**What is the difference between `Object.create`, `new`, and object literals in terms of prototype setup?**

**Answer:**
- **Object literal `{}`** — `[[Prototype]]` is `Object.prototype`
- **`new Constructor()`** — `[[Prototype]]` is `Constructor.prototype`
- **`Object.create(proto)`** — `[[Prototype]]` is explicitly `proto` (can be `null`)

```js
const a = {}; // [[Prototype]] = Object.prototype
const b = new Foo(); // [[Prototype]] = Foo.prototype
const c = Object.create(a); // [[Prototype]] = a
const d = Object.create(null); // [[Prototype]] = null (no inheritance)

console.log(Object.getPrototypeOf(a) === Object.prototype); // true
console.log(Object.getPrototypeOf(c) === a); // true
```

**Trap:** Interviewers ask this to see if you understand all three creation patterns and their prototype implications.

---

---
## Topic 12 — `this` Keyword

### Q111 · Easy
**What does `this` refer to in JavaScript?**

**Answer:**
`this` refers to the *execution context* — the object that is currently calling the function. Its value depends on *how* the function is called, not where it's defined (except for arrow functions, which use lexical `this`).

```js
const obj = {
  name: 'Devesh',
  greet() { console.log(this.name); }
};
obj.greet(); // 'Devesh' — this = obj

const fn = obj.greet;
fn(); // undefined (or TypeError in strict mode) — this = global/undefined
```

**Trap:** "this refers to the function" — wrong. `this` refers to the *caller*, not the function itself.

---

### Q112 · Medium
**What are the four rules for determining `this`?**

**Answer:**
1. **Default binding** — standalone call: `this` = global (non-strict) or `undefined` (strict)
2. **Implicit binding** — method call: `this` = object before the dot
3. **Explicit binding** — `call`, `apply`, `bind`: `this` = first argument
4. **`new` binding** — constructor call: `this` = newly created object

```js
function greet() { console.log(this.name); }
const obj = { name: 'Devesh', greet };

greet();           // 1. Default — undefined/global
obj.greet();       // 2. Implicit — obj
greet.call({ name: 'Ravi' }); // 3. Explicit — {name: 'Ravi'}
const g = new greet(); // 4. new — fresh object (name = undefined)
```

**Trap:** Arrow functions ignore all four rules — they always use lexical `this` from their definition scope.

---

### Q113 · Hard
**What is `this` inside a React Native component's event handler?**

**Answer:**
In class components, event handlers need `this` bound explicitly (via `.bind` in constructor or arrow class field). In function components, there is no `this` — use hooks. Arrow functions as class properties automatically bind `this`.

```js
// Class component — bind in constructor
class MyButton extends React.Component {
  constructor(props) {
    super(props);
    this.handlePress = this.handlePress.bind(this);
  }
  handlePress() { console.log(this.props.label); } // this = component
  render() {
    return ;
  }
}

// Arrow class field — auto-binds
class MyButton2 extends React.Component {
  handlePress = () => { console.log(this.props.label); }; // lexical this
}

// Function component — no this needed
function MyButton3({ label }) {
  const handlePress = () => console.log(label); // closure over label
  return ;
}
```

**Trap:** Inline arrow in render
Claude's r

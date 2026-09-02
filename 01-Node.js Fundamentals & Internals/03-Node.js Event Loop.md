# 1.3 Node.js Event Loop

This is **one of the highest-value Node.js interview topics**, especially for a senior developer.

And I want to build it from the ground up rather than immediately throwing the famous "Event Loop phases" diagram at you.

---

# 1.3.1 What problem does the Event Loop solve?

Let's start with the fundamental problem.

Node.js executes JavaScript primarily on **one main JavaScript thread**.

Imagine your server receives:

```text
Request A
Request B
Request C
Request D
Request E
```

If Node.js simply processed each request synchronously from beginning to end:

```text
Request A → finish
              ↓
Request B → finish
              ↓
Request C → finish
```

then a slow operation in Request A could prevent B, C, D, and E from being processed.

But Node.js is designed around **non-blocking I/O**.

So instead, when Node encounters an operation that can happen asynchronously, it can effectively say:

> "Start this operation, and I'll continue doing other JavaScript work. Let me know when the operation is ready."

That's where the **Event Loop** becomes important.

---

# 1.3.2 Simple example

Consider:

```javascript
console.log("A");

setTimeout(() => {
    console.log("B");
}, 1000);

console.log("C");
```

What is the output?

```text
A
C
B
```

Why?

Because `setTimeout()` doesn't mean:

> "Stop JavaScript execution for one second."

Instead, it means approximately:

> **"Make this callback eligible to run after at least this amount of time."**

The Event Loop helps coordinate when that callback gets a chance to execute.

---

# 1.3.3 The Call Stack

Before understanding the Event Loop, you need to understand the **Call Stack**.

Consider:

```javascript
function greet() {
    console.log("Hello");
}

greet();
```

When JavaScript executes:

```text
greet()
```

the function gets placed onto the call stack.

Conceptually:

```text
Call Stack
────────────
greet()
────────────
global code
```

When `greet()` finishes:

```text
Call Stack
────────────
global code
```

And eventually the stack becomes empty.

---

## Another example

```javascript
function a() {
    b();
}

function b() {
    c();
}

function c() {
    console.log("Hello");
}

a();
```

The stack grows:

```text
c()
b()
a()
global
```

Then functions return:

```text
b()
a()
global
```

Then:

```text
a()
global
```

Then:

```text
global
```

Then empty.

### Key idea

> **The Call Stack keeps track of the JavaScript functions currently being executed.**

And V8 manages this JavaScript execution.

---

# 1.3.4 Why can the Call Stack cause blocking?

Consider:

```javascript
function heavyTask() {
    for (let i = 0; i < 10_000_000_000; i++) {
        // expensive computation
    }
}

heavyTask();
```

The function remains on the call stack while it executes.

During that time, the main JavaScript thread is busy.

So other JavaScript work cannot execute.

This is what we mean by **blocking the JavaScript thread**.

For a Node.js server, that can mean:

```text
Request A
   ↓
CPU-heavy JS
   ↓
Main thread busy
   ↓
Request B waits
Request C waits
Request D waits
```

This is why:

> **CPU-heavy synchronous JavaScript can block Node.js.**

We'll return to this in the Production-Level Node.js section.

---

# 1.3.5 So where does the Event Loop come in?

Now consider:

```javascript
console.log("A");

setTimeout(() => {
    console.log("B");
}, 1000);

console.log("C");
```

The execution roughly begins:

```text
              V8
               ↓
          Execute JS
               ↓
        console.log("A")
               ↓
        setTimeout(...)
               ↓
        console.log("C")
```

The `setTimeout()` callback doesn't immediately execute.

Instead, Node.js/runtime mechanisms keep track of the timer.

After the timer becomes eligible, the callback can eventually be processed by the Event Loop.

Eventually:

```text
Timer ready
    ↓
Event Loop
    ↓
Callback ready to execute
    ↓
V8 / Call Stack
    ↓
console.log("B")
```

---

# 1.3.6 Important: Event Loop does NOT execute JavaScript

This is one of the most important distinctions.

Don't say:

> ❌ "The Event Loop executes JavaScript."

Instead:

> ✅ **The Event Loop coordinates when callbacks/tasks are ready to be executed, while V8 executes the JavaScript.**

Think:

```text
Event Loop
    ↓
"What JavaScript work is ready?"
    ↓
Put/select work for execution
    ↓
V8
    ↓
Execute JavaScript
```

---

# 1.3.7 Why is it called an "Event Loop"?

Because it continuously checks for work.

Very simplified:

```text
       ┌───────────────────┐
       │                   ↓
       │              Check work
       │                   │
       │                   ↓
       │             Is work ready?
       │              /          \
       │            Yes           No
       │             ↓             │
       │        Execute/          │
       │        schedule          │
       │             │             │
       └─────────────┴─────────────┘
```

It's continuously cycling through runtime work.

But **this is intentionally simplified**.

Node's actual Event Loop has multiple phases, which we'll learn shortly.

---

# 1.3.8 Let's look at asynchronous I/O

Consider:

```javascript
const fs = require("fs");

console.log("A");

fs.readFile("data.txt", () => {
    console.log("B");
});

console.log("C");
```

Output:

```text
A
C
B
```

Why?

Because when Node encounters:

```javascript
fs.readFile(...)
```

it doesn't synchronously wait for the filesystem operation to complete.

Conceptually:

```text
V8
 ↓
fs.readFile()
 ↓
Node.js API
 ↓
libuv / OS
 ↓
Start file operation
```

JavaScript continues:

```text
console.log("C")
```

Meanwhile, the file operation happens asynchronously.

Eventually:

```text
File operation complete
        ↓
Callback becomes ready
        ↓
Event Loop processes it
        ↓
V8 executes callback
```

---

# 1.3.9 This is the heart of Node.js

This architecture allows Node.js to handle many concurrent I/O operations without creating one JavaScript thread per request.

For example:

```text
               Node.js
                  │
            Main JS Thread
                  │
                V8
                  │
             Event Loop
        ┌─────────┼─────────┐
        ↓         ↓         ↓
      File       DB       Network
      I/O        I/O        I/O
```

While one operation is waiting on I/O, Node can continue processing other JavaScript work.

That's why Node.js is particularly good for workloads involving lots of:

* HTTP requests
* database operations
* network calls
* filesystem operations
* external APIs

---

# 1.3.10 Concurrency vs Parallelism

This is another important concept.

Suppose:

```text
Request A → waiting for DB
Request B → executing JS
Request C → waiting for API
Request D → executing JS
```

Node can **concurrently manage** these operations.

But that doesn't necessarily mean all JavaScript is executing in parallel on multiple CPU cores.

### Concurrency

> Multiple tasks are in progress during overlapping periods.

### Parallelism

> Multiple tasks are literally executing at the same time, typically on multiple CPU cores/threads.

For now:

```text
Node.js main JS execution
        ↓
     primarily
    single thread
```

but:

```text
Node.js runtime
      ↓
can involve
multiple underlying threads
```

We'll explore this properly when we cover libuv, Thread Pool, and Worker Threads.

---

# 1.3.11 Very important misconception

People often say:

> "Node.js is single-threaded, therefore it can only handle one request at a time."

❌ Wrong.

The better explanation:

> **JavaScript execution is primarily single-threaded, but Node.js can handle many concurrent I/O operations because it uses non-blocking I/O and an Event Loop to coordinate asynchronous work.**

For example:

```text
Request A
   ↓
DB query ────────────────┐
                         │
Request B                │
   ↓                     │
API call ─────────────┐  │
                      │  │
Request C             │  │
   ↓                  │  │
Execute JS            │  │
                      ↓  ↓
                 Results ready
                       ↓
                  Event Loop
                       ↓
                       V8
```

---

# 1.3.12 The architecture so far

We've now connected our first three topics:

```text
                    Node.js Runtime
                           │
            ┌──────────────┴──────────────┐
            │                             │
           V8                           libuv
            │                             │
     Executes JS                    Event Loop
            │                             │
      Call Stack                  Async operations
            │
      JavaScript
      execution
```

When asynchronous work completes:

```text
Async operation
      ↓
Callback becomes ready
      ↓
Event Loop
      ↓
Call Stack
      ↓
V8 executes callback
```

Again, this is a **learning-level model**, not yet the complete implementation.

---

# 1.3.13 Production example

Imagine your API:

```javascript
app.get("/users", async (req, res) => {
    const users = await db.query("SELECT * FROM users");

    res.json(users);
});
```

While the database is processing:

```text
Request
  ↓
Node starts DB operation
  ↓
JavaScript doesn't sit there doing nothing
  ↓
Event Loop can continue processing other work
```

That's why thousands of requests can be **in flight** even though JavaScript execution happens primarily on one main thread.

But if you instead do:

```javascript
app.get("/users", (req, res) => {

    // huge CPU-intensive synchronous operation

    res.json(...);
});
```

then:

```text
Request A
   ↓
CPU-heavy JS
   ↓
Main JS thread blocked
   ↓
Event Loop can't process other JS callbacks
   ↓
Other requests suffer
```

This distinction is extremely important for production Node.js.

---

# 📝 Notes for your notebook

I recommend keeping your notes like this:

```text
## 1.3 Event Loop

### Purpose

Node.js uses an Event Loop to coordinate asynchronous,
non-blocking operations while JavaScript execution happens
primarily on a single main thread.

### Call Stack

- Keeps track of currently executing JavaScript functions.
- Managed as part of JavaScript execution by V8.
- Only one JS execution path runs at a time on the main thread.
- Long-running synchronous JS blocks the main thread.

### Event Loop

- Coordinates when asynchronous callbacks/tasks are ready
  to execute.
- It does NOT execute JavaScript itself.
- V8 executes the JavaScript callbacks.

Simplified flow:

JavaScript
    ↓
V8 / Call Stack
    ↓
Node.js API
    ↓
Async operation
    ↓
Callback becomes ready
    ↓
Event Loop
    ↓
Call Stack
    ↓
V8 executes callback

Important:

- Node.js is not "one request at a time".
- JavaScript execution is primarily single-threaded.
- Node.js supports high concurrency through non-blocking I/O.
- Concurrency ≠ parallelism.
- CPU-heavy synchronous JS can block the Event Loop.
```

---

# 🎯 Interview checkpoint

Don't worry about the detailed Event Loop phases yet. **First make sure this basic model is solid.**

Answer these in your own words:

### Q1

**What problem does the Node.js Event Loop solve?**

### Q2

Consider:

```javascript
console.log("A");

setTimeout(() => {
    console.log("B");
}, 0);

console.log("C");
```

**Why is the output `A → C → B` instead of `A → B → C`?**

### Q3

**If the Event Loop doesn't execute JavaScript, what exactly does it do?**

### Q4 — Senior-level

Someone tells you:

> "Node.js is single-threaded, so if Request A is waiting for a database response, Requests B and C have to wait."

**How would you respond?**

Answer these like you're in an interview. I'll review them, and **then we'll go into the actual Event Loop phases**, which is where `timers`, `poll`, `check`, `setImmediate()`, etc. start making sense.

### My Thoughts-
So, Node.js event loop solves the problem where that JavaScript engine call stack is blocked because of the JavaScript code is executing for a long time, or it is a CPU-intensive code, or it is a long-running code, then the rest of the JavaScript code will not be executed. So that's where this event loop comes, where these asynchronous code or non-blocking input/output code is handled by the event loop. When I say handle, so it, like, so it just makes those processes running, and so it will be whenever these processes are completed and ready, then this call stack will be sent this JavaScript code, which will be executed. So mostly, so it is mostly about coordination, and event loop does not execute the code. So it just coordinates the async code, so it just coordinates the async code, non-blocking code. And yeah, so for question number two, the C will be printed because this setTimeout code is async, and this will be moved to the event loop, and JavaScript engine will be executing the console.log C first. And so yeah, Node.js is a single-threaded mainly because JavaScript code is run on the single thread, main thread. But request B and C do not have to wait because if request A is waiting for a database response, and if it is an asynchronous code, or if it is an async code, then this will be handled by the execution flow will be handled by the event loop. And that is how the B and C will be executed while the request A is coordinated by event loop, and then will be executed once the request A is ready to execute.

Good progress. Your **core mental model is correct**, especially these two points:

* **V8 executes JavaScript; the Event Loop coordinates asynchronous work.**
* **Request A waiting on I/O does not inherently block Requests B/C.**

I'd give this **8/10**. The main thing to fix is one subtle but important misconception about **what problem the Event Loop actually solves**.

---

## 1. Your answer about the Event Loop

You said:

> "Event loop solves the problem where the JavaScript engine call stack is blocked because JavaScript is executing for a long time..."

⚠️ This is the one part I want you to correct.

The Event Loop **doesn't solve a blocked Call Stack**.

If you have:

```javascript
while (true) {
}
```

or:

```javascript
function heavyWork() {
    for (let i = 0; i < 10_000_000_000; i++) {}
}

heavyWork();
```

the Event Loop **cannot help you**.

The main JavaScript thread is busy:

```text
Main JS Thread
      ↓
     V8
      ↓
 Call Stack
      ↓
heavyWork()
      ↓
     BLOCKED
```

The Event Loop is effectively waiting for the JavaScript thread to become available.

So:

> **The Event Loop does not prevent JavaScript from blocking.**

Instead, the Event Loop enables Node.js to **avoid blocking while waiting for asynchronous I/O**.

That's a very important distinction.

---

# 2. The actual problem the Event Loop helps solve

Suppose:

```javascript
const result = database.query();
```

If Node waited synchronously:

```text
Request A
   ↓
DB query
   ↓
WAIT 200 ms
   ↓
continue
```

the JavaScript thread would be sitting idle during those 200 ms.

Instead, with asynchronous I/O:

```text
Request A
   ↓
Start DB operation
   ↓
Don't wait
   ↓
Continue doing other JS work
```

Meanwhile:

```text
             DB
             │
             │ processing
             │
             ↓
       Result becomes ready
             │
             ↓
        Event Loop
             │
             ↓
       Callback ready
             │
             ↓
            V8
```

So the key benefit is:

> **Node.js can continue processing other work while asynchronous I/O is in progress.**

---

# 3. Your statement about coordination was excellent

You said:

> "It is mostly about coordination, and Event Loop does not execute the code."

✅ **Exactly.**

Keep this mental model:

```text
Event Loop
    ↓
Coordinates / determines what work is ready
    ↓
JavaScript execution
    ↓
V8
```

But there's one nuance:

The Event Loop doesn't simply take *any* async code and put it somewhere.

Different types of asynchronous operations are handled through different runtime mechanisms and eventually become eligible in particular Event Loop phases/queues.

**That's what we're going to learn next.**

---

# 4. Your `setTimeout(..., 0)` answer

You said:

> "C will be printed because setTimeout code is async."

✅ Correct result.

```text
A
C
B
```

But let's make your explanation more precise.

When JavaScript reaches:

```javascript
setTimeout(() => {
    console.log("B");
}, 0);
```

`0` does **not** mean:

> "Execute this immediately after the current line."

It means approximately:

> **"The timer should become eligible after the specified minimum delay."**

The current synchronous JavaScript execution continues:

```text
console.log("A")
       ↓
setTimeout(...)
       ↓
console.log("C")
```

Only after the current execution finishes can the timer callback eventually be executed.

So:

```text
A
↓
setTimeout registered
↓
C
↓
current JS execution finishes
↓
timer becomes eligible
↓
Event Loop
↓
V8 executes callback
↓
B
```

### ⭐ Important interview wording

Don't say:

> ❌ "The callback is moved to the Event Loop."

The Event Loop is not really a queue where callbacks are simply dumped.

Say:

> **"The timer is registered, and once its delay has elapsed, its callback becomes eligible to be processed by the appropriate Event Loop phase."**

We'll make this much clearer when we discuss **Event Loop phases**.

---

# 5. Your single-threaded explanation

You said:

> "Node.js is single-threaded mainly because JavaScript code is run on the single thread, main thread."

✅ Good.

I'd make it:

> **"Node.js primarily uses a single main thread for JavaScript execution. However, the Node.js runtime can use additional threads for certain operations, such as work handled by the libuv thread pool or Worker Threads."**

This distinction will become important later.

---

# 6. Your Request A/B/C explanation

You said:

> "Request B and C do not have to wait because if request A is waiting for a database response..."

✅ Correct.

But there's an important condition:

> **The database operation must be asynchronous/non-blocking from Node's perspective, and Request A must not perform CPU-heavy synchronous JavaScript while waiting.**

For example:

```text
Request A
   ↓
DB query ────────────────────→ result
   │
   │ waiting
   │
   └──────────────────────────────┐
                                  │
Request B → JavaScript executes   │
                                  │
Request C → JavaScript executes   │
                                  │
                                  ↓
                            DB result ready
                                  ↓
                            Event Loop
                                  ↓
                                 V8
```

That's the core reason Node.js is good at handling **I/O-heavy concurrent workloads**.

---

# One correction I'd really like you to remember

There are two completely different situations:

### Situation 1 — Waiting for I/O

```text
JavaScript
   ↓
Start DB request
   ↓
Return control
   ↓
Other JS can execute
```

✅ Event Loop architecture helps.

---

### Situation 2 — CPU-heavy JavaScript

```text
JavaScript
   ↓
CPU-heavy loop
   ↓
Main JS thread occupied
   ↓
Event Loop can't execute other JS
```

❌ Event Loop does **not** solve this.

This distinction is **extremely important for senior interviews**.

---

# Your answer, polished for an interview

If the interviewer asks:

> **"What problem does the Node.js Event Loop solve?"**

A strong answer would be:

> "Node.js executes JavaScript primarily on a single thread, so if we synchronously perform a long-running or CPU-intensive operation, that thread can be blocked. The Event Loop doesn't solve that blocking. Instead, it enables Node.js to handle asynchronous, non-blocking operations without making the JavaScript thread wait for their completion. While an I/O operation such as a database or network request is in progress, Node.js can process other JavaScript work. Once the asynchronous operation is ready, the Event Loop coordinates when its callback can execute, and V8 executes that JavaScript."

**That's a senior-level answer.**

---

## One thing I want to change in our learning sequence

We now have enough foundation to go into the **actual Event Loop mechanics**.

### Next: **1.3.1 Event Loop Components & Phases**

We'll cover:

```text
Call Stack
    ↓
Event Loop
    ↓
┌─────────────────────┐
│ Timers              │
│ Pending Callbacks   │
│ Poll                │
│ Check               │
│ Close Callbacks     │
└─────────────────────┘
```

And then we'll bring in:

* `setTimeout()`
* `setImmediate()`
* I/O callbacks
* `process.nextTick()`
* Promise microtasks

**But I recommend we don't jump into `nextTick()` and Promises yet.** First understand the core phases. Otherwise the Event Loop becomes a memorization exercise rather than something you can reason about.

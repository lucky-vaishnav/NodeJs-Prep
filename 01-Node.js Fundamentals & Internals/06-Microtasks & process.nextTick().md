# 1.3.3 Microtasks & `process.nextTick()`

This is one of the most important parts of the Node.js Event Loop for senior interviews because it explains why **Promises and `process.nextTick()` can execute before timers and I/O callbacks**, even though they aren't normal Event Loop phases.

---

## 1. First: What is a Microtask?

A **microtask** is a small piece of work that JavaScript schedules to run **after the current synchronous JavaScript execution finishes**, before the Event Loop continues with normal asynchronous phases.

The most common source in Node.js is:

```js
Promise.resolve().then(() => {
    console.log("Promise");
});
```

For example:

```js
console.log("A");

Promise.resolve().then(() => {
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

```text
Synchronous JavaScript
       ↓
A
       ↓
Promise callback scheduled
       ↓
C
       ↓
Current JS execution finishes
       ↓
Microtask executes
       ↓
B
```

So the basic rule is:

> **Microtasks execute after the current JavaScript execution completes, before the Event Loop moves on to other asynchronous work.**

---

# 2. Where does the Microtask Queue fit?

This is where our previous mental model needs to become slightly more detailed.

Previously we had:

```text
Timers
  ↓
Pending Callbacks
  ↓
Poll
  ↓
Check
  ↓
Close
```

Now add microtasks conceptually:

```text
Current JavaScript execution
          ↓
   Microtasks
          ↓
   Event Loop phase
```

And after callbacks/tasks execute, Node gets opportunities to process microtasks before continuing.

A simplified model:

```text
┌──────────────────────────────┐
│ Execute JavaScript callback  │
└──────────────┬───────────────┘
               ↓
       Microtasks execute
               ↓
       Continue Event Loop
```

**Important:** Microtasks are **not another Event Loop phase**.

That's a very important interview point.

---

# 3. Promise callbacks are microtasks

Consider:

```js
console.log("A");

setTimeout(() => {
    console.log("B");
}, 0);

Promise.resolve().then(() => {
    console.log("C");
});

console.log("D");
```

What happens?

First, synchronous code:

```text
A
D
```

Then the Promise callback is a microtask.

The timer is an Event Loop timer callback.

So:

```text
A
D
C
B
```

Why?

```text
Synchronous JS
   ↓
A
D
   ↓
Microtask
   ↓
C
   ↓
Timers phase
   ↓
B
```

Therefore:

> **Promise microtasks normally execute before timer callbacks once the current synchronous execution has finished.**

---

# 4. Now comes `process.nextTick()`

Node.js has another mechanism:

```js
process.nextTick(() => {
    console.log("nextTick");
});
```

This is **not the same thing as a Promise microtask**, although it has very high priority.

Consider:

```js
console.log("A");

process.nextTick(() => {
    console.log("B");
});

Promise.resolve().then(() => {
    console.log("C");
});

console.log("D");
```

The output is:

```text
A
D
B
C
```

Why?

Because Node processes the `process.nextTick()` queue **before the Promise microtask queue**.

Conceptually:

```text
Current JS
    ↓
process.nextTick()
    ↓
Promise microtasks
    ↓
Event Loop continues
```

So for now remember:

```text
process.nextTick()
       ↓
Promise microtasks
       ↓
Event Loop phases
```

---

# 5. Very important: `process.nextTick()` is not technically a Microtask

This distinction matters for senior interviews.

Don't say:

> "`process.nextTick()` is a microtask."

More accurate:

> **`process.nextTick()` uses Node.js's special next-tick queue, which is processed with higher priority than the Promise microtask queue.**

So:

```text
Node.js
├── nextTick queue
└── Microtask queue
      └── Promise callbacks
```

Both have priority over normal Event Loop phase callbacks, but `nextTick` has higher priority.

---

# 6. Example combining everything

Now look at this:

```js
console.log("A");

setTimeout(() => {
    console.log("B");
}, 0);

setImmediate(() => {
    console.log("C");
});

Promise.resolve().then(() => {
    console.log("D");
});

process.nextTick(() => {
    console.log("E");
});

console.log("F");
```

First:

```text
A
F
```

because they're synchronous.

Then:

```text
E
```

because `process.nextTick()` has priority.

Then:

```text
D
```

because Promise callbacks are microtasks.

Then we reach normal Event Loop processing:

```text
B / C
```

Their relative ordering can depend on context because we're comparing `setTimeout()` and `setImmediate()` from the main module.

So the important structure is:

```text
A
F
E
D
B/C
```

with **B/C potentially varying**.

---

# 7. Why does Node have `process.nextTick()`?

This is a very good senior-level question.

Node uses `process.nextTick()` to allow certain callbacks to run **immediately after the current operation completes**, before the Event Loop proceeds to another phase.

For example, an API can use it to ensure something happens asynchronously:

```js
function example(callback) {
    process.nextTick(callback);
}
```

Instead of calling:

```js
callback();
```

synchronously.

So:

```js
example(() => {
    console.log("callback");
});

console.log("after");
```

produces:

```text
after
callback
```

The callback is deferred until the current synchronous execution completes.

---

# 8. But there's a danger

Because `process.nextTick()` has such high priority, abusing it can **starve the Event Loop**.

For example:

```js
function loop() {
    process.nextTick(loop);
}

loop();
```

Conceptually:

```text
nextTick
   ↓
nextTick
   ↓
nextTick
   ↓
nextTick
   ↓
...
```

The Event Loop may never get an opportunity to proceed to timers, I/O, etc.

This is called:

> **Event Loop starvation**

This is a very important production implication.

Similarly, recursively creating huge numbers of Promise microtasks can also prevent normal Event Loop work from getting sufficient opportunity to run.

---

# 9. Your mental model so far

We can now upgrade our Event Loop model:

```text
                 JavaScript execution
                         ↓
                ┌────────────────┐
                │ current JS ends│
                └───────┬────────┘
                        ↓
              process.nextTick()
                        ↓
                 Promise microtasks
                        ↓
              Event Loop continues
                        ↓
        ┌──────────────────────────────┐
        │ Timers                       │
        │ Pending Callbacks            │
        │ Poll                         │
        │ Check                        │
        │ Close Callbacks              │
        └──────────────────────────────┘
```

Don't treat this as a literal implementation diagram yet. It's our **interview mental model**.

We'll refine exactly **when Node drains these queues** later.

---

## Checkpoint

Don't worry about getting these perfect—we'll use them to find gaps.

### Q1

What is the difference between a **normal Event Loop phase** and a **microtask**?

### Q2

What is the output?

```js
console.log("A");

Promise.resolve().then(() => {
    console.log("B");
});

process.nextTick(() => {
    console.log("C");
});

console.log("D");
```

### Q3

What is the output ordering?

```js
setTimeout(() => console.log("A"), 0);

Promise.resolve().then(() => console.log("B"));

process.nextTick(() => console.log("C"));

console.log("D");
```

### Q4 — Senior

Why can excessive recursive `process.nextTick()` calls be dangerous for a production Node.js server?

Answer these in your own words like you've been doing, and I'll correct the nuances before we go deeper into **Promise scheduling and microtask draining**.

Let's review all four.

### Q1 — Microtasks vs Event Loop phases

✅ **Correct.**

Your answer captures the important idea:

> Microtasks are not an Event Loop phase and are processed with priority after the current JavaScript execution.

For interview language:

> **Microtasks are processed after the current JavaScript execution completes and before normal Event Loop phase callbacks continue.**

---

### Q2 — Promise vs `process.nextTick()`

Your answer:

```text
A
D
C
B
```

✅ **Correct.**

The execution flow is:

```text
Synchronous JS
  ↓
A
D
  ↓
process.nextTick()
  ↓
C
  ↓
Promise microtask
  ↓
B
```

So remember:

```text
process.nextTick() > Promise microtask
```

---

### Q3

Code:

```js
setTimeout(() => console.log("A"), 0);

Promise.resolve().then(() => console.log("B"));

process.nextTick(() => console.log("C"));

console.log("D");
```

You answered:

```text
D
C
B
A
```

✅ **Correct.**

Exactly the priority model we want:

```text
1. Synchronous JS       → D
2. process.nextTick()   → C
3. Promise microtask    → B
4. Timer                → A
```

Excellent.

---

### Q4 — Event Loop starvation

You answered:

> event loop starvation

⚠️ Correct concept, but for a **senior interview**, explain the mechanism.

A stronger answer:

> **Recursive or excessive `process.nextTick()` calls can continuously drain the nextTick queue, preventing the Event Loop from reaching timers, I/O, and other phases. This can cause Event Loop starvation and delay other requests and callbacks.**

For example:

```js
function loop() {
    process.nextTick(loop);
}

loop();
```

Conceptually:

```text
nextTick
   ↓
nextTick
   ↓
nextTick
   ↓
nextTick
   ↓
   ...
```

The Event Loop doesn't get a reasonable opportunity to proceed.

---

### Next subtopic

We've now introduced the basic concept of microtasks and `process.nextTick()`. The next thing we should learn is:

```text
 Microtask / nextTick draining behavior
```

This is where we'll answer a more subtle question:

> **Does Node process all microtasks only once after the entire Event Loop phase, or can microtasks run between individual callbacks within a phase?**

That distinction is important for senior-level interview questions and will make the execution-order behavior much clearer.

---

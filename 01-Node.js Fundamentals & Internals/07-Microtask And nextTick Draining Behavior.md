# 1.3.3.1 Microtask / `nextTick` Draining Behavior

Previously, we simplified things as:

```text
Current JS
   ↓
nextTick
   ↓
Promise microtasks
   ↓
Event Loop phase
```

That's useful initially, but now we need to refine it.

The important question is:

> **If a phase has multiple callbacks, does Node execute all callbacks first and then process microtasks, or can microtasks execute between callbacks?**

### Answer: Microtasks can execute between callbacks.

Consider:

```js
setTimeout(() => {
    console.log("Timer 1");

    Promise.resolve().then(() => {
        console.log("Promise 1");
    });
}, 0);

setTimeout(() => {
    console.log("Timer 2");
}, 0);
```

You might initially think:

```text
Timer 1
Timer 2
Promise 1
```

But the important behavior to understand is:

```text
Timer 1
Promise 1
Timer 2
```

Conceptually:

```text
Timers phase
    ↓
Timer 1 callback starts
    ↓
Timer 1 callback finishes
    ↓
microtasks are processed
    ↓
Promise 1
    ↓
next timer callback
    ↓
Timer 2
```

So Node doesn't simply do:

```text
Run every timer callback
        ↓
Then run all Promises
```

Instead, microtask processing can happen **after an individual callback completes**, before Node proceeds to another callback.

---

# 2. Why does this matter?

Consider a server receiving requests.

Suppose several callbacks are ready:

```text
Callback A
Callback B
Callback C
```

If Callback A creates a Promise:

```js
function callbackA() {
    Promise.resolve().then(() => {
        // some work
    });
}
```

The Promise callback doesn't necessarily wait until **all** ready callbacks have finished.

Conceptually:

```text
Callback A
   ↓
Promise microtask
   ↓
Callback B
   ↓
Callback C
```

This is why microtasks can have a significant effect on Event Loop responsiveness.

---

# 3. `process.nextTick()` behaves similarly in terms of priority

Consider:

```js
setTimeout(() => {
    console.log("Timer 1");

    process.nextTick(() => {
        console.log("nextTick");
    });
}, 0);

setTimeout(() => {
    console.log("Timer 2");
}, 0);
```

Conceptually:

```text
Timer 1
nextTick
Timer 2
```

Because after the first callback completes, Node processes the nextTick queue before continuing.

And remember the priority:

```text
process.nextTick()
       ↓
Promise microtasks
       ↓
next Event Loop callback
```

---

# 4. What if a microtask creates another microtask?

This is where things become interesting.

```js
Promise.resolve().then(() => {
    console.log("A");

    Promise.resolve().then(() => {
        console.log("B");
    });
});

setTimeout(() => {
    console.log("C");
}, 0);
```

Output:

```text
A
B
C
```

Why?

Because the first Promise creates another Promise microtask.

Node continues draining the microtask queue before moving on to the timer.

Conceptually:

```text
Microtask 1
    ↓
A
    ↓
Microtask 2 gets added
    ↓
B
    ↓
Microtask queue empty
    ↓
Timers
    ↓
C
```

This leads directly to the possibility of **microtask starvation**.

---

# 5. Microtask starvation

Consider:

```js
function loop() {
    Promise.resolve().then(loop);
}

loop();
```

Each Promise callback schedules another Promise callback.

So conceptually:

```text
Promise
  ↓
Promise
  ↓
Promise
  ↓
Promise
  ↓
...
```

If the microtask queue continuously remains non-empty, normal Event Loop work can be delayed.

That means things such as:

```text
Timers
I/O
setImmediate()
incoming requests
```

may have to wait.

So there are two related starvation risks:

```text
process.nextTick() recursion
        ↓
nextTick starvation

Promise microtask recursion
        ↓
microtask starvation
```

---

# 6. Important senior-level distinction

Don't say:

> "Promises always execute before everything."

That's too broad.

A better statement:

> **Promise callbacks are microtasks and are processed after the current JavaScript execution and before the Event Loop proceeds to subsequent normal callbacks/phases.**

And:

> **`process.nextTick()` has even higher priority than Promise microtasks in Node.js.**

---

# 7. Let's refine our mental model

Instead of this oversimplified model:

```text
JavaScript
   ↓
Microtasks
   ↓
Event Loop
```

Think:

```text
                Event Loop
                    │
                    ▼
             Event Loop callback
                    │
                    ▼
              callback finishes
                    │
                    ▼
        ┌──────────────────────┐
        │ process.nextTick()   │
        └──────────┬───────────┘
                   ↓
        ┌──────────────────────┐
        │ Promise microtasks   │
        └──────────┬───────────┘
                   ↓
          next callback/phase
```

And this can happen **repeatedly throughout Event Loop processing**.

This is the more useful senior-level mental model.

---

# 8. One terminology point

Earlier we said:

> "Microtasks execute before the Event Loop."

For beginner-level understanding, that's okay.

But now let's make it more precise:

❌ Don't say:

> "Microtasks execute before the Event Loop."

Because the Event Loop is continuously coordinating execution.

✅ Say:

> **"Microtasks are processed with priority after the current JavaScript callback completes, before Node proceeds to subsequent Event Loop work."**

That's much safer in a senior interview.

---

## Notes

Add this under `1.3.3`:

```text
### Microtask / nextTick Draining

Microtasks can be processed between individual callbacks,
not only after an entire Event Loop phase finishes.

Example:

Timer 1
   ↓
Promise microtask
   ↓
Timer 2

After a callback completes, Node processes:
1. process.nextTick() queue
2. Promise microtask queue
3. continues with subsequent Event Loop work


### Priority

Current callback
      ↓
process.nextTick()
      ↓
Promise microtasks
      ↓
next Event Loop callback


### Recursive microtasks

A microtask can schedule another microtask.

Promise 1
   ↓
Promise 2
   ↓
Promise 3
   ↓
...

Node continues processing microtasks while they remain
available before continuing normal Event Loop work.

Excessive/recursive microtasks can cause Event Loop starvation.


### Important

Microtasks are NOT an Event Loop phase.

They are processed with priority between normal callback
execution and subsequent Event Loop work.
```

---

## Checkpoint

Let's test whether this deeper model is clear.

### Q1

What is the output?

```js
setTimeout(() => {
    console.log("A");

    Promise.resolve().then(() => {
        console.log("B");
    });
}, 0);

setTimeout(() => {
    console.log("C");
}, 0);
```

### Q2

What is the output?

```js
setTimeout(() => {
    console.log("A");

    process.nextTick(() => {
        console.log("B");
    });

    Promise.resolve().then(() => {
        console.log("C");
    });
}, 0);

setTimeout(() => {
    console.log("D");
}, 0);
```

### Q3 — Senior

Why can this code be dangerous?

```js
function loop() {
    Promise.resolve().then(loop);
}

loop();
```

And most importantly:

**What kind of normal Node.js work can be delayed because of it?**

---
Good — **Q1 and Q2 are correct**, and Q3 has the right concept but needs a little more precision.

### Q1

```js
setTimeout(() => {
    console.log("A");

    Promise.resolve().then(() => {
        console.log("B");
    });
}, 0);

setTimeout(() => {
    console.log("C");
}, 0);
```

Your answer:

```text
A
B
C
```

✅ Correct.

Flow:

```text
Timer 1
  ↓
A
  ↓
Promise microtask
  ↓
B
  ↓
Timer 2
  ↓
C
```

---

### Q2

Your answer:

```text
A
B
C
D
```

✅ Correct.

The important priority is:

```text
Timer 1
   ↓
A
   ↓
nextTick → B
   ↓
Promise → C
   ↓
Timer 2
   ↓
D
```

So:

```text
process.nextTick()
       ↓
Promise microtask
       ↓
next callback
```

Excellent.

---

### Q3

You said:

> Event Loop starvation, Event Loop will never be executed.

✅ **Correct concept**, but let's make it interview-accurate.

The Event Loop itself isn't "never executed." Rather, **normal Event Loop work can be continuously delayed because the microtask queue keeps getting more work.**

This code:

```js
function loop() {
    Promise.resolve().then(loop);
}

loop();
```

creates:

```text
Microtask
   ↓
creates another Microtask
   ↓
creates another Microtask
   ↓
creates another Microtask
   ↓
...
```

So Node can keep processing microtasks and **starve normal Event Loop work**, such as:

* timers
* I/O callbacks
* `setImmediate()`
* incoming request handling

### Strong senior interview answer

> **This creates an effectively endless chain of Promise microtasks. Because Node keeps draining the microtask queue before continuing normal Event Loop work, timers, I/O callbacks, and other callbacks can be starved or severely delayed.**

---
### Next step

Next, we'll tackle **Promise scheduling in more depth**, including:

```text
Promise.then()
Promise.catch()
Promise.finally()
async/await
await scheduling
```

and how `async/await` fits into the **same Promise microtask model**.

That will complete the core **asynchronous execution model** before we move forward.

# `async/await` + Timers + `setImmediate()`

This is the point where all the pieces we've learned start coming together:

```text
Synchronous JS
      ↓
process.nextTick
      ↓
Promise microtasks
      ↓
Event Loop phases
      ↓
Timers / Poll / Check ...
```

## 1. Basic `await` + timer

```js
async function test() {
  console.log("A");

  await Promise.resolve();

  console.log("B");
}

console.log("C");

test();

setTimeout(() => {
  console.log("D");
}, 0);

console.log("E");
```

Output:

```text
C
A
E
B
D
```

### Why?

Synchronous:

```text
C
A
E
```

At `await`:

```text
Promise microtask:
[B]
```

Timer:

```text
Timers phase:
[D]
```

Microtasks are processed before normal Event Loop work:

```text
B
```

Then eventually:

```text
D
```

So:

```text
C → A → E → B → D
```

---

# 2. `await` + `setImmediate()`

```js
async function test() {
  console.log("A");

  await Promise.resolve();

  console.log("B");
}

console.log("C");

test();

setImmediate(() => {
  console.log("D");
});

console.log("E");
```

Output:

```text
C
A
E
B
D
```

Same fundamental reason:

```text
C A E
  ↓
Promise microtask
  ↓
B
  ↓
Event Loop
  ↓
setImmediate
  ↓
D
```

The important rule:

> **A Promise microtask scheduled during the current execution is processed before the Event Loop proceeds to normal timer/check callbacks.**

---

# 3. The interesting case: timer created after `await`

Now:

```js
async function test() {
  console.log("A");

  await Promise.resolve();

  console.log("B");

  setTimeout(() => {
    console.log("C");
  }, 0);
}

console.log("D");

test();

console.log("E");
```

Output:

```text
D
A
E
B
C
```

Flow:

```text
Synchronous
───────────
D
A
E

Promise microtask
─────────────────
B
 ↓
register timer C

Event Loop
──────────
C
```

So the timer doesn't execute while the Promise microtask is executing.

It becomes eligible for the **Timers phase** later.

---

# 4. `await` + timer + immediate

Now let's make it more interesting:

```js
async function test() {
  console.log("A");

  await Promise.resolve();

  console.log("B");

  setTimeout(() => {
    console.log("C");
  }, 0);

  setImmediate(() => {
    console.log("D");
  });
}

console.log("E");

test();

console.log("F");
```

Synchronous:

```text
E
A
F
```

Promise microtask:

```text
B
```

During `B`, we register:

```text
timer → C
immediate → D
```

Now the relative order between `C` and `D` depends on **where the Event Loop is when those callbacks are scheduled**.

Since this code is executing from the initial script's Promise microtask processing, the exact timer-vs-immediate ordering from the main module should **not be treated as a universal guaranteed `C → D` or `D → C` rule**.

The safe interview answer is:

```text
E → A → F → B → [C/D ordering depends on timing/context]
```

The important guaranteed relationship is:

```text
E → A → F → B
```

before those normal Event Loop callbacks.

---

# 5. Why I want you to be careful here

You already learned:

```text
setTimeout(0) vs setImmediate()
```

and the rule:

> Main module → ordering not guaranteed
> I/O callback → `setImmediate()` before `setTimeout(0)`

Don't replace that with:

> "`setImmediate()` always runs before a timer created inside an async function."

That's too broad.

`async/await` doesn't create a special Event Loop phase.

The important question is:

> **From which execution context are the timer and immediate being scheduled?**

That context determines the possible ordering.

---

# 6. Very important production connection

Suppose you have:

```js
async function handler() {
  const result = await databaseQuery();

  processResult(result);
}
```

While the DB query is pending:

```text
handler()
   ↓
databaseQuery()
   ↓
await
   ↓
handler pauses
   ↓
Node can handle other requests
```

When the DB query completes:

```text
DB result
   ↓
Promise settles
   ↓
handler continuation scheduled
   ↓
microtask
   ↓
processResult()
```

This is why asynchronous I/O doesn't block the Node.js JS thread.

But if you do:

```js
async function handler() {
  const result = await databaseQuery();

  while (true) {}
}
```

the `await` itself is fine.

The problem is what happens **after resumption**:

```text
await DB
  ↓
Promise settles
  ↓
microtask
  ↓
while(true)
  ↓
❌ JS thread blocked
```

So `async/await` does **not** magically make CPU-heavy code non-blocking.

That's a very common production misconception.

---

## Checkpoint

Now combine everything:

```js
async function test() {
  console.log("A");

  await Promise.resolve();

  console.log("B");

  setTimeout(() => {
    console.log("C");
  }, 0);

  Promise.resolve().then(() => {
    console.log("D");
  });

  console.log("E");
}

console.log("F");

test();

setTimeout(() => {
  console.log("G");
}, 0);

console.log("H");
```

### Your task

Give me:

1. **Exact output order**
2. The state of the **Promise microtask queue**
3. The state of the **timer queue**

Do it step-by-step rather than just guessing the final output.

---

**Rephrased:** You predicted:

`F → A → H → B → E → D → G → C`

Yes — **that is correct. ✅ 10/10**

Let's verify the queue reasoning, because this one tests exactly what we want.

### 1. Initial synchronous execution

```text
F
A
H
```

At `await Promise.resolve()` inside `test()`, the continuation is scheduled:

```text
Promise microtask:
[B → continue test()]
```

The external timer is also registered:

```text
Timer:
[G]
```

---

### 2. Resume `test()` via microtask

```text
B
```

Then inside that microtask:

```js
setTimeout(() => console.log("C"), 0);
```

adds:

```text
Timer:
[G, C]
```

Then:

```js
Promise.resolve().then(() => console.log("D"));
```

adds a new Promise microtask:

```text
Promise microtask:
[D]
```

Then synchronous execution **inside the current microtask** continues:

```text
E
```

So we have:

```text
Output:
F A H B E

Promise microtasks:
[D]

Timers:
[G, C]
```

---

### 3. Drain Promise microtasks

```text
D
```

Now there are no more Promise microtasks.

---

### 4. Timers phase

The timers were registered in this order:

```text
G
C
```

So:

```text
G
C
```

Final:

```text
F → A → H → B → E → D → G → C
```

### 🔑 What you've demonstrated here

You correctly combined:

* synchronous execution
* `await`
* Promise microtasks
* microtask FIFO
* timers
* timer registration order

And importantly:

> **A Promise microtask created while another Promise microtask is executing is processed before Node proceeds to normal Event Loop callbacks such as timers.**

So your Event Loop + microtask mental model is becoming solid.

---

### Next in

We've now covered most of the **scheduling mechanics**.

Next we'll handle the remaining important piece:

**`async/await` error handling + rejected Promises + `throw` + unhandled rejections.**

## 1.3.4 Promise Scheduling & `async/await`

We'll focus specifically on **how Promises and `async/await` are scheduled internally**, building directly on the microtask concepts we just covered.

### 1. Promise `.then()` is a microtask

Consider:

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
Synchronous JS
    ↓
console.log("A")
    ↓
register .then()
    ↓
console.log("C")
    ↓
current execution finishes
    ↓
Promise reaction runs as microtask
    ↓
"B"
```

Important:

> Calling `.then()` does **not** execute its callback immediately. It schedules the callback as a Promise microtask.

---

### 2. What about `.catch()` and `.finally()`?

They follow the same scheduling model.

```js
Promise.reject("error")
  .catch(() => {
    console.log("catch");
  });

console.log("end");
```

Output:

```text
end
catch
```

`catch()` callback is also processed through the Promise microtask mechanism.

Similarly:

```js
Promise.resolve()
  .finally(() => {
    console.log("finally");
  });

console.log("end");
```

Output:

```text
end
finally
```

So for your mental model:

```text
.then()     → Promise microtask
.catch()    → Promise microtask
.finally()  → Promise microtask
```

---

## 3. Now the important part: `async`

Consider:

```js
async function test() {
  console.log("inside");
}

console.log("A");

test();

console.log("B");
```

Output:

```text
A
inside
B
```

This is an important interview point.

An `async` function **does not automatically make its entire body asynchronous**.

The function starts executing synchronously.

```text
A
 ↓
test()
 ↓
"inside"
 ↓
B
```

The special Promise/microtask behavior becomes important when we encounter `await`.

---

# 4. What happens at `await`?

This is one of the most important concepts in Node.js async interviews.

Consider:

```js
async function test() {
  console.log("A");

  await Promise.resolve();

  console.log("B");
}

test();

console.log("C");
```

Output:

```text
A
C
B
```

Why?

Execution starts:

```text
test()
  ↓
console.log("A")
  ↓
await Promise.resolve()
```

At `await`, the function's continuation is scheduled through the Promise microtask mechanism.

The function effectively says:

> "I cannot continue from here right now. Continue me asynchronously when the awaited Promise is settled."

So execution returns to the surrounding synchronous code:

```text
console.log("C")
```

Then the microtask runs:

```text
console.log("B")
```

Mental model:

```text
async function
     ↓
execute synchronously
     ↓
encounter await
     ↓
pause function continuation
     ↓
schedule continuation as Promise microtask
     ↓
return control to current JS execution
     ↓
current JS finishes
     ↓
microtask runs
     ↓
resume after await
```

---

# 5. `await` does NOT block the Event Loop

This distinction is extremely important for a senior Node.js interview.

```js
await somePromise();
```

does **not** mean:

> "Block Node.js until the Promise finishes."

Instead:

> "`await` suspends the current async function and allows the JavaScript thread to continue processing other work."

For example:

```js
async function handler() {
  const data = await fetchData();

  console.log(data);
}

console.log("A");

handler();

console.log("B");
```

Conceptually:

```text
A
 ↓
handler()
 ↓
fetchData()
 ↓
await
 ↓
handler pauses
 ↓
B
 ↓
...
 ↓
Promise settles
 ↓
microtask
 ↓
handler resumes
 ↓
console.log(data)
```

This is why `async/await` can be used for non-blocking I/O.

---

## 6. Important distinction: synchronous blocking vs `await`

### Synchronous blocking

```js
while (true) {}
```

The main JS thread is stuck.

Nothing else can execute.

### `await`

```js
await somePromise;
```

The current async function pauses, **but the JS thread is free to execute other work**.

That's a very important distinction:

```text
while(true)
   ↓
blocks JS thread ❌

await Promise
   ↓
suspends current async function
   ↓
JS thread remains available ✅
```

---

## Checkpoint — don't answer yet with explanation

Predict the output:

```js
async function test() {
  console.log("A");

  await Promise.resolve();

  console.log("B");
}

console.log("C");

test();

console.log("D");

Promise.resolve().then(() => {
  console.log("E");
});
```

What is the exact output order?

**Answer it first; then I'll walk through the microtask ordering step-by-step.**

---

**Rephrased:** You predicted the output as `C → A → D → B → E`.

You're **correct**. ✅ **9.5/10**

```text
C
A
D
B
E
```

### Why?

```js
console.log("C");
```

Runs synchronously → `C`

Then:

```js
test();
```

Inside `test()`:

```js
console.log("A");
```

Runs synchronously → `A`

Then:

```js
await Promise.resolve();
```

The function pauses. Its continuation (`console.log("B")`) is scheduled as a **Promise microtask**.

Control returns to the caller:

```js
console.log("D");
```

→ `D`

Then this is registered:

```js
Promise.resolve().then(() => {
  console.log("E");
});
```

This creates another Promise microtask.

So now the microtask queue is conceptually:

```text
[B continuation]
[E callback]
```

Microtasks are processed in FIFO order:

```text
B
E
```

Therefore:

```text
C → A → D → B → E
```

### Important senior-level takeaway

`await` doesn't simply mean:

> "Run the rest of the function later."

More precisely:

> **When execution reaches `await`, the async function's continuation is scheduled through the Promise microtask mechanism, and the function yields control back to the surrounding JavaScript execution.**

This distinction becomes particularly important when multiple `await`s and Promise callbacks interact.

---

### One more important case

Consider:

```js
async function test() {
  console.log("A");

  await Promise.resolve();

  console.log("B");

  await Promise.resolve();

  console.log("C");
}

console.log("D");

test();

console.log("E");
```

The output is:

```text
D
A
E
B
C
```

Notice that **each `await` creates another suspension/resumption point**.

We'll go deeper into this next, because it is a common senior-level interview area: **how `await` maps conceptually to Promise continuations and why multiple `await`s create multiple microtask steps.**

---

**Rephrasing your question:**
When we reach `await Promise`, does `await` immediately execute the Promise/microtask, or does it schedule/register the continuation and execute it only after the current synchronous JavaScript execution finishes?

Yes — **your second understanding is correct.** The important distinction is:

> **`await` does not execute the continuation immediately. It schedules the continuation to resume through the Promise microtask mechanism. That continuation runs after the current synchronous JavaScript execution yields.**

For example:

```js
async function test() {
  console.log("A");

  await Promise.resolve();

  console.log("B");
}

console.log("C");
test();
console.log("D");
```

Execution:

```text
C
 ↓
test()
 ↓
A
 ↓
await Promise.resolve()
 ↓
register/schedule continuation
 ↓
return control to current synchronous execution
 ↓
D
 ↓
current synchronous execution finishes
 ↓
Promise microtask executes
 ↓
B
```

Output:

```text
C
A
D
B
```

### One subtle but important correction

Don't think of it exactly as:

```text
await Promise
     ↓
"register the Promise itself as a microtask"
```

The **Promise is not what gets put into the microtask queue**.

More accurately:

```text
await Promise
      ↓
Promise is/gets settled
      ↓
continuation after await
      ↓
scheduled as a Promise reaction/microtask
```

So with:

```js
await Promise.resolve();
console.log("B");
```

the thing that eventually runs as the microtask is essentially the **continuation of the async function** — the work represented by:

```js
console.log("B");
```

### Compare `.then()` and `await`

These are conceptually very similar:

```js
Promise.resolve().then(() => {
  console.log("B");
});
```

and:

```js
await Promise.resolve();
console.log("B");
```

Both involve a **Promise continuation that runs through the microtask mechanism**.

The key mental model for your notes:

```text
await Promise
     ↓
pause current async function
     ↓
schedule continuation via Promise microtask
     ↓
continue current synchronous JS
     ↓
current JS callback finishes
     ↓
microtasks are drained
     ↓
resume async function
```

And this is why we say:

> **`await` pauses the async function, not the JavaScript thread.**

That distinction is very important for senior Node.js interviews.



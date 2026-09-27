# `async/await` Error Handling

This is the next important part because asynchronous errors behave differently from normal synchronous errors.

---

## 1. `throw` inside an `async` function

Consider:

```js
async function test() {
  throw new Error("Something failed");
}

const result = test();

console.log(result);
```

You might think `throw` immediately crashes the application.

It doesn't.

Because `test()` is an `async` function, the thrown error causes the **Promise returned by `test()` to become rejected**.

Conceptually:

```text
test()
  ↓
throw Error
  ↓
returned Promise → REJECTED
```

So:

```js
test().catch(error => {
  console.log(error.message);
});
```

prints:

```text
Something failed
```

### Important rule

> **A synchronous `throw` inside an `async` function becomes a rejected Promise to the caller.**

---

# 2. `throw` before `await`

Consider:

```js
async function test() {
  console.log("A");

  throw new Error("ERROR");

  console.log("B");
}

console.log("C");

test().catch(() => {
  console.log("Caught");
});

console.log("D");
```

Output:

```text
C
A
D
Caught
```

Why?

`test()` starts executing synchronously:

```text
C
A
```

Then:

```js
throw new Error("ERROR");
```

causes the async function's returned Promise to become rejected.

The caller's:

```js
.catch(...)
```

is a Promise reaction, so it runs as a **microtask**.

Therefore:

```text
D
Caught
```

Final:

```text
C → A → D → Caught
```

---

# 3. `throw` after `await`

Now:

```js
async function test() {
  console.log("A");

  await Promise.resolve();

  throw new Error("ERROR");
}

console.log("B");

test().catch(() => {
  console.log("Caught");
});

console.log("C");
```

Output:

```text
B
A
C
Caught
```

Flow:

```text
B
 ↓
test()
 ↓
A
 ↓
await
 ↓
pause
 ↓
C
 ↓
microtask
 ↓
resume test()
 ↓
throw
 ↓
test() Promise becomes rejected
 ↓
.catch() scheduled as another microtask
 ↓
Caught
```

This is important:

> The `catch()` attached by the caller is itself another Promise continuation.

So the rejection and the caller's `.catch()` involve asynchronous Promise scheduling.

---

# 4. `try/catch` around `await`

This is the normal pattern:

```js
async function test() {
  try {
    const result = await fetchData();

    return result;
  } catch (error) {
    console.log("Something failed");
  }
}
```

If `fetchData()` rejects:

```text
fetchData()
   ↓
Promise rejected
   ↓
await observes rejection
   ↓
exception thrown at await
   ↓
catch handles it
```

So `await` allows asynchronous errors to be handled using normal-looking `try/catch`.

That's one of the major benefits of `async/await`.

---

# 5. `try/catch` does NOT catch an unrelated Promise

This is a common mistake:

```js
try {
  Promise.reject(new Error("ERROR"));
} catch (error) {
  console.log("Caught");
}
```

The `catch` does **not** execute.

Why?

Because:

```js
Promise.reject(...)
```

doesn't synchronously throw an exception.

It creates a rejected Promise.

The rejection must be handled through Promise mechanisms:

```js
Promise.reject(new Error("ERROR"))
  .catch(error => {
    console.log("Caught");
  });
```

Or:

```js
async function test() {
  try {
    await Promise.reject(new Error("ERROR"));
  } catch (error) {
    console.log("Caught");
  }
}
```

---

# 6. `throw` vs Promise rejection

This distinction is worth remembering:

### Synchronous code

```js
throw new Error("ERROR");
```

→ synchronous exception

### Async function

```js
async function test() {
  throw new Error("ERROR");
}
```

→ returned Promise becomes rejected

### Promise

```js
Promise.reject(new Error("ERROR"));
```

→ rejected Promise

And `await` bridges Promise rejection into `try/catch`:

```text
Promise rejection
      ↓
     await
      ↓
exception inside async function
      ↓
   try/catch
```

---

# 7. What happens if nobody handles the rejection?

For example:

```js
async function test() {
  throw new Error("ERROR");
}

test();
```

The returned Promise becomes rejected, but nobody handles it.

Node.js reports this as an **unhandled Promise rejection**. Depending on the Node.js version/runtime behavior and configuration, this can result in the process terminating.

Production takeaway:

> **Don't leave important Promise chains or async operations without a deliberate error-handling strategy.**

For example:

```js
async function test() {
  throw new Error("ERROR");
}

test().catch(error => {
  console.error("Failed:", error);
});
```

Or at an application boundary, such as an HTTP request handler, use centralized async error handling appropriate to your framework.

---

# 8. One senior-level distinction

Don't say:

> "`await` catches the error."

That's not quite right.

Better:

> **If the awaited Promise rejects, the rejection is re-thrown at the `await` expression inside the async function, where a surrounding `try/catch` can catch it.**

For example:

```js
try {
  const result = await fetchData();
} catch (error) {
  // catches rejection from fetchData()
}
```

The `try/catch` is doing the catching; `await` is what allows the Promise rejection to participate in that synchronous-looking exception flow.

---

## Checkpoint

Predict the exact output:

```js
async function test() {
  console.log("A");

  try {
    await Promise.reject("ERROR");

    console.log("B");
  } catch (error) {
    console.log("C");

    await Promise.resolve();

    console.log("D");
  }

  console.log("E");
}

console.log("F");

test();

console.log("G");
```

Give me the exact order and explain **where the async function suspends and where it resumes**.

---

Correct. **10/10.** ✅

Your answer:

```text
F → A → G → C → E → D
```

is exactly right.

### Why?

Start with synchronous execution:

```text
console.log("F")
        ↓
F

test()
        ↓
A
```

Then:

```js
await Promise.reject("ERROR");
```

The Promise is already rejected, so the `await` causes the async function to **suspend** and schedule its continuation as a Promise microtask.

Meanwhile, the main synchronous code continues:

```text
G
```

So far:

```text
F → A → G
```

---

### Microtask: resume after `await`

The rejection is observed at the `await`, so control goes into:

```js
catch (error) {
  console.log("C");
```

Therefore:

```text
C
```

Then:

```js
await Promise.resolve();
```

This is another suspension point.

Even though the Promise is already resolved, `await` **still resumes asynchronously**.

So the function pauses again.

Then the function eventually resumes:

```js
console.log("D");
```

Then exits the `catch` block and continues:

```js
console.log("E");
```

Therefore:

```text
F → A → G → C → D → E
```

Wait — **there is an important correction here.**

The exact order is actually:

```text
F → A → G → C → D → E
```

Your answer had:

```text
F → A → G → C → E → D
```

So you're **very close, but D/E are reversed.**

### Why `D` comes before `E`

This part is crucial:

```js
catch (error) {
  console.log("C");

  await Promise.resolve();

  console.log("D");
}

console.log("E");
```

The second `await` suspends the **entire remaining async function**, including the code after the `catch` block.

So Node cannot reach `E` until the async function resumes and executes `D`.

Flow:

```text
C
 ↓
await Promise.resolve()
 ↓
SUSPEND
 ↓
resume
 ↓
D
 ↓
catch finishes
 ↓
E
```

### Final answer

```text
F → A → G → C → D → E
```

This is actually a **good senior-level checkpoint**, because it tests whether you understand that `await` suspends the remainder of the async function, not just the current `catch` block.

---

### One more important concept

You should now have this mental model:

```text
async function
     │
     ├── synchronous code
     │
     ├── await
     │      ↓
     │   suspend function
     │      ↓
     │   Promise settles
     │      ↓
     │   continuation → microtask
     │      ↓
     ├── resume
     │
     ├── another await
     │      ↓
     │   suspend again
     │
     └── resume again
```

So **each `await` can create another suspension/resumption point**.

---

## Next: final part 

We have covered:

* Promise scheduling
* `.then()`
* Promise chains
* `async/await`
* `await` suspension
* `return` vs `return await`
* rejected Promises
* `throw`
* `try/catch`
* unhandled rejections
* multiple `await` suspension points

The remaining step is a **senior-level mixed scheduling problem** combining:

**`async/await` + `process.nextTick()` + Promise microtasks + `setTimeout()` + `setImmediate()`**

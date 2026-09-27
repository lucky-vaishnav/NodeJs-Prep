# `return` vs `return await`

This is a subtle but useful senior-level Node.js concept.

Consider:

```js
async function test() {
  return Promise.resolve("hello");
}
```

An `async` function **always returns a Promise**.

So:

```js
const result = test();

console.log(result);
```

`result` is a Promise.

---

## 1. `return Promise`

```js
async function test() {
  return Promise.resolve("hello");
}
```

Conceptually:

```text
test()
  ↓
returns a Promise
  ↓
that Promise resolves to "hello"
```

You can consume it:

```js
test().then(value => {
  console.log(value);
});
```

Output:

```text
hello
```

---

# 2. `return await Promise`

Now:

```js
async function test() {
  return await Promise.resolve("hello");
}
```

Here:

```text
await Promise.resolve("hello")
        ↓
pause async function
        ↓
resume through Promise microtask
        ↓
return "hello"
```

The async function still returns a Promise to its caller.

So both:

```js
return Promise.resolve("hello");
```

and:

```js
return await Promise.resolve("hello");
```

eventually produce a Promise that resolves to:

```text
hello
```

---

# 3. Then why does `return await` matter?

The biggest practical difference appears with **`try/catch`**.

Consider:

```js
async function test() {
  try {
    return Promise.reject(new Error("Failed"));
  } catch (error) {
    console.log("Caught");
  }
}
```

You might expect:

```text
Caught
```

But that's **not what happens**.

The `catch` does not catch the rejection here.

Why?

Because:

```js
return Promise.reject(...)
```

returns the rejected Promise from the `try` block.

The rejection happens **asynchronously**, after the `try/catch` execution has already finished.

---

## 4. `return await` changes this

```js
async function test() {
  try {
    return await Promise.reject(new Error("Failed"));
  } catch (error) {
    console.log("Caught");
  }
}
```

Now:

```text
await Promise.reject(...)
        ↓
async function suspends
        ↓
Promise rejection
        ↓
async function resumes with rejection
        ↓
rejection is thrown at await
        ↓
catch catches it
```

Output:

```text
Caught
```

### This is the important distinction

```js
return Promise.reject(error);
```

means approximately:

> "Return this Promise; let the caller handle its rejection."

Whereas:

```js
return await Promise.reject(error);
```

means:

> "Wait for this Promise inside this function, so a local `try/catch` can handle the rejection."

---

# 5. Senior interview rule

Don't say:

> "`return await` is always better."

That's incorrect.

In modern JavaScript, if you **don't need local error handling**, this is generally unnecessary:

```js
async function getData() {
  return await fetchData();
}
```

You can normally write:

```js
async function getData() {
  return fetchData();
}
```

The Promise can simply be returned to the caller.

But if you need the local `try/catch` to handle the rejection:

```js
async function getData() {
  try {
    return await fetchData();
  } catch (error) {
    // local handling
  }
}
```

then `return await` can be meaningful.

---

# 6. Important connection to what we just learned

This also connects directly to our microtask discussion.

With:

```js
async function test() {
  return await Promise.resolve("A");
}
```

the `await` introduces an asynchronous continuation.

So conceptually:

```text
test()
 ↓
await Promise
 ↓
suspend
 ↓
Promise microtask
 ↓
resume
 ↓
return "A"
```

Whereas:

```js
async function test() {
  return Promise.resolve("A");
}
```

doesn't require the function to suspend at an `await`.

---

## One correction to keep in your notes

Avoid memorizing:

> "`return await` creates an extra Promise."

That's not the useful mental model.

The useful model is:

> **`return await` makes the async function wait for the Promise before returning from that function, which is especially relevant when local `try/catch` or `finally` needs to observe the Promise's settlement.**

---

### Checkpoint

What will this print?

```js
async function test() {
  try {
    return Promise.reject("ERROR");
  } catch (error) {
    return "CAUGHT";
  }
}

test().then(value => {
  console.log(value);
}).catch(error => {
  console.log(error);
});
```

And then compare it with:

```js
async function test() {
  try {
    return await Promise.reject("ERROR");
  } catch (error) {
    return "CAUGHT";
  }
}

test().then(value => {
  console.log(value);
}).catch(error => {
  console.log(error);
});
```

Tell me the output of **both cases**.

---
**Rephrased:** You’re saying:

* **Case 1:** `ERROR`
* **Case 2:** `CAUGHT`

Exactly right. ✅ **10/10**

### Case 1

```js id="q1k5ty"
async function test() {
  try {
    return Promise.reject("ERROR");
  } catch (error) {
    return "CAUGHT";
  }
}
```

`return Promise.reject()` returns the rejected Promise **without awaiting it**.

So the `try/catch` finishes before the rejection is observed locally.

Result:

```text id="3j5y8v"
ERROR
```

The caller's `.catch()` receives it.

---

### Case 2

```js id="9r0n3c"
async function test() {
  try {
    return await Promise.reject("ERROR");
  } catch (error) {
    return "CAUGHT";
  }
}
```

Here `await` makes the async function suspend.

When the rejection is observed at the `await`, it becomes an exception **inside `test()`**, so the local `catch` handles it.

Result:

```text id="6iqf6h"
CAUGHT
```

Then `test()` resolves with `"CAUGHT"`, so:

```js id="qk4z3p"
.then(...)
```

runs.

### The clean mental model

```text id="v9j2x4"
return Promise
    ↓
pass Promise/rejection to caller

return await Promise
    ↓
wait for settlement inside this function
    ↓
local try/catch can handle rejection
```

One small terminology point: in Case 2, `.then()` isn't **catching** the error. The `try/catch` inside `test()` catches it first, and then `test()` resolves successfully with `"CAUGHT"`, which is why `.then()` receives it.

---

**Next:** Promise chains and how multiple `.then()` callbacks are placed into the microtask queue. This will tie together everything we've learned so far about microtask ordering.



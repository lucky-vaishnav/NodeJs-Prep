## Scheduling Exercise

This one combines everything we've learned so far.

Before solving, remember the priority model we're using:

```text
Current synchronous JS
        ↓
process.nextTick()
        ↓
Promise microtasks / async continuations
        ↓
Event Loop callbacks
   ├── Timers
   ├── Poll
   └── Check (setImmediate)
```

And remember the important nuance:

> `await` suspends the **rest of that async function**. It does not block the JavaScript thread.

### Exercise

Predict the **exact output order**:

```js
async function test() {
  console.log("A");

  await Promise.resolve();

  console.log("B");

  process.nextTick(() => {
    console.log("C");
  });

  Promise.resolve().then(() => {
    console.log("D");
  });

  setTimeout(() => {
    console.log("E");
  }, 0);

  setImmediate(() => {
    console.log("F");
  });

  console.log("G");
}

console.log("H");

test();

process.nextTick(() => {
  console.log("I");
});

Promise.resolve().then(() => {
  console.log("J");
});

setTimeout(() => {
  console.log("K");
}, 0);

setImmediate(() => {
  console.log("L");
});

console.log("M");
```

### Your task

Give me:

1. **Exact output order**
2. Briefly explain:

   * where `test()` suspends
   * when it resumes
   * when `C` and `D` are scheduled
   * how `E/F` relate to `K/L`

Take your time with the queues. The tricky part is **not just knowing the priority — it's knowing when each callback gets added to its queue.**

---

You're very close on the **overall queue reasoning**, but there are **two important corrections**.

Your answer:

```text
H → A → M → B → J → D → I → C → K/L → E/F
```

The issue is the ordering of **`I`, `J`, `C`, `D`**, and the fact that the timer/immediate groups have a deterministic registration order.

Let's walk it carefully.

### Step 1 — Initial synchronous execution

```js
console.log("H");
test();
process.nextTick(I);
Promise.resolve().then(J);
setTimeout(K, 0);
setImmediate(L);
console.log("M");
```

`test()` starts synchronously:

```text
H
A
```

Then it reaches:

```js
await Promise.resolve();
```

So `test()` suspends.

The rest of the main script continues:

```text
M
```

At this point:

```text
H → A → M
```

---

### Step 2 — Microtask / nextTick processing

We have:

```text
nextTick queue:
I

Promise microtask queue:
test continuation (B)
J
```

Because `process.nextTick()` has priority, **`I` runs before the Promise microtasks**.

So:

```text
I
```

Then the Promise microtask queue begins.

The continuation of `test()` runs:

```text
B
```

During this continuation, it schedules:

```text
nextTick → C
Promise microtask → D
timer → E
immediate → F
```

And prints:

```text
G
```

So now:

```text
H → A → M → I → B → G
```

The important subtlety is that **`C` does not interrupt the currently draining Promise microtask sequence**. `D` is already a Promise microtask and is processed as part of that drain.

The existing `J` was already ahead of newly-added `D`, so:

```text
J → D
```

Then the newly scheduled nextTick `C` is processed at the appropriate subsequent nextTick checkpoint.

So the sequence becomes:

```text
H → A → M → I → B → G → J → D → C
```

---

### Step 3 — Timers and immediates

We initially registered:

```text
K
L
```

Later, inside `test()`:

```text
E
F
```

Therefore within their respective categories:

```text
Timers:
K → E

Check:
L → F
```

The exact **timer phase vs check phase ordering from the top-level script is not something you should represent as a guaranteed `K/L` ordering**.

So your `K/L` and `E/F` grouping was directionally correct, but the useful guaranteed relationships are:

```text
K before E
L before F
```

### Final conceptual ordering

```text
H → A → M → I → B → G → J → D → C → [Timers / Check]
```

with:

```text
Timers: K → E
Check:  L → F
```

The exact interleaving of the Timer and Check groups depends on where the Event Loop begins relative to timer eligibility.

---

## One correction to lock into your notes

The key thing you were testing is:

```text
test()
  ↓
await
  ↓
test continuation becomes Promise microtask
  ↓
resume
  ↓
schedule nextTick(C) + Promise(D)
  ↓
current Promise-microtask draining continues
  ↓
D
  ↓
nextTick(C) gets its next opportunity
```

So don't use a simplistic rule like:

> "Every time `process.nextTick()` is scheduled, it immediately jumps ahead of all Promise microtasks."

That's **too broad**.

The exact Node.js interaction between the nextTick queue and Promise microtask queue depends on the checkpoint at which work is being drained.

---

# ✅ Promise Scheduling & `async/await` — COMPLETED

You now have covered the major senior-level pieces:

* Promise microtask scheduling
* `.then()`, `.catch()`, `.finally()`
* Promise chains
* `async/await`
* `await` suspension/resumption
* Already-resolved vs pending Promises
* Multiple `await`s
* `return` vs `return await`
* `throw` inside async functions
* Rejected Promises
* `try/catch` with `await`
* Unhandled Promise rejections
* Interaction with timers
* Interaction with `process.nextTick()`
* Mixed scheduling problems

## 1. Multiple `await`s

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

Output:

```text
D
A
E
B
C
```

You already understand the first `await`:

```text
D
 ↓
test()
 ↓
A
 ↓
await #1
 ↓
pause test()
 ↓
E
 ↓
microtask → resume test()
 ↓
B
```

Now something important happens.

When we reach:

```js
await Promise.resolve();

console.log("C");
```

the function **pauses again**.

So there is another Promise continuation that needs to be scheduled.

Conceptually:

```text
Synchronous execution
        │
        ├── D
        ├── test()
        │    └── A
        │        └── await #1
        │             ↓
        └── E
             ↓
       Microtask queue
             │
             └── Resume test()
                    ↓
                    B
                    ↓
                  await #2
                    ↓
              schedule another
              microtask continuation
                    ↓
              next microtask
                    ↓
                    C
```

Therefore:

```text
D → A → E → B → C
```

### Key idea

Each `await` is a **potential suspension point**.

For example:

```js
await promise1;
console.log("A");

await promise2;
console.log("B");

await promise3;
console.log("C");
```

Conceptually:

```text
start
 ↓
await promise1
 ↓
resume microtask
 ↓
A
 ↓
await promise2
 ↓
resume another microtask
 ↓
B
 ↓
await promise3
 ↓
resume another microtask
 ↓
C
```

So don't think:

> "`async` function becomes one microtask."

Instead:

> **An async function executes synchronously until it reaches an `await` that causes suspension. Its continuation is then resumed through Promise microtask scheduling. Every subsequent suspension can create another continuation.**

---

# 2. What if the Promise is already resolved?

This is a very common interview trap.

You might think:

```js
await Promise.resolve();
```

means:

> "The Promise is already resolved, so continue immediately."

**No.**

Even when the Promise is already fulfilled, `await` still resumes the async function **asynchronously through the Promise microtask mechanism**.

Example:

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

Not:

```text
A
B
C
```

So:

```text
Promise already resolved
        ↓
await
        ↓
still suspend current async function
        ↓
schedule continuation
        ↓
current synchronous execution continues
        ↓
microtask
        ↓
resume
```

This is an important rule to remember:

> **An already-fulfilled Promise does not make `await` synchronous.**

---

# 3. What if the Promise is pending?

Now:

```js
async function test() {
  console.log("A");

  const result = await fetchData();

  console.log(result);
}
```

Suppose `fetchData()` takes 2 seconds.

The flow is roughly:

```text
test()
 ↓
A
 ↓
await pending Promise
 ↓
pause test()
 ↓
return control
 ↓
other Node.js work can execute
 ↓
...
 ↓
Promise settles
 ↓
schedule continuation
 ↓
microtask
 ↓
resume test()
 ↓
console.log(result)
```

This is why `await` is particularly useful with I/O.

The important distinction is:

### Already fulfilled

```js
await Promise.resolve();
```

Still resumes asynchronously through a microtask.

### Pending

```js
await somePendingPromise;
```

The function waits for the Promise to settle, **without blocking the JS thread**, and then its continuation is scheduled.

---

# 4. What if we `await` a normal value?

Another interesting case:

```js
async function test() {
  console.log("A");

  const value = await 100;

  console.log(value);
}

console.log("B");

test();

console.log("C");
```

Output:

```text
B
A
C
100
```

Why?

Even though `100` isn't a Promise, `await` effectively treats the value as an already-fulfilled Promise-like result for scheduling purposes.

Conceptually:

```text
await 100
   ↓
treat result as fulfilled Promise
   ↓
schedule continuation
   ↓
current synchronous execution continues
   ↓
microtask
   ↓
100
```

Therefore:

> **`await` always introduces asynchronous resumption, even when you await an already-resolved Promise or a plain value.**

That's a very useful interview rule.

---

## Checkpoint

Predict the **exact output order**:

```js
async function test() {
  console.log("A");

  await 100;

  console.log("B");

  await Promise.resolve();

  console.log("C");
}

console.log("D");

test();

console.log("E");

Promise.resolve().then(() => {
  console.log("F");
});
```

Take it step-by-step:

1. What runs synchronously?
2. What gets scheduled at the first `await`?
3. What is already in the microtask queue when `test()` resumes?
4. What happens at the second `await`?

Give me the output order, and we'll verify your reasoning rather than just the final answer.

---

**Rephrased:** You predicted the output as **`D → A → E → B → C → F`**.

You're **very close, but there is one important microtask-ordering issue.** The correct output is:

```text
D
A
E
B
F
C
```

### Let's trace it carefully

Initial synchronous execution:

```js
console.log("D"); // D
test();
```

Inside `test()`:

```js
console.log("A"); // A
await 100;
```

`await 100` schedules the continuation (`B...`) as a Promise microtask.

Then control returns:

```js
console.log("E"); // E
```

Then this is registered:

```js
Promise.resolve().then(() => {
  console.log("F");
});
```

So at this point, the microtask queue is:

```text
1. Resume test() → B
2. F
```

So far:

```text
D → A → E
```

### Microtask #1

Resume `test()`:

```js
console.log("B"); // B
```

Then:

```js
await Promise.resolve();
```

This **suspends `test()` again** and schedules its next continuation (`C`) as a **new microtask**.

But remember: `F` was already waiting in the queue.

So now:

```text
Microtask queue:

F
C
```

Therefore:

```text
F
C
```

Final:

```text
D → A → E → B → F → C
```

### 🔑 Important lesson

This is exactly why understanding **microtask queue ordering** matters.

When a running microtask schedules another microtask, the new microtask is added to the queue; it doesn't jump ahead of microtasks that were already waiting.

Conceptually:

```text
Initial queue:
[B, F]

Execute B:
  B schedules C

Queue becomes:
[F, C]

Execute F
Execute C
```

So your first part was completely right. The mistake was assuming that because `C` belongs to the same async function, it would immediately continue before `F`.

**This is a very good senior-level output question.**

# Promise Chains & Microtask Scheduling

This is important because a common interview mistake is thinking:

> "All `.then()` callbacks in a Promise chain are already sitting in the microtask queue."

They aren't.

Each `.then()` continuation is generally scheduled **when the previous Promise settles**, so later `.then()` callbacks may be added to the queue only after earlier ones execute.

---

## 1. Basic Promise chain

```js
Promise.resolve()
  .then(() => {
    console.log("A");
  })
  .then(() => {
    console.log("B");
  })
  .then(() => {
    console.log("C");
  });

console.log("D");
```

Output:

```text
D
A
B
C
```

At first:

```text
Synchronous:
D
```

Then the first `.then()` is scheduled:

```text
Microtask queue:
[A]
```

Execute `A`.

But the important part is that the next `.then()`:

```js
.then(() => console.log("B"))
```

is attached to the Promise returned by the previous `.then()`.

So after `A` completes:

```text
A's Promise settles
       ↓
B continuation gets scheduled
       ↓
microtask queue
       ↓
B
```

Then the same thing happens for `C`.

Conceptually:

```text
Initial:
[A]

Execute A:
[B]

Execute B:
[C]

Execute C:
[]
```

So the chain progresses **one Promise continuation at a time**.

---

# 2. Compare independent `.then()` callbacks

Now look at this:

```js
Promise.resolve().then(() => {
  console.log("A");
});

Promise.resolve().then(() => {
  console.log("B");
});

console.log("C");
```

Output:

```text
C
A
B
```

Here both Promise callbacks are scheduled during the initial synchronous execution:

```text
Microtask queue:

[A, B]
```

So they execute FIFO:

```text
A
B
```

### Difference

#### Promise chain

```text
A → B → C
```

Each next callback depends on the previous Promise settling.

#### Independent Promises

```text
A
B
```

Both can already be queued during synchronous execution.

This distinction is extremely useful for output questions.

---

# 3. The interesting case

Consider:

```js
Promise.resolve()
  .then(() => {
    console.log("A");

    Promise.resolve().then(() => {
      console.log("B");
    });
  })
  .then(() => {
    console.log("C");
  });
```

Let's trace it.

Initially:

```text
Microtask queue:
[first then]
```

Execute first microtask:

```text
A
```

Inside it, we create:

```js
Promise.resolve().then(() => console.log("B"));
```

So now `B` gets added to the queue.

But when the first `.then()` finishes successfully, its **next chained `.then()`** also becomes eligible and is scheduled.

The important ordering is:

```text
After A:

Microtask queue:
[B, C]
```

Therefore:

```text
A
B
C
```

### Key lesson

A microtask created **inside** a running microtask is appended to the queue.

It doesn't automatically jump to the front.

---

# 4. Promise chain + `setTimeout`

Now combine everything:

```js
Promise.resolve()
  .then(() => {
    console.log("A");
  })
  .then(() => {
    console.log("B");
  });

setTimeout(() => {
  console.log("C");
}, 0);

console.log("D");
```

Output:

```text
D
A
B
C
```

Why?

Initial synchronous:

```text
D
```

Then Promise microtask:

```text
A
```

`A` schedules the next Promise continuation:

```text
B
```

Microtasks are processed before normal timer callback execution:

```text
C
```

So:

```text
D → A → B → C
```

---

# 5. Promise chain + `process.nextTick()`

Now the Node-specific version:

```js
Promise.resolve()
  .then(() => {
    console.log("A");

    process.nextTick(() => {
      console.log("B");
    });
  })
  .then(() => {
    console.log("C");
  });
```

The conceptual ordering is:

```text
A
C
B
```

Why?

After `A` executes:

```text
process.nextTick(B)
```

is registered.

The next Promise continuation `C` is also scheduled.

But there's an important nuance here: **`process.nextTick()` has higher priority than Promise microtasks when Node reaches its next microtask checkpoint**, but a `nextTick` scheduled *during* a Promise microtask does not necessarily interrupt the remainder of the Promise microtask queue immediately. In this example, the next chained Promise reaction can run before the newly scheduled `nextTick`.

So don't use the oversimplified rule:

> "`nextTick` always immediately jumps ahead of every Promise microtask."

The safer senior-level model is:

> **Node processes the `process.nextTick` queue with higher priority than the Promise microtask queue at its microtask checkpoints, but scheduling context matters when callbacks are already being drained.**

We'll revisit this nuance later when we do the combined **`nextTick` + Promise + timer + `setImmediate`** interview questions.

---

# 6. The key mental model

For Promise chains, remember:

```text
.then()
   ↓
Promise settles
   ↓
corresponding reaction becomes a microtask
   ↓
microtask executes
   ↓
returned Promise settles
   ↓
next .then() becomes a microtask
```

So:

```js
p
 .then(A)
 .then(B)
 .then(C)
```

is conceptually:

```text
p settles
   ↓
[A]
   ↓
A executes
   ↓
[B]
   ↓
B executes
   ↓
[C]
   ↓
C executes
```

Whereas:

```js
p.then(A);
p.then(B);
p.then(C);
```

can result in:

```text
[A, B, C]
```

being queued from the same settled Promise and then processed FIFO.

---

## Senior interview takeaway

If an interviewer asks:

> **"Are all `.then()` callbacks in a Promise chain placed into the microtask queue at once?"**

A strong answer is:

> **"No. The first reaction becomes a microtask when its Promise is fulfilled or rejected. When that callback executes and settles the Promise returned by that `.then()`, the next reaction becomes eligible and is queued as another microtask. So a Promise chain progresses through successive microtask scheduling."**

That's the level of detail I'd expect from a senior Node.js developer.

### Checkpoint

Predict the output:

```js
Promise.resolve()
  .then(() => {
    console.log("A");

    Promise.resolve().then(() => {
      console.log("B");
    });
  })
  .then(() => {
    console.log("C");
  });

Promise.resolve().then(() => {
  console.log("D");
});

console.log("E");
```

Give me the exact order.

---

**Rephrased:** You predicted the order as **`E → A → B → C → D`**.

You're **almost there, but `C` and `B` are reversed**.

The correct output is:

```text
E
A
D
B
C
```

### Let's trace the queue carefully

During synchronous execution:

```js
Promise.resolve()
  .then(() => { ...A... })
  .then(() => { ...C... });
```

The first `then` is queued:

```text
[A]
```

Then:

```js
Promise.resolve().then(() => {
  console.log("D");
});
```

adds another microtask:

```text
[A, D]
```

Finally:

```js
console.log("E");
```

runs synchronously:

```text
E
```

---

### Microtask #1 → `A`

Execute:

```text
A
```

Inside `A`:

```js
Promise.resolve().then(() => {
  console.log("B");
});
```

This queues `B`.

Also, when the first `.then()` finishes, the Promise returned by it fulfills, causing the chained `C` continuation to be queued.

At this point, the queue is conceptually:

```text
[D, B, C]
```

So the remaining execution is:

```text
D
B
C
```

Final:

```text
E → A → D → B → C
```

### The important lesson

You made the same type of mistake we discussed earlier: **`B` is created inside the `A` microtask, but `D` was already waiting in the microtask queue.**

FIFO matters.

```text
Before A:
[A, D]

A runs:
[D, B, C]

Then:
D → B → C
```

This is exactly the kind of queue reasoning I want you to become comfortable with for senior Node.js interviews.


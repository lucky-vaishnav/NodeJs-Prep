# `async/await` + `process.nextTick()`

You already know the simplified priority:

```text
Current synchronous JS
        ↓
process.nextTick()
        ↓
Promise microtasks
        ↓
normal Event Loop work
```

But now let's see what happens when they interact.

---

## 1. Basic example

```js
async function test() {
  console.log("A");

  await Promise.resolve();

  console.log("B");
}

console.log("C");

test();

process.nextTick(() => {
  console.log("D");
});

console.log("E");
```

Let's trace it.

### Synchronous execution

```text
C
A
E
```

When `test()` reaches:

```js
await Promise.resolve();
```

its continuation (`B`) is scheduled through the Promise microtask mechanism.

Then:

```js
process.nextTick(() => console.log("D"));
```

adds `D` to the nextTick queue.

So after synchronous execution:

```text
nextTick queue:
D

Promise microtask queue:
B
```

Node gives nextTick priority.

Therefore:

```text
D
B
```

Final output:

```text
C
A
E
D
B
```

---

# 2. Now reverse the registration order

```js
async function test() {
  console.log("A");

  await Promise.resolve();

  console.log("B");
}

console.log("C");

process.nextTick(() => {
  console.log("D");
});

test();

console.log("E");
```

Synchronous:

```text
C
A
E
```

At this point:

```text
nextTick:
D

Promise microtask:
B
```

Again:

```text
D
B
```

Final:

```text
C
A
E
D
B
```

The important thing is **not the order in which `nextTick` and `await` appeared in the source code**.

What matters is the queues that exist when Node reaches the microtask checkpoint.

---

# 3. Now a more interesting example

```js
async function test() {
  console.log("A");

  await Promise.resolve();

  console.log("B");

  process.nextTick(() => {
    console.log("C");
  });

  console.log("D");
}

test();

Promise.resolve().then(() => {
  console.log("E");
});
```

Let's carefully trace this.

Initially:

```text
Synchronous:
A
```

`await` schedules the continuation:

```text
Promise microtasks:
[resume test → B]
```

Then the independent Promise:

```js
Promise.resolve().then(() => console.log("E"));
```

adds:

```text
Promise microtasks:
[B, E]
```

Now execute `B`:

```text
B
```

Inside `B`:

```js
process.nextTick(() => console.log("C"));
```

registers `C`.

Then:

```js
console.log("D");
```

runs immediately within the same microtask:

```text
D
```

So you might expect:

```text
C
E
```

But there's an important nuance here.

A `process.nextTick()` scheduled **while Node is already draining Promise microtasks** doesn't necessarily interrupt the remaining Promise microtask queue immediately.

So conceptually:

```text
Initial Promise queue:
[B, E]

Execute B:
  schedule nextTick C
  print D

Promise queue still has:
[E]

Execute E:
  E

Then Node processes the nextTick queue:
C
```

So the result is:

```text
A
B
D
E
C
```

### Important senior-level nuance

This is why the simplistic statement:

> "`process.nextTick()` always executes before Promise microtasks"

is **too broad**.

A better statement is:

> **Node processes the `process.nextTick` queue with higher priority at its microtask checkpoints, but a `nextTick` scheduled while Promise microtasks are already being drained does not necessarily interrupt the remaining Promise microtasks immediately.**

This distinction is worth knowing for senior interviews.

---

# 4. Why `process.nextTick()` can be dangerous

Consider:

```js
function loop() {
  process.nextTick(loop);
}

loop();
```

Each nextTick schedules another nextTick.

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

Because Node prioritizes this queue, normal Event Loop work can be delayed significantly.

That means things like:

```text
Timers
I/O callbacks
setImmediate()
Incoming requests
```

can be starved.

This is why recursive `process.nextTick()` should generally be avoided unless there is a very specific reason.

---

# 5. Compare with `await`

This:

```js
async function loop() {
  await Promise.resolve();
  loop();
}

loop();
```

also creates an effectively endless sequence of Promise microtasks.

Conceptually:

```text
Promise microtask
      ↓
resume loop()
      ↓
await
      ↓
new Promise microtask
      ↓
resume loop()
      ↓
await
      ↓
...
```

This can also starve normal Event Loop work.

So both mechanisms can cause starvation:

```text
process.nextTick recursion
        ↓
nextTick starvation

Promise microtask recursion
        ↓
microtask starvation
```

---

## Senior mental model

Keep this:

```text
Current JS callback
       ↓
Node reaches microtask checkpoint
       ↓
process.nextTick queue
       ↓
Promise microtask queue
       ↓
normal Event Loop work
```

But remember the nuance:

> **Don't think of `nextTick` as a magical interrupt that jumps ahead of every currently-running or already-draining Promise callback.**

---

### Checkpoint

Predict the exact output:

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

  console.log("E");
}

console.log("F");

test();

Promise.resolve().then(() => {
  console.log("G");
});

console.log("H");
```

Give me the exact order and, more importantly, tell me **what the microtask queue looks like at each stage**.

---

**Rephrased:** You predicted:

`F → A → H → B → E → C → G → D`

You're **very close, but `C` and `G` are the key point**.

The correct output is:

```text
F
A
H
B
E
G
D
C
```

### Step-by-step

#### 1. Synchronous execution

```js
console.log("F"); // F
test();
```

Inside `test()`:

```js
console.log("A"); // A
await Promise.resolve();
```

`await` schedules the continuation of `test()`.

Then:

```js
Promise.resolve().then(() => console.log("G"));
```

adds another Promise microtask.

Finally:

```js
console.log("H"); // H
```

So after synchronous execution:

```text
F A H

Promise microtask queue:
[resume test → B, G]
```

---

### 2. First Promise microtask → resume `test()`

```text
B
```

Then inside `test()`:

```js
process.nextTick(() => console.log("C"));
```

adds `C` to the nextTick queue.

Then:

```js
Promise.resolve().then(() => console.log("D"));
```

adds `D` to the Promise microtask queue.

Then:

```js
console.log("E");
```

runs immediately as part of the current microtask:

```text
B
E
```

At this point conceptually:

```text
nextTick:
[C]

Promise microtasks:
[G, D]
```

---

### 3. Remaining Promise microtasks

Here's the important part.

The Promise microtask queue already has:

```text
[G, D]
```

So those continue:

```text
G
D
```

Only after the current Promise-microtask draining reaches the appropriate checkpoint does the `nextTick` callback execute:

```text
C
```

Therefore:

```text
F → A → H → B → E → G → D → C
```

### 🔑 What you should take away

You correctly identified the tricky part: `C` is a `process.nextTick()` callback created **inside a Promise microtask**.

But don't use this simplistic rule:

> "`nextTick` always jumps immediately before Promise microtasks."

Instead:

> **When Node reaches a microtask checkpoint, `process.nextTick` has higher priority, but a nextTick scheduled during an ongoing Promise-microtask drain does not necessarily interrupt Promise microtasks that are already queued.**

This is a subtle Node.js behavior, and you're now getting into genuinely **senior-level Event Loop questions**.

**Next, we'll combine `async/await` with `setTimeout()` / `setImmediate()` / I/O**, which will bring together almost everything you've learned.


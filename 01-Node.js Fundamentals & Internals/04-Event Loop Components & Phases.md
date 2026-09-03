We’ve finished the **basic Event Loop mental model**. Now let’s go one level deeper.

# 1.3.1 Event Loop Phases

This is where you'll start understanding **why `setTimeout()`, `setImmediate()`, I/O callbacks, etc. execute when they do**.

The first thing to know is that Node.js's Event Loop isn't just:

> "Check whether something is ready → execute it."

It has several **phases**, and each phase is responsible for a particular type of work.

---

## 1. The Event Loop at a high level

A simplified Node.js Event Loop looks like this:

```text
                 ┌───────────────┐
                 │     Timers    │
                 └───────┬───────┘
                         ↓
                 ┌───────────────┐
                 │Pending Callbacks│
                 └───────┬───────┘
                         ↓
                 ┌───────────────┐
                 │     Idle /    │
                 │   Prepare     │
                 └───────┬───────┘
                         ↓
                 ┌───────────────┐
                 │     Poll      │
                 └───────┬───────┘
                         ↓
                 ┌───────────────┐
                 │     Check     │
                 └───────┬───────┘
                         ↓
                 ┌───────────────┐
                 │ Close Callbacks│
                 └───────┬───────┘
                         │
                         └────────→ repeat
```

The phases are:

1. **Timers**
2. **Pending callbacks**
3. **Idle / prepare** — mainly internal to Node
4. **Poll**
5. **Check**
6. **Close callbacks**

For your interview preparation, the most important ones initially are:

> **Timers → Poll → Check → Close callbacks**

The others are useful to know, but we don't need to memorize their internal details.

---

# 2. Why does Node have phases?

Different asynchronous operations become ready at different times.

For example:

```javascript
setTimeout(...)
```

is related to the **Timers** phase.

Whereas:

```javascript
setImmediate(...)
```

is associated with the **Check** phase.

And many I/O callbacks are handled around the **Poll** phase.

So the Event Loop needs a structured mechanism to determine:

> **"What type of work is ready to be processed right now?"**

That's what the phases provide.

---

# 3. Phase 1 — Timers

The Timers phase handles callbacks associated with:

```javascript
setTimeout()
setInterval()
```

Example:

```javascript
setTimeout(() => {
    console.log("Timer");
}, 1000);
```

When you call:

```javascript
setTimeout(callback, 1000);
```

Node doesn't say:

> "Execute exactly one second later."

Instead, the timer establishes a **minimum delay**.

After approximately 1 second, the callback becomes eligible to execute.

Then the Event Loop can execute it during the appropriate timers processing.

### Important interview point

```javascript
setTimeout(callback, 0);
```

does **NOT** mean:

> "Execute immediately."

It means approximately:

> "Execute after the minimum delay has elapsed, when the Event Loop gets an opportunity."

So:

```javascript
console.log("A");

setTimeout(() => {
    console.log("B");
}, 0);

console.log("C");
```

produces:

```text
A
C
B
```

---

# 4. Phase 2 — Pending Callbacks

The **Pending Callbacks** phase handles certain callbacks that were deferred from the previous loop iteration.

You don't need to memorize a huge list of examples here.

The important conceptual understanding is:

> **Some system-level I/O callbacks that couldn't be handled immediately can be processed in this phase.**

This is much less important for everyday Node.js development than Timers, Poll, and Check.

So in your notes, just keep:

```text
Pending Callbacks:
- Executes certain deferred system/I/O callbacks.
- Less commonly discussed in application-level code.
```

---

# 5. Phase 3 — Idle / Prepare

This phase is primarily used internally by Node.js/libuv.

You generally won't write application code specifically targeting this phase.

For interview purposes:

> **Know that it exists; don't spend significant time memorizing its internals.**

---

# 6. Phase 4 — Poll ⭐

This is one of the **most important Event Loop phases**.

The Poll phase is responsible for:

* Retrieving new I/O events
* Processing many I/O-related callbacks
* Determining whether the Event Loop should wait for new I/O

For example:

```javascript
fs.readFile("data.txt", callback);
```

or network activity.

Conceptually:

```text
                Poll Phase
                    │
           ┌────────┴────────┐
           ↓                 ↓
       I/O ready         No I/O ready
           │                 │
           ↓                 ↓
      Execute I/O       Wait for I/O
       callbacks           events
```

The Poll phase is therefore central to Node's **non-blocking I/O model**.

---

# 7. Poll phase can wait

This is an important detail.

Suppose there is currently no immediate work to execute.

The Event Loop doesn't necessarily have to continuously spin at 100% CPU checking:

```text
"Anything ready?"
"Anything ready?"
"Anything ready?"
```

Instead, the Poll phase can **wait for I/O events** when appropriate.

Conceptually:

```text
Poll
 ↓
No immediate work
 ↓
Wait for I/O
 ↓
I/O becomes ready
 ↓
Process callback
```

This is one of the reasons the Event Loop can efficiently manage lots of I/O.

---

# 8. Phase 5 — Check ⭐

The Check phase is particularly important because this is where:

```javascript
setImmediate()
```

callbacks are executed.

Example:

```javascript
setImmediate(() => {
    console.log("Immediate");
});
```

Conceptually:

```text
Event Loop
    ↓
Poll
    ↓
Check
    ↓
setImmediate callback
```

So remember:

> **`setImmediate()` → Check phase**

This will become very important when we compare:

```javascript
setTimeout(..., 0)
```

vs

```javascript
setImmediate(...)
```

---

# 9. Phase 6 — Close Callbacks

This phase handles callbacks associated with closing certain resources.

For example:

```javascript
socket.on("close", () => {
    console.log("Socket closed");
});
```

Conceptually:

```text
Resource
   ↓
closed
   ↓
Close callback
   ↓
Close callbacks phase
```

Again, you don't need to spend much time on this phase yet.

---

# 10. The most important phases for you

For now, build this mental model:

```text
Timers
  ↓
Pending Callbacks
  ↓
Idle / Prepare
  ↓
Poll ⭐
  ↓
Check ⭐
  ↓
Close Callbacks
  ↓
Repeat
```

And associate:

```text
setTimeout()
setInterval()
       ↓
    Timers


I/O callbacks
       ↓
      Poll


setImmediate()
       ↓
     Check
```

This association is much more useful than simply memorizing the diagram.

---

# 11. A very important misconception

Don't think:

> "Every callback goes through every phase."

❌ That's not correct.

Instead:

> **Different types of callbacks are associated with different phases.**

For example:

```text
setTimeout()
     ↓
Timers


I/O callback
     ↓
Poll


setImmediate()
     ↓
Check
```

And later we'll add another layer:

```text
process.nextTick()
Promise callbacks
```

These don't fit neatly into the six-phase diagram because **microtask processing happens between/after callback execution points**, and `process.nextTick()` has its own special priority in Node.js.

That's why we'll study those separately.

---

# 12. A simplified execution example

Consider:

```javascript
console.log("A");

setTimeout(() => {
    console.log("B");
}, 0);

setImmediate(() => {
    console.log("C");
});

console.log("D");
```

We know immediately:

```text
A
D
```

will execute synchronously.

Then:

```text
setTimeout → Timers
setImmediate → Check
```

But here's an important point:

### The order between B and C is NOT universally guaranteed.

Depending on where this code is executed, the ordering can differ.

For example, when executed from the main module:

```text
A
D
B
C
```

is commonly observed.

But if the same pattern occurs inside an I/O callback, the relationship changes and `setImmediate()` can run before the timer.

**Don't memorize `setTimeout(0) always beats setImmediate()`.**

That's wrong.

We'll dedicate a section to this because it's a classic Node.js interview question.

---

# 13. Why `setImmediate()` exists

This is worth understanding conceptually.

Suppose you're inside an I/O callback:

```javascript
fs.readFile("data.txt", () => {

    setImmediate(() => {
        console.log("Immediate");
    });

});
```

After the I/O callback is processed during the Poll phase, the Event Loop can proceed to:

```text
Poll
 ↓
Check
 ↓
setImmediate()
```

So `setImmediate()` can be useful when you want something to execute **after the current I/O processing, during the Check phase**.

This is why it's particularly useful in certain I/O-related scenarios.

---

# 14. The Event Loop mental model

At this point, I want you to have this picture:

```text
                     Node.js
                        │
                       V8
                        │
                 Execute JavaScript
                        │
                   Call Stack
                        │
                        ↓
                  ┌───────────┐
                  │ Event Loop│
                  └─────┬─────┘
                        │
        ┌───────────────┼────────────────┐
        ↓               ↓                ↓
     Timers            Poll             Check
        │               │                │
  setTimeout       I/O callbacks    setImmediate
  setInterval
```

And:

```text
V8
 ↓
Executes JavaScript

Event Loop
 ↓
Coordinates when callbacks can execute

libuv
 ↓
Provides much of the underlying event-loop/I/O machinery
```

---

# 📝 Notes for your notebook

I'd write:

```text
## 1.3.1 Event Loop Phases

Node.js Event Loop consists of several phases:

1. Timers
2. Pending Callbacks
3. Idle / Prepare
4. Poll
5. Check
6. Close Callbacks

Important phases:

Timers
- Executes eligible callbacks for setTimeout/setInterval.
- Timer delay is a minimum threshold, not guaranteed execution time.
- setTimeout(fn, 0) does not mean immediate execution.

Poll
- Handles/retrieves many I/O-related events/callbacks.
- Can wait for new I/O when appropriate.
- Important part of Node.js non-blocking I/O model.

Check
- Executes setImmediate() callbacks.

Close Callbacks
- Handles callbacks related to closing resources.

Important associations:

setTimeout/setInterval → Timers
I/O callbacks          → Poll
setImmediate           → Check

Important:
- Not every callback goes through every phase.
- Event Loop coordinates callbacks; V8 executes JavaScript.
- setTimeout(0) vs setImmediate() ordering depends on context.
```

---

# 🎯 Interview checkpoint

Before we move to **`setTimeout()` vs `setImmediate()`**, answer these:

### Q1

**What are the major phases of the Node.js Event Loop?**

Don't just list them — tell me what **Timers, Poll, and Check** are responsible for.

### Q2

What will this print?

```javascript
console.log("A");

setTimeout(() => {
    console.log("B");
}, 0);

setImmediate(() => {
    console.log("C");
});

console.log("D");
```

And most importantly:

**Can you guarantee whether B or C comes first? Why or why not?**

### Q3 — Senior

If the Event Loop is currently in the **Poll phase** and an I/O operation has completed, what broadly happens to its callback?

Take your time and answer in your own words. We’ll use your answers to make sure the Event Loop model is solid before adding **microtasks and `process.nextTick()`**, which is where many Node.js interviews get tricky.

### My Thoughts-
So for question number one, what are the major phases? So yeah, major phases are three: one, timer, poll, and check phases. Total number of phases are six, if we really mention each of them. So timer, and then check for the previous callbacks, and then they will be prepared, or the idle phase, and then polling, then check, and close callbacks, just for the closing of the resources, like socket.close. So timer phase have this setTimeout and setInterval-related operations. Polling phase have this input-output operations, and it will also wait if there is no input-output operation is ready. And check phase is for the setImmediate. And for question two, so A and D will be printed, and between setTimeout and setImmediate, B and C. So generally it will be A, D, B, C, but if the setImmediate is part of a input-output callbacks, then setImmediate will be executed first. So yeah, we cannot, we will not be sure that what will be sequence of printing B and C. So we will not guaranteed. And so once the input-output operation has completed, for the callbacks of this will be executed when the new event loop phase is started. But if we want to really execute it after input-output operation has been completed, we can use setImmediate. So these are my answer. And after reviewing my this answer, there are two questions from my side. So please rephrase this question and just answer it in shorter format. So first is that the code for question two which you shared, if we really talk about this code, then the sequence will be, sure that the sequence will be A, D, B, C, because setImmediate we have written it manually, and we can see that this is not part of a input-output operation. So for this code, it will be always with this sequence: A, D, B, C. And my second question is that for check phase. So for check phase, there is only one setImmediate is tracked for this check phase, or there are more operations or type of operations which are tracked or handled in this check phase?


### Questions

1. **For the code in Question 2, is `A → D → B → C` guaranteed because the `setImmediate()` is not inside an I/O callback?**

2. **In the Check phase, does Node.js handle only `setImmediate()` callbacks, or are there other types of operations handled there as well?**

### 1. Is `A → D → B → C` guaranteed?

**No — this is the important correction.**

For this code:

```js
console.log("A");

setTimeout(() => console.log("B"), 0);

setImmediate(() => console.log("C"));

console.log("D");
```

We can guarantee:

```text
A
D
```

But **we cannot guarantee whether `B` or `C` comes first**.

Possible output:

```text
A
D
B
C
```

or:

```text
A
D
C
B
```

Why? Because the ordering between `setTimeout(0)` and `setImmediate()` **when scheduled from the main module** depends on timing/runtime conditions.

The important interview statement is:

> **`setTimeout(0)` vs `setImmediate()` ordering is not guaranteed when scheduled from the main module. However, when both are scheduled inside an I/O callback, `setImmediate()` executes before the timer.**

So your second answer was mostly correct, but don't say "`setImmediate` is always after I/O"; rather, **inside an I/O callback, it is commonly used to run something immediately after the poll phase, before timers in the next iteration.**

---

### 2. Does Check phase handle only `setImmediate()`?

For interview purposes, **yes, think of the Check phase as the phase where `setImmediate()` callbacks are executed.**

```text
Poll
  ↓
Check
  ↓
setImmediate() callbacks
```

`setImmediate()` is the **main/public API associated with the Check phase**.

There are internal Node.js mechanisms involved around the phase, but you don't need to memorize a list of other user-facing operations.

A good senior-level answer:

> **The Check phase is primarily responsible for executing callbacks scheduled with `setImmediate()`.**

One small correction to your original answer: an I/O callback doesn't necessarily mean its callback is simply "executed when the new Event Loop phase starts." The callback is processed according to the appropriate phase/queue, and **Poll is the key phase for many I/O callbacks**.

You're doing well here. Your understanding of **Timers → Poll → Check** is solid enough to move deeper into the `setTimeout()` vs `setImmediate()` behavior next.

---

### Questions

1. **When the Event Loop is in the Poll phase and there is no I/O work ready, how long does it wait before moving on?**

2. **When an I/O operation completes, in which Event Loop phase is its callback actually executed — the Pending Callbacks phase or the Poll phase?**

### 1. How long does Poll wait?

There is **no fixed waiting time** like 10 ms or 100 ms.

The Poll phase decides dynamically whether it should wait for I/O.

Broadly:

```text
Poll phase
   ↓
Are there I/O callbacks ready?
   ├── Yes → process them
   │
   └── No
       ↓
   Is there something scheduled for the next phase?
       ├── Yes → don't wait; move toward Check/next iteration
       └── No → may wait for new I/O
```

So the key point for interviews is:

> **The Poll phase can wait for I/O, but the waiting duration is dynamically determined by Node.js/libuv based on pending work and timers/other scheduled operations. There is no single fixed wait time.**

For example, if you have:

```js
setTimeout(() => {
  console.log("timer");
}, 1000);
```

and Poll has nothing else to do, it doesn't make sense to wait indefinitely for I/O. The existence of the timer gives the event loop a reason to eventually wake up and process the timer.

---

### 2. Where does an I/O callback execute?

This is the more important distinction.

**Many I/O callbacks are processed in the Poll phase itself.**

For example, conceptually:

```text
I/O operation completes
        ↓
callback becomes ready
        ↓
Poll phase
        ↓
I/O callback executes
```

So if you're thinking:

> "I/O completed → Pending Callbacks phase must execute it"

**Don't use that as the general rule.**

The **Pending Callbacks phase** is for a more specific category of deferred system-level/I/O callbacks. It is **not the general phase for every completed I/O operation**.

A good interview-level mental model is:

```text
Timers
   ↓
Pending Callbacks     ← certain deferred system callbacks
   ↓
Idle / Prepare
   ↓
Poll                  ← many I/O callbacks are handled here
   ↓
Check                 ← setImmediate()
   ↓
Close Callbacks
```

### One important nuance

When we say:

> "I/O operation completed"

the actual behavior depends on **what kind of I/O operation and which Node.js API is involved**. Node/libuv may use the OS directly or its thread pool, and the resulting callback is scheduled according to the relevant mechanism.

So don't try to memorize:

> "Every I/O callback = Poll."

Instead, say:

> **"The Poll phase is responsible for retrieving and processing many I/O-related callbacks, while the Pending Callbacks phase handles certain callbacks that were deferred to a later iteration."**

That's the safer **senior-level interview answer**.

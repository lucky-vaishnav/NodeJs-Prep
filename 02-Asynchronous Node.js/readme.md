We completed Phase 1 — Node.js Fundamentals & Internals. Next, we continue with the original roadmap, Phase 2, without repeating topics we've already learned.

## Phase 2 — Asynchronous Node.js

Here's our current plan:

| Topic                           | Status                                                   |
| ------------------------------- | -------------------------------------------------------- |
| 2.1 Callbacks                   | Partially covered — fill only the gaps                   |
| 2.2 Promises                    | Covered — skip basic concepts                            |
| 2.3 `async/await`               | Covered — skip basic concepts                            |
| 2.4 Promise scheduling          | Covered in depth                                         |
| 2.5 Error handling              | Partially covered — focus on missing production patterns |
| 2.6 Concurrency vs. parallelism | Needs systematic coverage                                |

### Next topic: 2.1 Callbacks — Missing Senior-Level Concepts

We'll focus on what's still useful:

* Error-first callbacks and their conventions.

* Callback hell and control-flow problems.

* Callback execution more than once and how to prevent it.

* Error propagation through nested callbacks.

* When callbacks are still appropriate in production Node.js.

We'll skip callback scheduling and Event Loop fundamentals because we've already covered them.

Let's continue with the missing callback concepts first.

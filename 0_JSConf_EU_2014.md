# Preview

[YT talk](https://www.youtube.com/watch?v=8aGhZQkoFbQ)

JS is a *single-threaded*, *non-blocking*, *asynchronous*, *concurrent* language.

For JS to work correctly it needs to run inside an *host environment* (i.e. browser, node, ...).

This host environment has to provide JS with:
- a runtime (i.e. Chrome V8 Engine) which contains an *heap* (where memory allocation happens) and a *call stack*
- a *callback* or (*task*) *queue*
- an *event loop*
- (in the browser) *Web APIs* like the *DOM*, *ajax*, etc...
and uses them to correctly run our code.

JS is a single-threaded language, so its runtime Engine has a single call-stack and basically can only do one thing at a time.

The call-stack is a data-structure (last-in first-out) which *records where in the program we are*.
When the Engine runs a function, it puts a new block on top of the stack, and when a function returns the Engine pops the corresponding block off the stack.

What is a blocking behavior? Blocking code is basically code that is slow.
Since JS is single-threaded the Engine can only execute one function per time. If that function takes a long time to complete, the rest of our program is stuck waiting.

A way to handle blocking behavior is by using asynchronous callbacks.
Asynchronous is code that can be separated into 2 parts.
One part is run in the "now" moment, as soon as the piece of code is met by the Engine (usually sets up a listener for some sort of event), but the other part, the asynchronous callback, is deferred for a (usually unspecified) "later" time.
The big thing is that while the browser waits for whatever event will trigger the call of the callback, it keeps running other code.

But how do asynchronous callbacks and the call-stack work together?

When an async chunk of code runs the Engine puts the corresponding block on top of the call stack.
The sync part of the code (set up the listener) executes, then somehow the block is removed from the stack so that other blocks can be executed, and when the event fires the async part (callback) is called and gets its own block in the stack.

To understand how this is possible, we need to introduce the event loop.

The JS runtime (V8 Engine, ...) is single-threaded and by itself can only do one thing per time.
But we saw that the browser provides more than just the runtime.
By using Web APIs, a callback queue and an event loop we are able to run processes concurrently inside our program.

Let's imagine a simple program:
```js
console.log('Hello');

setTimeout(() => console.log('World'), 2000);

console.log('Log');
```
The program starts running, which is sort of like putting a `main` function call at the bottom of the call-stack, and the Engine executes the code statement by statement.

The first statement met is `console.log('Hello');`.
The Engine pushes the `.log` method on top of the stack, executes it and removes it from the stack.

Then it goes to the second statement, which is the `setTimeout` call.
`setTimeout` is a Web API. It takes in a callback and a timer and runs the callback when the timer is done.
Its code can be divided into 2 parts.
The first part sets up the timer.
The second part is the callback.

When the Engine pushes the `setTimeout` call in the call-stack, it only runs the first part of the code. So it starts the timer and considers the `setTimeout` call done. Thus it removes its block from the stack and moves to the next statement.

The last statement is again a `console.log`, which is executed right away.

But we are not done yet.
At some point the timer will run out and the Engine will have to run the callback.
But the Engine can't just execute the callback, because there might be other things going on inside the call-stack at that moment.
Instead the Engine pushes the callback inside the task queue (first-in first-out).

Between the task queue and the call-stack sits the event loop.
The event loop only has one job, it watches the call-stack and as soon as it is empty, the event loop takes the first task from the task queue and pushes that into the call-stack for execution.

So when the timer runs out, the browser pushes the callback in the task queue, the event loop sees the call-stack empty, thus takes the callback and pushes it in the stack.

The final output of our program is:
```
Hello
Log
World
```

Like we said, the event loop will push a task from the task queue into the call-stack only when the call-stack is empty.
We can leverage this and use `setTimeout(cb, 0);` to defer the call of the callback for "later" but still "as soon as possible".

**NB:** our example program was by using `setTimeout`, but the same process happens for other Web APIs.

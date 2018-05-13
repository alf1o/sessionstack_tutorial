# Event Loop and Async Programming

[original article](https://blog.sessionstack.com/how-javascript-works-event-loop-and-the-rise-of-async-programming-5-ways-to-better-coding-with-2f077c4438b5)

JavaScript is a single-threaded language, this means that its Engine can only have one call-stack and basically execute only one chunk of code at a time.

This is a limitation, because in case of a very expensive operation our whole application will be stuck and unresponsive during its execution.
If the browser starts processing too many tasks in the call-stack, the app could be unresponsive for a long time. In this case many browsers will raise an error asking the user to kill the page.

## asynchrony

The way we handle blocking code is with asynchrony.

Given a program its code can be divided in 2 parts:
- some code to be executed "now", as soon as it's met
- some code that for reasons cannot be executed "now" and thus is deferred for "later" (usually an unspecified moment in time)

The gap between the "now" moment in which the code is met and the "later" moment in which the code is executed is what makes up asynchrony.

The best thing about asynchrony is that it is non-blocking.
While the Engine waits for that "later" moment to arrive, it can keep executing other code.

This can actually be confusing:
```js
// imagine `ajax` is a utility to perform network requests
var data = ajax(url);
console.log(data); // --> undefined
```
While the browser fires the request and waits for the response the Engine keeps executing code, thus at the `console.log(data);` statement the `data` variable isn't yet filled with any data.

The typical way with which asynchrony is handled in JS is by using callbacks.
A callback is a function that *wraps the continuation of our program*, so that when the async action terminates such callback is executed moving us to the next step.

## Event loop

Up until ES6 the JS Engine didn't have any notion of time.

The Engine itself doesn't decide when to execute code, it simply runs through the program and executes it statement by statement.

So how is it possible to have async code inside a JS program?

To run JS code we need an host environment (i.e. browser, node, ...).
The host environment provides the Engine to execute the code, but it also provides a task (or callback) queue, an event loop and a bunch of APIs (i.e. Web APIs (DOM, XHR, setTimeout, ...) in the browser, C++ APIs in node).

By using the task queue and the event loop the browser gives the Engine code to execute over time. Thanks to this we can have async code (and concurrent processes) in a JS program.

The Engine sort of works as an on-demand execution environment for JS code.
The task queue is what schedules the code to run, and the event loop is what passes this code to the Engine at the right time.

How does it all work together?

We talked about "now" and "later" code.
Each async chunk of code can be divided into a "now" bit and a "later" bit.
Usually, the "now" bit sets up an event listener, while the "later" bit is the actual callback.

So the browser is made listen for a certain event.
When (and if) this event fires the browser passes the callback to the Engine.
But it can't just push the callback inside the call-stack because there might be other code being executed at that moment.

Instead the browser pushes the callback inside the task queue.

Between the task queue and the call-stack there is the event loop.
The event loop constantly watches the call-stack and as soon as it is empty, the event loop retrieves the first task from the task queue and pushes it inside the call-stack for execution.

Each iteration of the event loop is called a tick.
A task in the task queue is simply a callback.

ES6 specifies how the event loop should work, meaning that technically the Engine is no longer just an execution environment. One main reason for this change is the introduction of Promises in ES6 that require access to a direct, fine-grained control over scheduling operations on the event loop queue.

## Web APIs

To implement async in our program we use these Web APIs, which like we said allow us to set up a listener for some sort of event.

*Web APIs are essentially threads that we can't directly access but we can only make calls to.*
They are pieces of the browser in which concurrency kicks in.

## Job queue

Like we said ES6 specifies how the event loop should work.
Also it introduces the job queue.

The job queue is a queue within each tick of the event loop.

It will be used by Promises.
Promises handle every bit of code async, and that asynchrony is relative to the job queue.

In general some async actions won't cause a whole new event to be pushed at the end of the task queue, but rather at the end of the job queue for the current tick.

TODO: callback downsides, promises, async await

## async await

Recently JS introduced async functions.

The main purpose of async functions is to write sequential, sync-looking code and hide the async as an implementation detail.

Async functions can be seen as syntactic sugar for the generators + promises + run utility pattern.

We use the `async` keyword to tell JS that the following function is an async function.

When an async function is executed it returns a promise.
The returned value of the function will be the fulfilment value of the promise.
If at some point the function throws an error, the promise returned will be rejected with that error.

An async function can contain `await` expressions.

The `await` keyword allows us to pause the async function execution and execute other code, just like the `yield` keyword in a generator.
The main difference is that async functions assume the expression they `await` for is a promise and they automatically resume execution when the promise resolves passing the resolution value to the `async` keyword.

**NB:** the `await` keyword can only be used inside an `async` function. Elsewhere it will be a `SyntaxError`.

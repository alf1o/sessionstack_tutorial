# Environment

[original article](https://blog.sessionstack.com/how-does-javascript-actually-work-part-1-b0bacc073cf)

The host environment in which JS runs provides an Engine, a task (or callback) queue, an event loop and a bunch of APIs.

The JS code is executed by the Engine.

A JavaScript Engine consists of 2 main components:
- heap, where the memory allocation happens
- call-stack, where the stack frames are as the code gets executed

JavaScript is a single-threaded programming language, which means its Engine only has one call-stack and therefore can only do one thing at a time.

The call-stack is a data-structure which basically records where in our program we are.
As the Engine runs through the program it pushes a statement at the top of the stack, executes it and pops it off the stack. Then goes to the next statement.

Each entry in the call-stack is called a *stack frame*.

"Blowing the stack" or "stack overflow" is when we exceed the maximum capacity of the stack.

Running on a single-thread makes things easier than on multi-thread, but it also limits the power of our program.
Since the Engine can only execute one chunk of code per time, what happens when we have slow code?
The whole application gets stuck.

The way we handle blocking code is by using asynchrony.

# Inside the Engine

The JavaScript Engine is a program that executes JavaScript code.

It can be implemented either as a *standard interpreter* or as a *just-in-time compiler* (JIT) which compiles JS code into bytecode.

Some of the most famous JS Engines are V8 (C++, Chrome/Node), Rhino (Java), SpiderMonkey (Firefox).

All 3 of them use a JIT compiler which compiles JS code during execution.

The main difference is that the V8 Engine compiles JS code directly into machine code rather than having an intermediate bytecode step.

The V8 Engine uses several threads internally:
- the main thread gets the JS code, compiles it and runs it.
- there is also a second thread for compiling, so that while the code is running the V8 can keep optimizing it.
- another thread, called Profiler thread, watches the code for "hot" chunks and notifies the runtime to optimize that chunk.
- a bunch of other threads handle garbage collection.

When first executing the code, the V8 compiles JS code into machine, without actually optimizing it. This allows for a fast start in the execution process.

As the code runs, the Profiler thread gathers data, and based on this data the second compiler thread optimizes the code.

## inlining

One of the basic optimizations is to inline as much code as possible.
Inlining is the process of replacing a call-site with the body of the called function.

TODO

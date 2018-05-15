# Inside the Engine

[original article](https://blog.sessionstack.com/how-javascript-works-inside-the-v8-engine-5-tips-on-how-to-write-optimized-code-ac089e62b12e)

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

When first executing the code, the V8 compiles JS code into machine code, without actually optimizing it. This allows for a fast start in the execution process.

As the code runs, the Profiler thread gathers data, and based on this data the second compiler thread optimizes the code.

# The JIT compiler

[original article](https://hacks.mozilla.org/2017/02/a-crash-course-in-just-in-time-jit-compilers/)

The JS Engine can be implemented either as a standard interpreter or as a Just-In-Time compiler.

JavaScript is an high-level language. It is closer to the human language than it is to the machine language.

In general, when some code runs it must be somehow turned into machine code, a set of instructions a computer can understand and execute.

There are 2 ways this process can happen:
- by using an interpreter
- by using a compiler

## interpreter

An interpreter executes the code line by line.

At run-time the single line is turned into machine language and executed by the computer.
When the execution of a line finishes, the interpreter moves to the next.

Interpreters are good to get started right away with code execution. There's no need for a preparation step.
As soon as a file loads, the Engine starts running it.

This is typically what we want in browsers, that's why initially JS Engines were implemented as interpreters.

The downside of interpreters is that code isn't optimized.
The Engine simply "translates" it into machine code, without changes.

## compiler

A compiler executes code in 2 steps.

The first step is the compilation step.
The code is transformed from an high-level language to a low-level language (bytecode).

The second step is the actual execution of the bytecode.

The compilation step happens ahead of time, and it's usually distinct from the execution step. Thus the compiler can actually take its time and optimize the code.

The compiled version can then be transported into other machines and executed there.

A compiler can't be used in the browser because there's not enough time to perform the compilation step.

## JIT

A JIT compiler is able to compile code at run-time, as it's being executed.
Thus we have both the speedy starting of an interpreter and the optimizations of a compiler.

The way this is implemented is by adding a new thread inside the Engine, called Profiler, which has the task of watching the code for "hot" pieces.
The Profiler keeps track of how many time a piece of code is run and what types are used.

When the same code is executed more than once, we say it is "warm".
When the same code is executed many times, we say it is "hot".

Initially everything is run through an interpreter.

During execution the Profiler thread reports "warm" and "hot" code to the compiler thread which optimizes it.

The compiled version of a piece of code is stored keeping track both of the line number and the variable type.

If a line has to be executed and a compiled version that uses the same variable types exists then that version is run instead.

Lines of code that are "warm" are sent to the baseline compiler.
It will make some optimizations, but we still want the process to be fast to not hold up execution for too long.

When the code becomes "hot" the compiler decides it is worth taking extra time to optimize it, thus it is sent to the optimizing compiler.

In order to make a faster version of the code, the optimizing compiler has to make some assumptions.

For example, if the compiler can assume that all the objects created by a particular construct have the same shape (same property names added in the same order) it can optimize the creation of said objects.

The optimizing compiler uses the information gathered by the Profiler. It assumes that if something has been true for all the previous executions of a line of code, then it will continue to be true.

Of course at some point an assumption might not be true anymore, so the compiler has to perform some checks before actually running the compiled version of a line of code.
When this happens the baseline-compiled version of the code (or maybe even its initial interpreted version) runs rather than the optimizing-compiled version.

This process is called de-optimization or bail-out.

**Note:** if code gets constantly optimized and de-optimized it might end up being slower than its interpreted version. Browsers have limits to avoid this scenario.

## type specialization

There are many optimizations a compiler can perform.

One of them is type specialization.

JavaScript is a dynamically typed language. This means we don't need to specify the type of a value before using it. Also the same variable can hold values of different types at different points in the program.

This makes optimizing code harder, because compilers cannot assume the same variable will always hold the same type.

```js
function arraySum(arr) {
  var sum = 0;
  for (var i = 0; i < arr.length; i++) {
    sum += arr[i];
  }
}
```
In this code the line `sum += arr[i];` is "hot" because lives inside a `for` loop, but because of dynamic typing it can't be easily optimized.

The compiler can't be sure that `sum` and `arr[i]` will always be numbers.
So the way it handles this is by creating more than one compiled version of the same line of code.
Depending on which type is used, that version is used to run the line.

Monomorphic code (always called with same types) will only get one compiled version.
Polymorphic code (called with different types) will get a version for each type combination.

This means that to decide which version to run, the compiler needs to answer to a bunch of questions.

For example, in our code, in the case of the line `sum += arr[i];` the compiler has to check:
- is `sum` an integer?
- is `arr` an array?
- is `i` an integer?
- is `arr[i]` an integer?

This "type checking" has to happen each time the line `sum += arr[i];` has to be executed.

Code execution would be faster if the compiler didn't have to do the checking for each iteration of the loop.

So, inside the optimizing compiler, the whole function is compiled together and almost all the type checks are moved to happen before the loop.
The only type check the compiler has to do for each iteration is the one for `arr[i]`.

Some compilers optimize this even further. For example they might label at some point `arr` as an array of integers, in which case there would be no need to check `arr[i]` for its type.

## conclusion

A JavaScript engine implemented with a JIT compiler makes JS code run faster by monitoring it as it's running and optimizing "hot" code based on the information gathered.

Even though JIT compilers are great they also come with the trade-off of some overhead:
- optimization and de-optimization
- memory used for monitoring code and recover information when bail-outs happen
- memory used to store baseline and optimized versions of the code

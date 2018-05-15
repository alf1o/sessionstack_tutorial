# Memory Management

[original article](https://blog.sessionstack.com/how-javascript-works-memory-management-how-to-handle-4-common-memory-leaks-3f28b94cfbec)

Some programming languages provide methods for the developer to directly handle memory allocations.
For example the C language has the `malloc` and the `free` functions.

At least for now, JavaScript is not such language.

JS allocates memory when values need to be stored and automatically frees it when they are not used anymore.
This process is called *garbage collection* and is usually handled by the Engine itself.

This automatic process of allocating/freeing memory gives the false impression that we can forget about memory management.
That's not the case.

Automatic memory management can have limitations, or even bugs, and we need to understand them in order to handle them.

## memory lifecycle

The memory lifecycle is pretty straightforward:
1. Allocate memory: memory is allocated by the OS which allows the program to use it. In high-level languages like JS this is handled automatically
2. Use memory: the program actually makes use of the memory with a series of read/write operations on the variables in our code
3. Release memory: the memory that is not needed is freed so that it can be reused again. Just like before this step happens without us calling for it

## what is memory

At the hardware level memory consists of a large number of flip-flops (can store 1 of 2 states).
Each flip-flop can hold 1 bit and is addressable by a unique identifier, so that we can write to them (change their state).

We can imagine memory as being this huge array of bits that we can read and write to.

Bits are organized together into larger groups that we can use to represent numbers and characters.

Usually numbers are stored as either 4 bits or 8 bits (1 byte).
While characters are either 16 bits or 32 bits.

**NB:** characters are first represented as numbers by an encoding like UTF-8 and then they can be stored.

In the OS memory is stored all data used by all our programs and the programs code, including the OS code.

When we write a program the compiler can calculate at compile time the memory that the program will need based on the primitive data-types inside it.
The required amount is then allocated in the *stack space*.

**Note:** it is called stack space because, as functions get called, their memory is allocated on top of existing memory. As they terminate they are removed in a last-in first-out order.

For example:
```c
int n; // 4 bytes
int x[4]; // array of 4 elements, each 4 bytes
double m; // 8 bytes
```
For this simple program the compiler knows will need `4 + (4 * 4) + 8 = 28` bytes, thus it can request such amount of memory from the OS before the program is executed.

Notice that the compiler also knows the exact memory address of each variable.

## dynamic allocation

But it is not always possible to know at compile time how much memory the program is going to need.
For example, if we wait for a certain user interaction, or maybe for a certain async action to be performed, and then build something like an array based on that data we receive, the compiler cannot know at compile time the size this array will have, thus cannot pre-allocate memory in the stack space from the OS.

Our program needs to ask the OS for the right amount of memory at run-time. Rather than from the stack space, this memory is assigned from the *heap space*.

**NB:** to fully understand how dynamic memory allocation works we need to know about *pointers*.

Comparing static and dynamic memory allocation:
- **Static**
  - size must be know at compile time
  - allocation happens at compile time
  - is assigned to the stack space
  - is freed in a last-in first-out order
- **Dynamic**
  - size might be unknown at compile time
  - is performed at run-time
  - is assigned to the heap
  - there is no particular order of assignment

## allocation in JS

Like we said, JS is an high-level language that handles memory allocation by itself as we assign values to variables.

i.e.:
```js
var n = 374; // allocates memory for a number
var s = 'sessionstack'; // allocates memory for a string
var o = {
  a: 1,
  b: null
}; // allocates memory for an object and its contained values
var a = [1, null, 'str'];  // (like object) allocates memory for the
                           // array and its contained values
function f(a) {
  return a + 3;
} // allocates a function (which is a callable object)
// function expressions also allocate an object
someElement.addEventListener('click', function() {
  someElement.style.backgroundColor = 'blue';
}, false);
```

Some function calls can also result in memory allocation:
```js
var d = new Date(); // allocates a Date object
var e = document.createElement('div'); // allocates memory for a DOM element
```

Finally, methods can also allocate new values or objects:
```js
var s1 = 'sessionstack';
var s2 = s1.substr(0, 3); // s2 is a new string
// Since strings are immutable,
// JavaScript may decide to not allocate memory,
// but just store the [0, 3] range.
var a1 = ['str1', 'str2'];
var a2 = ['str3', 'str4'];
var a3 = a1.concat(a2);
// new array with 4 elements being
// the concatenation of a1 and a2 elements
```

## memory usage in JS

Using memory just means reading and writing it.

Every time we somehow interact with a variable, or object properties or even passing arguments to a function, we are using memory.

## releasing memory

Most of the memory management issues come with releasing memory.

The task to decide whether some allocated memory is not needed any longer is not easy to be performed programmatically and usually requires the programmer to handle it.

High-level languages though, implement a piece of software, called *garbage collector*, which has the job of tracking memory allocation in order to find out when a piece of memory is not needed any longer and thus free it.

Unfortunately, this process is *undecidable*, meaning it can't be solved by an algorithm. Thus, garbage collectors have to do with approximations.

## garbage collection

The main concept garbage collection algorithms rely on is *reference*.

In the context of memory management, an object `a` is said to reference another object `b`, if `a` has an access to `b` (either implicit or explicit).

For example, a JS object has an implicit reference to its `[[Prototype]]` linked object and an explicit reference to its properties values.

In this sense, the concept of object is extended to also scopes.

### reference counting

The simplest garbage collector algorithm is called reference counting:

*an object is considered garbage-collectable if there are 0 references to it.*

```js
// 2 objects are created.
// 'o2' is referenced by 'o1' object as one of its properties.
// None can be garbage-collected
var o1 = {
  o2: {
    x: 1
  }
};

var o3 = o1; // the 'o3' variable is the second thing that
            // has a reference to the object pointed by 'o1'.

o1 = 1;      // now, the object that was originally in 'o1' has a         
            // single reference, embodied by the 'o3' variable

var o4 = o3.o2; // reference to 'o2' property of the object.
                // This object has now 2 references: one as
                // a property.
                // The other as the 'o4' variable

o3 = '374'; // The object that was originally in 'o1' has now zero
            // references to it.
            // It can be garbage-collected.
            // However, what was its 'o2' property is still
            // referenced by the 'o4' variable, so it cannot be
            // freed.

o4 = null; // what was the 'o2' property of the object originally in
           // 'o1' has zero references to it.
           // It can be garbage collected.
```

This algorithm seems to work fine, but what happens when we have cycles?

A *cycle* is when two objects reference one another:
```js
function f() {
  var o1 = {};
  var o2 = {};
  o1.p = o2; // o1 references o2
  o2.p = o1; // o2 references o1. This creates a cycle.
}

f();
```
In this code both `o1` and `o2` go out of scope after `f` is done running, so they are effectively useless and could be freed.
The problem is that they both reference one another, thus the reference counting algorithm won't release them.

### mark and sweep

A better algorithm is the mark and sweep one.
In order to decide whether or not an object is garbage collectable *the algorithm determines if such object is reachable*.

The algorithm goes through 3 steps:
1. identify *roots*: in general roots are global variables which get referenced in the code (i.e. `window`, `global`)
2. mark objects: starting from the roots the algorithm marks them and their children as active (not-collectable). Anything that cannot be reached is marked as garbage
3. free: finally the garbage collector frees all memory pieces that are marked as garbage and returns the memory to the OS

This algorithm is better than the reference counting one, because if an object doesn't have any reference to it is of course not reachable, and thus marked as garbage. At the same time though, if we have a cycle, the algorithm can still identify objects that can be collected.

All modern browser use some variation of the mark and sweep algorithm.

### trade-offs

Even though garbage collectors are useful they come with a bunch of problems.
One of them is non-determinism.
Garbage collectors are unpredictable and we can't really tell when a "sweep" will be performed.

This means that in some cases the program will use more memory space than it actually needs, and with particular sensitive application, we might experience short pauses due to the garbage collector running in the background.

That said, many garbage collectors share the common pattern of performing collection passes during memory allocations. This means that if no allocations are performed, garbage collectors (usually) will stay idle.

For example:
1. A sizable set of allocations is performed.
2. Most of these elements (or all of them) are marked as unreachable (suppose we null a reference pointing to a cache we no longer need).
3. No further allocations are performed.
In this scenario, most GCs will not run any further collection passes. Even though there are unreachable references available for collection, these are not claimed by the collector.
These are not strictly leaks but still, result in higher-than-usual memory usage.

## memory leaks

Memory leaks are pieces of memory that the application needed in the past but not anymore, but they haven't been returned to the OS.

### global variables

Global variables are always added as properties on the `window` object, thus they will be always reachable and never collected by the garbage collector algorithm.

If we use global variables to temporarily store and process large bits of data these will be around for the rest of our program.
The best thing to do is to manually reassign `null` to such global variables once we are done with them.

### timers

When timers, like `setInterval` reference some external data, the garbage collector won't be able to release that data from memory for as long as the timer keeps going.

For example:
```js
var serverData = loadData();
setInterval(function() {
  var renderer = document.getElementById('renderer');
  if(renderer) {
    renderer.innerHTML = JSON.stringify(serverData);
  }
}, 5000); //This will be executed every ~5 seconds
```
If at some point the `renderer` object is replaced or removed the code inside `setInterval` will become useless, but will also keep running until we clear it.
As long as `setInterval` runs its dependencies won't be collected.
This means that the `serverData` variable which probably stores some big data, won't be collected either.

The same is true for event listeners and callbacks:
```js
var element = document.getElementById('launch-button');
var counter = 0;
function onClick(event) {
  counter++;
  element.innerHtml = 'text ' + counter;
}
element.addEventListener('click', onClick);
```
In this code, as long as the listener is set up, the garbage collector won't be able to release neither the `element` nor the `counter` variable nor the `onClick` function object.

The best practice is to always clear our observers once we don't need them:
```js
element.removeEventListener('click', onClick);
element.parentNode.removeChild(element);
// Now when element goes out of scope,
// both `element` and `onClick` will be collected even in old browsers
// that don't handle cycles well
```

### closures

A closure is a function that has access to the variable in the scope within which it was declared even when it is called from outside of that scope.

Closures can be cause of memory leaks:
```js
var theThing = null;
var replaceThing = function () {
  var originalThing = theThing;
  // Define a closure that references originalThing but doesn't ever
  // actually get called. But because this closure exists,
  // originalThing will be in the lexical environment for all
  // closures defined in replaceThing, instead of being optimized
  // out of it. If you remove this function, there is no leak.
  var unused = function () {
    if (originalThing)
      console.log("hi");
  };
  theThing = {
    longStr: new Array(1000000).join('*'),
    // While originalThing is theoretically accessible by this
    // function, it obviously doesn't use it. But because
    // originalThing is part of the lexical environment, someMethod
    // will hold a reference to originalThing, and so even though we
    // are replacing theThing with something that has no effective
    // way to reference the old value of theThing, the old value
    // will never get cleaned up!
    someMethod: function () {}
  };
  // If you add `originalThing = null` here, there is no leak.
};
setInterval(replaceThing, 1000);
```

If we have a large object that is used by some closures, but not by any closures that we need to keep using, we should make sure that the local variable no longer points to it once we are done with it.
Unfortunately, these bugs can be pretty subtle.

### out of DOM references

In some cases we might store DOM nodes inside data-structures.
When we store a reference to a DOM node in a variable, or object, or array, there will be 2 references to that node.
One is our variable, and one is in the DOM tree.

To get rid of such element we need to make both references unreachable.

One major memory leak can happen when we don't clear references to nested DOM nodes.

Let's say we have a `table` in the DOM and we keep a reference to one of its cells inside our program.

At some point we decide to remove the `table` from the DOM, but we keep the reference to the cell.
We might think that the garbage collector will remove the `table`, leaving only that single cell alive.
But that's not the case.
The cell is a child node of the `table` and DOM nodes keep a reference to their parent. This means that our reference to the single cell, will actually keep alive the whole `table`.

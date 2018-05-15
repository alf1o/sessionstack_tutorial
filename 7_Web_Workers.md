# Web Workers

[original article](https://blog.sessionstack.com/how-javascript-works-the-building-blocks-of-web-workers-5-cases-when-you-should-use-them-a547c0757f6a)

## single thread and async programming

JavaScript is a single threaded language, but we can leverage Web APIs, together with the callback queue and the event loop to execute some operations async.

This allows us to have our UIs work while the browser waits for network requests, or listens for inputs/events.

Web APIs work like separate threads that JS can't directly access but can still call to execute predefined tasks.

But Web APIs don't cover all the cases in which we would like to have async code.
If our program needs to perform some heavy calculations, something like a huge `for` loop, the code will be executed sync, thus blocking the UI.

What we usually do in these cases is to break down the heavy process into smaller steps so that other processes, like UI re-rendering, callbacks, etc..., can be executed in-between these steps (cooperative concurrency):

For example:
```js
// `data` is a very big array
function heavyWork(data) {
  return data.filter(/* predicate */).map(/* projectionFunction */);
}
```
can be instead be written as:
```js
const res = [];
function heavyWorkInSmallSteps(data) {
  res.concat(
    // Rather than working on the whole array at once, we only work on a small
    // chunk (1000 items in this case).
    data.splice(0, 1000).filter(/* ... */).map(/* ... */)
  );
  // If the `data` array is not empty, we schedule some more work for "later".
  if (data.length) setTimeout(() => heavyWorkInSmallSteps(data), 0);
}
```
The `setTimeout(cb, 0);` "hack" allows us to immediately push a task at the end of the callback queue.
In this way tasks already scheduled can be executed between steps and the UI isn't stuck.

## what are web workers

But wouldn't it be better to execute these calculations off of the main thread?
Thanks to HTML5 we can, by using Web Workers.

Web Workers are in-browser threads that can be used to execute JS code.
They allow us to run CPU intensive tasks in the background without affecting the UI.
In fact all operations in the main program and the worker program happen in parallel.

Web Workers are not part of JS itself, but rather they are provided by the browser and can be accessed by JS.

There are 3 kinds of Web Workers:
- Dedicated Workers, can only communicate with the main process
- Shared Workers, can communicate with all processes running on the same origin
- Service Workers (progressive web apps)

## how to use web workers

The browser can provide multiple instances of the JS Engine, each on its own thread, each running its own files.
Thus Web Workers need their own `.js` file.

To create a Worker from the main program we would use the `Worker` constructor passing in the location of the Worker file:
`var worker = new Worker('worker.js');`

If the `worker.js` file exists and is accessible, the browser will spawn a new thread and there download the file asynchronously.

Once the download is done, the file will be executed and the Worker starts its job.

If the path to the file returns a `404`, the Worker will fail silently.

The main program and the Worker don't share resources with each other, but instead communicate via a `message` event system to which both have to subscribe.

To trigger the event inside a file, we use the `postMessage` function from the other file: `worker.postMessage();`.

`postMessage` accepts either a string or a JSON object as first argument. That represents the data that one thread shares with the other.

So in case we wanted the Worker to like `filter` and `map` an array, we would have to pass this array via `postMessage`:
```js
var worker = new Worker('worker.js');

function startComputation() {
  // We send a message to the `worker` asking it to call the function
  // `heavyWork` on the `data` array.
  worker.postMessage({ fn: 'heavyWork', data });
}

// We listen for when the `worker` will send us a message.
// The `event` object has a `data` property which is what the `worker` is
// sending us back after the computation.
worker.addEventListener('message', evt => console.log(evt.data), false);
```

The Worker file might look something like this:
```js
// In case of Dedicated Workers the listener is added implicitly to the main
// thread.
addEventListener('message', evt => {
  // Just like before the `event` object has a `data` property on it.
  var data = evt.data;
  switch(data.fn) {
    case 'heavyWork':
      // The `heavyWork` function is defined somewhere in the Worker file.
      var result = heavyWork(data.data);
      // After the work is done, we send the `result` to the main program.
      postMessage(result);
      break;
    default:
      postMessage('unknown command');
  }
}, false);
```

Browsers will usually stringify/parse the objects we pass automatically, but still this work, plus the work of copying the object itself adds much overhead to this message system.

But still, what we gain from using a Worker is that all the computation is performed in isolation from the main thread which will be free to respond to other user interactions or events.

The event system between main program and Workers allows us to also handle errors:
```js
// We listen for the `error` event which will be fired every time an execption
// occurs in the Worker file.
worker.addEventListener('error', err => {
  // The `error` object has 3 properties on it.
  console.log('At line: ', err.lineno);
  console.log('In file: ', err.filename);
  console.log('Message: ', err.message);
}, false);
```

**Note:** to terminate a Worker, we can either use `close();` inside the Worker file, or `worker.terminate();` inside the main file.

The biggest downsides of Workers are that:
- the data we send back and forth must be each time copied and stringified/parsed
- the Worker doesn't share the global scope with the main thread
- the Worker can't manipulate the DOM

**NB:** we could also transfer the data. In this case we won't need to copy it.
In fact the data "ownership" is transferred, but the data object itself is not moved from its memory location.
Once we transfer data it will be inaccessible from the original file.
Transferring data is almost instantaneous, but is limited to `ArrayBuffer`s.

**NB:** ES2018 will add `SharedArrayBuffer`s and `Atomic`s which will allow main thread and Worker thread to actually share memory, thus working exactly as threads do in multi-threaded languages.

## when to use Workers

Like we saw Workers can't be used to manipulate the DOM, thus their biggest use case will be to take some heavy computation off the main program.

This might be the case for image manipulation, text manipulation (i.e. parsing), array manipulation (i.e. `map`, `filter`, `sort`), prefetching data (since Workers have access to ajax requests).
Also Service Workers are used to build progressive web apps, which will be able to work in a limited way even in case of bad connection or no connection at all.

# Event Loop and Async Programming

JavaScript is a single-threaded language, this means that its Engine can only have one call-stack and basically execute only one chunk of code at a time.

This is a limitation, because in case of very expensive operations our whole application will be stuck and unresponsive during execution.

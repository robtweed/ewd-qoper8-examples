# ewd-qoper8-examples

Rob Tweed <rtweed@mgateway.com>  
16 August 2017, M/Gateway Developments Ltd [http://www.mgateway.com](http://www.mgateway.com)  

Twitter: [@rtweed](https://twitter.com/rtweed)

Google Group for discussions, support, advice etc: [http://groups.google.co.uk/group/enterprise-web-developer-community](http://groups.google.co.uk/group/enterprise-web-developer-community)

© 2017 M/Gateway Developments Ltd

---

# Table of Contents

  * [Getting Started With ewd-qoper8](#toe-getting-started)
  * [Testing Adding a Message to the ewd-qoper8 Queue](#toe-adding-message-to-queue) 
  * [Handling Multiple Messages](#toe-handling-multiple-messages) 
  * [Defining The Worker Process Pool Size](#toe-defining-worker-process-pool-size)
  * [Defining a Worker Process Message Handler](#toe-defining-worker-process-message-handler)
      * [Example Worker Module](#toe-example-worker-module)
      * [The start event handler](#toe-start-event-handler) 
      * [The stop event handler](#toe-stop-event-handler) 
      * [The message event handler](#toe-message-event-handler) 
  * [Configuring ewd-qoper8 To Use Your Worker Handler Module](#toe-configuring-ewd-qoper8-to-use-your-worker-handler-module)
  * [Handling the Results Object Returned from a Worker](#toe-handling-results-object-returned-from-worker)
  * [Simpler Message Handling with the handleMessage Function](#toe-simpler-message-handling-with-handleMessage-function)

## <a id="toe-getting-started"></a>Getting Started With ewd-qoper8

ewd-qoper8 is pre-configured with a set of default methods that essentially take a “do nothing”
action. You can therefore very simply test ewd-qoper8 by writing and running the following script
file:
```javascript
var qoper8 = require('ewd-qoper8');
var q = new qoper8.masterProcess();
console.log(q.version() + ' running in process ' + process.pid);
q.start();
```
*(You'll find a copy of this in the /tests sub-folder within the ewd-qoper8 module folder - see test1.js)*

This example will start the ewd-qoper8 master process, with a worker process pool size of 1. It will
then sit waiting for messages to be added to the queue - which isn't going to happen in this script.
When ewd-qoper8 starts, it does not start any worker processes. A worker processes is only
started when:
- a message is added to the queue; AND
- no free worker process is available; AND
- the maximum worker process pool size has not yet been reached
So if you run the above script you should just see something like this:

```
robtweed@ubuntu:~/qoper8$ sudo node node_modules/ewd-qoper8/lib/tests/test1
ewd-qoper8 Build 3.0.0; 1 March 2016 running in process 12599
Worker Bootstrap Module file written to node_modules/ewd-qoper8-worker.js
========================================================
ewd-qoper8 is up and running. Max worker pool size: 1
========================================================
```

Notice the second line of output: ewd-qoper8 always automatically creates a file containing the core worker process logic from which it bootstraps itself. However, since the test script doesn't add any messages to ewd-qoper8's queue, no worker processes are created and the master process will just sit waiting.
To stop ewd-qoper8, just press CTRL & C within the process console, or send a SIGINT message
from another process, eg:
```
kill 10947
```
*Note: the number should be the process Id for the ewd-qoper8 master process*

In either case you should see something like the following:
```
^C*** CTRL & C detected: shutting down gracefully...
Master process will now shut down
```
Alternatively we could add something like the following at the end of the test script:
```javascript
setTimeout(function() {
 q.stop();
}, 10000);
```
This time, after 10 seconds you'll now see ewd-qoper8 shut itself down


## <a id="toe-adding-message-to-queue"></a>Testing Adding a Message to the ewd-qoper8 Queue

You use ewd-qoper8's `addToQueue()` method to add messages to the queue. Messages are JavaScript objects and should have a type property defined. The type value is entirely up to you. Defining a type assists in message and response handling. The built-in logging reports will assume your messages have a type property.

So we could do the following test (see test2.js in the /tests sub-folder):
```javascript
var qoper8 = require('ewd-qoper8');
var q = new qoper8.masterProcess();
q.on('started', function() {
 console.log(q.version() + ' running in process ' + process.pid);
 var messageObj = {
 type: 'testMessage1',
 hello: 'world'
 };
 this.addToQueue(messageObj);
});
q.start();
setTimeout(function() {
 q.stop();
}, 10000);
```
Any ewd-qoper8 activity should be defined within its 'started' event handler. In the example above you can see a message object being created and queued using `this.addToQueue()` within the
q.on('started') handler.

Running the above script should produce output similar to the following:
```
robtweed@ubuntu:~/qoper8$ sudo node node_modules/ewd-qoper8/lib/tests/test2
Worker Bootstrap Module file written to node_modules/ewd-qoper8-worker.js
========================================================
ewd-qoper8 is up and running. Max worker pool size: 1
========================================================
ewd-qoper8 Build 3.0.0; 1 March 2016 running in process 12576
no available workers
master process received response from worker 12581: {"type":"workerProcessStarted","ok":12581}
new worker 12581 started and ready so process queue again
checking if more on queue
worker 12581 received message: {"type":"testMessage1","hello":"world"}
master process received response from worker 12581:
{"type":"testMessage1","finished":true,"message":{"error":"No handler found for testMessage1
message"}}
Master process has finished processing response from worker process 12581 which is back in
available pool
signalling worker 12581 to stop
worker 12581 received message: {"type":"qoper8-exit"}
worker process 12581 will now shut down
exit received from worker process 12581
No worker processes are running
Master process will now shut down
```

You'll see that it reports starting a worker process to handle the message that was added to the queue - in this case with a process id (PID) of 12581.

As we've not yet specified a specific handler function for incoming messages, by default the worker process simply sends an error message back to the master process, which, as a result, returns the worker process to the available pool. The error message confirms that no handler was defined for the message.

Note that worker process 12581 is not shut down after it has finished handling the message, but persists, waiting and available for handling any future messages that are added to the queue (which, of course, in this example, doesn't happen).

As in the previous example, after 10 seconds, ewd-qoper8 is instructed to shut down. This time you can see that the master process first signals the worker process to shut down, then waits for a few seconds before shutting itself down.

If you leave ewd-qoper8 running, you'll see it regularly reporting the duration that each worker process has been idle. If the default idle limit (1 hour, or 3,600,000ms) is exceeded, ewd-qoper8 will shut down the worker process. As soon as a new message is added to the queue, ewd-oper8 will immediately start a new worker process to handle it.

As before, if you don't use ewd-qoper8's `stop()` method within your script, CTRL&C or a SIGINT message will shut down ewd-qoper8.


## <a id="toe-handling-multiple-messages"></a>Handling Multiple Messages

So far we've only queued a single message. It's worth looking at what happens if multiple messages are added to the queue. The following example (see test3.js in the /tests folder) will
queue two messages:
```javascript
var qoper8 = require('ewd-qoper8');
var q = new qoper8.masterProcess();
q.on('started', function() {
 var messageObj = {
 type: 'testMessage1',
 hello: 'world'
 };
 this.addToQueue(messageObj);
 messageObj = {
 type: 'testMessage2',
 hello: 'rob'
 };
 this.addToQueue(messageObj);
});
q.start();
setTimeout(function() {
 q.stop();
}, 5000);
```

The output should look something like this:
```
robtweed@ubuntu:~/qoper8$ sudo node node_modules/ewd-qoper8/lib/tests/test3
Worker Bootstrap Module file written to node_modules/ewd-qoper8-worker.js
========================================================
ewd-qoper8 is up and running. Max worker pool size: 1
========================================================
no available workers
no available workers
master process received response from worker 12593: {"type":"workerProcessStarted","ok":12593}
new worker 12593 started and ready so process queue again
checking if more on queue
more items found on queue - processing again
no available workers
worker 12593 received message: {"type":"testMessage1","hello":"world"}
master process received response from worker 12593:
{"type":"testMessage1","finished":true,"message":{"error":"No handler found for testMessage1
message"}}
Master process has finished processing response from worker process 12593 which is back
in available pool
checking if more on queue
worker 12593 received message: {"type":"testMessage2","hello":"rob"}
master process received response from worker 12593:
{"type":"testMessage2","finished":true,"message":{"error":"No handler found for testMessage2
message"}}
Master process has finished processing response from worker process 12593 which is back in available pool signalling worker 12593 to stop worker 12593 received message: {"type":"qoper8-exit"}
worker process 12593 will now shut down
exit received from worker process 12593
No worker processes are running
Master process will now shut down
```

Both messages are immediately added to the queue. Adding the first triggers a worker process to be started and the message is dispatched to the worker as soon as it has signalled back that it has started up and is ready for use.

Because we haven't specified otherwise, ewd-qoper8 defaults to a worker process pool size of 1, so the second message will remain on the queue until the single worker process (PID 12593 in this
above example) has finished handling the first message. As soon as the worker process becomes available, the second, queued message is dispatched to it for processing.

This sequence will continue automatically until the queue is empty. Try editing the example script so that it adds more messages to the queue, and see what happens.


## <a id="toe-defining-worker-process-pool-size"></a>Defining The Worker Process Pool Size

Configuration of ewd-qoper8 is done by modifying its various default properties. The best place to do this is within a “start” event handler.

For example, you can specify a worker process pool size of 2 in one of two ways:
```javascript
q.on('start', function() {
 this.setWorkerPoolSize(2);
});
```
or:
```javascript
q.on('start', function() {
 this.worker.poolSize = 2;
});
```

If you modify the previous example to use 2 worker processes, you should see a quite different
result, particularly if you queue up a larger number of message. For example, take the following
script (see test4.js in the /tests folder):
```javascript
var qoper8 = require('ewd-qoper8');
var q = new qoper8.masterProcess();
q.on('start', function() {
 this.setWorkerPoolSize(2);
});
q.on('started', function() {
 var noOfMessages = 5;
 var messageObj;
 for (var i = 0; i < noOfMessages; i++) {
 messageObj = {
 type: 'testMessage1',
 hello: 'world'
 };
 this.addToQueue(messageObj);
 }
});
q.start();
setTimeout(function() {
 q.getAllWorkerStats();
}, 5000);
setTimeout(function() {
 q.stop();
}, 10000);
```

The worker pool size is being increased to 2 within the `q.on('start')` handler. Within the `q.on('started')` handler, 5 messages are being queued. After 5 seconds, the `q.getAllWorkerStats()` method is invoked: this will ask each worker process to report back a number of vital statistics, including the number of messages it has processed.

Here's an example of the output generated by this script:
```
rrobtweed@ubuntu:~/qoper8$ sudo node node_modules/ewd-qoper8/lib/tests/test4
Worker Bootstrap Module file written to node_modules/ewd-qoper8-worker.js
========================================================
ewd-qoper8 is up and running. Max worker pool size: 2
========================================================
no available workers
no available workers
no available workers
no available workers
no available workers
master process received response from worker 12643: {"type":"workerProcessStarted","ok":12643}
new worker 12643 started and ready so process queue again
checking if more on queue
more items found on queue - processing again
no available workers
master process received response from worker 12644: {"type":"workerProcessStarted","ok":12644}
new worker 12644 started and ready so process queue again
checking if more on queue
more items found on queue - processing again
no available workers
worker 12643 received message: {"type":"testMessage1","hello":"world"}
master process received response from worker 12643:
{"type":"testMessage1","finished":true,"message":{"error":"No handler found for testMessage1
message"}}
Master process has finished processing response from worker process 12643 which is back in
available pool
checking if more on queue
more items found on queue - processing again
no available workers
worker 12643 received message: {"type":"testMessage1","hello":"world"}
master process received response from worker 12643:
{"type":"testMessage1","finished":true,"message":{"error":"No handler found for testMessage1
message"}}
Master process has finished processing response from worker process 12643 which is back in
available pool
checking if more on queue
more items found on queue - processing again
no available workers
worker 12644 received message: {"type":"testMessage1","hello":"world"}
master process received response from worker 12643:
{"type":"testMessage1","finished":true,"message":{"error":"No handler found for testMessage1
message"}}
Master process has finished processing response from worker process 12643 which is back in
available pool
checking if more on queue
worker 12643 received message: {"type":"testMessage1","hello":"world"}
worker 12643 received message: {"type":"testMessage1","hello":"world"}
master process received response from worker 12643:
{"type":"testMessage1","finished":true,"message":{"error":"No handler found for testMessage1
message"}}
Master process has finished processing response from worker process 12643 which is back in
available pool
master process received response from worker 12644:
{"type":"testMessage1","finished":true,"message":{"error":"No handler found for testMessage1
message"}}
Master process has finished processing response from worker process 12644 which is back in
available pool
worker 12643 received message: {"type":"qoper8-getStats"}
master process received response from worker 12643: {"type":"qoper8-stats","stats":{"pid":
12643,"upTime":"0 days 0:00:04","noOfMessages":6,"memory":
{"rss":"22.47","heapTotal":"9.03","heapUsed":"4.42"}}}
worker 12644 received message: {"type":"qoper8-getStats"}
master process received response from worker 12644: {"type":"qoper8-stats","stats":{"pid":
12644,"upTime":"0 days 0:00:04","noOfMessages":3,"memory":
{"rss":"22.09","heapTotal":"9.03","heapUsed":"4.40"}}}
signalling worker 12643 to stop
signalling worker 12644 to stop
worker 12643 received message: {"type":"qoper8-exit"}
worker process 12643 will now shut down
worker 12644 received message: {"type":"qoper8-exit"}
worker process 12644 will now shut down
exit received from worker process 12643
exit received from worker process 12644
No worker processes are running
Master process will now shut down
```

This time you'll see that ewd-qoper8 started 2 worker processes (PIds 12643 and 12644). In the example above, they handled 6 and 3 messages respectively. 

You'll find that the balance of messages handled by each worker may vary each time you run the script: it takes time for each worker process to start up and become ready for handling messages, so the first one may have handled several messages before the second one is ready.

You're probably wondering why a total of 9 messages was handled by the worker processes when only 5 were added to the queue. The reason is that a message is sent to each worker as part of its startup sequence, and another message was sent to each worker to ask for its vital statistics (as a result of running the `q.getAllWorkerStats()` method).


## <a id="toe-defining-worker-process-message-handler"></a>Defining a Worker Process Message Handler

In the examples above the worker processes have been applying a default “message” event handler. That's what's generating output lines such as this:
```
master process received response from worker 12643:
{"type":"testMessage1","finished":true,"message":{"error":"No handler found for testMessage1
message"}}
```

You customise the behaviour of the worker processes by creating a Worker Handler Module that ewd-qoper8 will load into each worker process when they are started.

Your module can define any of the following event handlers:
- **start**: this will be invoked whenever a worker process starts, at the point at which it is ready for use
- **stop**: this will be invoked just before a worker process closes down
- **message**: this is invoked whenever a message is received by the worker. This is where you define your logic for handling all messages (with the exception of ewd-qoper8's own control messages)

### <a id="toe-example-worker-module"></a>Example Worker Module

Here's a simple example of a worker module:
```javascript
module.exports = function() {
 this.on('start', function() {
 if (this.log) console.log('Worker process ' + process.pid + ' starting...');
 });
 this.on('message', function(messageObj, send, finished) {
 var response = {
 hello: 'world'
 };
 finished(response);
 });
 this.on('stop', function() {
 if (this.log) console.log('Worker process ' + process.pid + ' stopping...');
 });

};
```
You should always adhere to the pattern shown above:
- create a function that is exported from the module
- the function should have no arguments
- within the function you can define any or all of the worker's hander functions: `this.on('start')`, `this.on('end')` and `this.on('message')` handler functions.

### <a id="toe-start-event-handler"></a>The start event handler

The start event handler is where you can do things such as connect to databases or load other modules that you'll need in your worker process.

Within the handler's callback function, `'this'` provides you access to all the worker's methods and properties.

The `on('start')` event's callback function can take a single optional argument: `isFirst`

This argument will be true if this is the first time a worker process has been started since ewdqoper8 itself was started. This is useful in situations where you want to initialise data in a database each time ewd-qoper8 is started, but before any subsequent activity occurs.

### <a id="toe-start-event-handler"></a>The start event handler

Your stop event handler is where you can do things such as cleanly disconnect from databases or tidy up other resources before the worker process is terminated. Within the handler's callback function, `'this'` provides you access to all the worker's methods and properties.

### <a id="toe-message-event-handler"></a>The message event handler

The message event handler is where you'll define how to process all incoming messages that you have added to the queue. How they are processed is entirely up to you.

Its callback function provides three argument:
- **messageObj**: the raw incoming message object, sent from the master process's queue.
- **send**: a function that allows you to send a message to the master process without returning the
worker back to the available pool
- **finished**: a function that allows you to send a message to the master process, and signalling to
the master process that you have finished using the worker. The worker will be returned back to the available pool

Within the handler's callback function, `'this'` provides you access to all the worker's methods and properties. What you do with the message within the worker process is entirely up to you.
Once you've finished processing the message, you send the results back to the master process by invoking the `finished()` method.

This takes a single argument:
- **resultObj**: an object containing the results that are to be returned to the master process

On receipt of the message created by the `finished()` method, the master process will return the worker back to the available pool
You can optionally send more messages back to the master process during processing, prior to using the `finished()` method. To do this, use the `send()` method. This takes the same argument
as the `finished()` method (`resultObj`). 

The difference is that on receipt of the message, the master
process does not return the worker process to the available pool.
By default, both the `send()` and `finished()` functions return to the master process a message whose type property is the same as that of the message being handled. You can optionally use
the `send()` function to return messages with a different type property to the master process: to do this, simply define a type property in the `resultObj` object argument. Note that you cannot override the type property of the `finished()` function's result object.

Make sure that your `on('message')` handler logic always ends with an invocation of the `finished()` function, and only invoke it once - failure to do so will cause the worker process to not be released back to the available pool.


## <a id="toe-configuring-ewd-qoper8-to-use-your-worker-handler-module"></a>Configuring ewd-qoper8 To Use Your Worker Handler Module

You instruct ewd-qoper8 to load your worker module by setting the property `this.worker.module` from within the on('start') method handler. The module you specify will be loaded (using require()) into each worker process when it is started.

For example, if you saved your module in `./node_modules/exampleModule.js`, then you instruct ewd-qoper8 to load it as follows, eg:
```javascript
q.on('start', function() {
 this.worker.module = 'exampleModule';
});
```

If your module is saved elsewhere, modify the module path accordingly. For example if you look at the example script test5.js in the /tests folder, you'll see that it specifies:

```javascript
q.on('start', function() {
 this.setWorkerPoolSize(2);
 this.worker.module = 'ewd-qoper8/lib/tests/example-worker-module';
});
```
Try running the test5.js script to see the effect of this module on the messages returned to the master process.


## <a id="toe-handling-results-object-returned-from-worker"></a>Handling the Results Object Returned from a Worker

When a results object is returned from a Worker process, you normally define how the master process should handle it. So far we've been letting ewd-qoper8's default action to take place,
which is to simply report the returned result message to the console.

The basic mechanism for handling messages returned by worker processes is to define an `on('response')` handler, eg:

```javascript
q.on('response', function(responseObj, pid) {
 console.log('Received from worked ' + pid + ': ' + JSON.stringify(responseObj, null, 2));
});
```

As you can see above, the on('response') handler callback function provides two arguments:
- **resultsObj**: the raw incoming results object, sent from the worker process.
- **pid**: the process Id of the worker that handled the original message and sent this response

How you handle each returned message and what you do with it is up to you. Within the `on('response')` handler's callback function, `'this'` provides access to all of the master process's
properties and methods.

Note that your `on('response')` handler function method intercepts all messages returned by worker processes, including ewd-qoper8's own ones. You'll be able to distinguish them because their type will have 'qoper8-' as a prefix.

For a worked example, take a look at test6.js in the /tests folder:
```javascript
var qoper8 = require('ewd-qoper8');
var q = new qoper8.masterProcess();
q.on('start', function() {
 this.setWorkerPoolSize(2);
 this.worker.module = 'ewd-qoper8/lib/tests/test-workerModule1';
});
q.on('response', function(responseObj, pid) {
 console.log('Received from worked ' + pid + ': ' + JSON.stringify(responseObj, null, 2));
});
q.on('started', function() {
 var noOfMessages = 5;
 var messageObj;
 for (var i = 0; i < noOfMessages; i++) {
 messageObj = {
 type: 'testMessage1',
 hello: 'world'
 };
 this.addToQueue(messageObj);
 }
});
q.start();
setTimeout(function() {
 console.log(q.getStats());
 q.getWorkerAvailability(function(available) {
 console.log('Worker availability: ' + JSON.stringify(available));
 });
}, 5000);
setTimeout(function() {
 q.stop();
}, 10000);
```


## <a id="toe-simpler-message-handling-with-handleMessage-function"></a>Simpler Message Handling with the handleMessage Function

Although the `addMessage()` method and the `on('response')` event provide the basic mechanisms for handling messages within the ewd-qoper8 master process, the `handleMessage()` method
provides a much simpler and slicker mechanism.

The `handleMessage()` function has two arguments:
- **messageObj**: the message object to be added to the queue
- **callback**: a callback function which provides a single argument: responseObj containing the response object that was sent by the worker process that handled the message

Note that the callback function will fire for messages sent from the worker process using both the `send()` and `finished()` methods.

For a worked example, take a look at test7.js in the /tests folder:
```javascript
'use strict'
var qoper8 = require('ewd-qoper8');
var q = new qoper8.masterProcess();
q.on('start', function() {
 this.toggleLogging();
 this.worker.poolSize = 1;
 this.worker.module = 'ewd-qoper8/lib/tests/test-workerModule2';
});
q.on('stop', function() {
 console.log(this.getStats());
});
q.on('started', function() {
 var noOfMessages = 5;
 var messageObj;
 for (let i = 0; i < noOfMessages; i++) {
 messageObj = {
 type: 'testMessage1',
 hello: 'world'
 };
 this.handleMessage(messageObj, function(response) {
 console.log('** response received for message ' + i + ': ' + JSON.stringify(response));
 });
 }
});
q.start();
setTimeout(function() {
 q.stop();
}, 10000);
```
Here's the `on('message')` handler in the worker module for this example:
```javascript
 this.on('message', function(messageObj, send, finished) {
 send({
 info: 'intermediate message',
 pid: process.pid
 });
 count++;
 var results = {
 count: count,
 time: new Date().toString()
 };
 finished(results);
 });
 ```

Notice the way this is sending messages using both the `send()` and `finished()` functions. Both will be intercepted by the `handleMessage()` function's callback.

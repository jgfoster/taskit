# TaskIT

>Anything that can go wrong, will go wrong. -- Murphy's Law

Expressing and managing concurrent computations is indeed a concern of importance to develop applications that scale. A web application may want to use different processes for each of its incoming requests. Or maybe it wants to use a "thread pool" in some cases. In other case, our desktop application may want to send computations to a worker to not block the UI thread. 

Processes in Pharo are implemented as green threads scheduled by the virtual machine, without depending on the machinery of the underlying operating system. This has several consequences on the usage of concurrency we can do:

- Processes are cheap to create and to schedule. We can create as many as them as we want, and performance will only degrade is the code executed in those processes do so, what is to be expected.
- Processes provide concurrent execution but no real parallelism. Inside Pharo, it does not matter the amount of processes we use. They will be always executed in a single operating system thread, in a single operating system process.

Besides how expensive it is to create a process is an important concern to decide how we organize our application processes, a more difficult task arises when we want to synchronize such processes. For example, maybe we need to execute two processes concurrently and we want a third one to wait the completion of the first two before starting. Or maybe we need to maximize the parallelism of our application while enforcing the concurrent access to some piece of state. And all these issues require avoiding the creation of deadlocks.

TaskIT is a library that ease Process usage in Pharo. It provides abstractions to execute and synchronize concurrent tasks, and several pre-built mechanisms that are useful for many application developers. This chapter explores starts by familiarizing the reader with TaskIT's abstractions, guided by examples and code snippets. At the end, we discuss TaskIT extension points and possible customizations.

## Downloading

TODO

## Asynchronous Tasks

TaskIT's main abstraction are, as the name indicates it, tasks. A task is a unit of execution. By splitting the execution of a program in several tasks, TaskIT can run those tasks concurrently, synchronize their access to data, or order even help in ordering and synchronizing their execution.

### First Example

Launching a task is as easy as sending the message `schedule` to a block closure, as it is used in the following first code example:
```smalltalk
[ 1 + 1 ] schedule.
```
>The selector name `schedule` is chosen in purpose instead of others such as run, launch or execute. TaskIT promises you that a task will be *eventually* executed, but this is not necessarilly right away. In other words, a task is *scheduled* to be executed at some point in time in the future.

This first example is however useful to clarify the first two concept but it remains too simple. We are schedulling a task that does nothing useful, and we cannot even observe it's result (*yet*). Let's explore some other code snippets that may help us understand what's going on.

The following code snippet will schedule a task that prints to the `Transcript`. Just evaluating the expression below will make evident that the task is actually executed. However, a so simple task runs so fast that it's difficult to tell if it's actually running concurretly to our main process or not.
```smalltalk
[ 'Happened' logCr ] schedule.
```
The real acid test is to schedule a long-running task. The following example schedules a task that waits for a second before writing to the transcript. While normal synchronous code would block the main thread, you'll notice that this one does not. 
```smalltalk
[ 1 second wait.
'Waited' logCr ] schedule.
```

### Schedule vs fork
You may be asking yourself what's the difference between the `schedule` and `fork`. From the examples above they seem to do the same but they do not. In a nutshell, to understand why `schedule` means something different than `fork`, picture that using TaskIT two tasks may execute inside a same process, or in a pool of processes, while `fork` creates a new process every time.

You will find a longer answer in the section below explaining *runners*. In TaskIT, tasks are not directly scheduled in Pharo's global `ProcessScheduler` object as usual `Process` objects are. Instead, a task is scheduled in a task runner. It is the responsibility of the task runner to execute the task.

## Retrieving a Task's Result with Futures

In TaskIT we differentiate two different kind of tasks: some tasks are just *scheduled* for execution, they produce some side-effect and no result, some other tasks will produce (generally) a side-effect free value. When the result of a task is important for us, TaskIT provides us with a *future* object. A *future* is no other thing than an object that represents the future value of the task's execution. We can schedule a task with a future by using the `future` message on a block closure, as follows.

```smalltalk
aFuture := [ 2 + 2 ] future.
```

One way to see futures is as placeholders. When the task is finished, it deploys its result into the corresponding future. A future then provides access to its value, but since we cannot know *when* this value will be available, we cannot access it right away. Instead, futures provide an asynchronous way to access it's value by using *callbacks*. A callback is an object that will be executed when the task execution is finished.  

>In general terms, we do not want to **force** a future to retrieve his value in an asynchronous way.
>By doing so, we would be going back to the synchronous world, blocking a process' execution, and not exploiting concurrency.
>Later sections will discuss about synchronous (blocking) retrieval of a future's value.

A future can provide two kind of results: either the task execution was a success or a failure. A success happens when the task completes in a normal way, while a failure happens when an uncatched exception is risen in the task. Because of these distinction, futures allow the subscription of two different callbacks using the methods `onSuccessDo:` and `onFailureDo:`.

In the example below, we create a future and subscribe to it a success callback. As soon as the task finishes, the value gets deployed in the future and the callback is called with it.
```smalltalk
aFuture := [ 2 + 2 ] future.
aFuture onSuccessDo: [ :result | result logCr ].
```
We can also subscribe callbacks that handle a task's failure using the `onFailureDo:` message. If an exception occurs and the task cannot finish its execution as expected, the corresponding exception will be passed as argument to the failure callback, as in the following example.
```smalltalk
aFuture := [ Error signal ] future.
aFuture onFailureDo: [ :error | error sender method selector logCr ].
```

Futures accept more than one callback. When its associated task is finished, all its callbacks will be *scheduled* for execution. In other words, the only guarantee that callbacks give us is that they will be all eventually executed. However, the future itself cannot guarantee neither **when** will the callbacks be executed, nor **in which order**. The following example shows how we can subscribe several success callbacks for the same future.

```smalltalk
future := [ 2 + 2 ] future.
future onSuccessDo: [ :v | FileStream stdout nextPutAll: v asString; cr ].
future onSuccessDo: [ :v | 'Finished' logCr ].
future onSuccessDo: [ :v | [ v factorial logCr ] schedule ].
future onFailureDo: [ :error | error logCr ].
```

Callbacks work wether the task is still running or already finished. If the task is running, callbacks are registered and wait for the completion of the task. If the task is already finished, the callback will be immediately scheduled with the already deployed value. See below a code examples that illustrates this: we first create a future and subscribes a callback before it is finished, then we  wait for its completion and subscribe a second callback afterwards. Both callbacks are scheduled for execution.

```smalltalk
future := [ 1 second wait. 2 + 2 ] future.
future onSuccessDo: [ :v | v logCr ].

2 seconds wait.
future onSuccessDo: [ :v | v logCr ].
```

## Task Runners: Controlling How Tasks are executed 

So far we created and executed tasks without caring too much on the form they were executed. Indeed, we knew that they were run concurrently because they were non-blocking. We also said already that the difference between a `schedule` message and a `fork` message is that scheduled messages are run by a **task runner**.

A task runner is an object in charge of executing tasks *eventually*. Indeed, the main API of a task runner is the `schedule:` message that allows us to tell the task runner to schedule a task.
```smalltalk
aRunner schedule: [ 1 + 1 ]
```

A nice extension built on top of schedule is the  `future:` message that allows us to schedule a task but obtain a future of its eventual execution.

```smalltalk
future := aRunner future: [ 1 + 1 ]
```

Indeed, the messages `schedule` and `future` we have learnt before are only syntax-sugar extensions that call these respective ones on a default task runner. This section discusses several useful task runners already provided by TaskIT.

### New Process Task Runner

A new process task runner, instance of `TKTNewProcessTaskRunner`, is a task runner that runs each task in a new separate Pharo process. 

```smalltalk
aRunner := TKTNewProcessTaskRunner new.
aRunner schedule: [ 1 second wait. 'test' logCr ].
```
Moreover, since new processes are created to manage each task, scheduling two different tasks will be executed concurrently. For example, in the code snippet below, we schedule twice a task that printing the identity hash of the current process.

```smalltalk
aRunner := TKTNewProcessTaskRunner new.
task := [ 10 timesRepeat: [ 10 milliSeconds wait.
				('Hello from: ', Processor activeProcess identityHash asString) logCr ] ].
aRunner schedule: task.
aRunner schedule: task.
```

The generated output will look something like this:

```
'Hello from: 887632640'
'Hello from: 949846528'
'Hello from: 887632640'
'Hello from: 949846528'
'Hello from: 949846528'
'Hello from: 887632640'
'Hello from: 949846528'
'Hello from: 887632640'
'Hello from: 949846528'
'Hello from: 887632640'
'Hello from: 949846528'
'Hello from: 887632640'
'Hello from: 949846528'
'Hello from: 887632640'
'Hello from: 949846528'
'Hello from: 887632640'
'Hello from: 949846528'
'Hello from: 887632640'
'Hello from: 949846528'
'Hello from: 887632640'
```

First, you'll see that a different processes is being used to execute each task. Also, their execution is concurrent, as we can see the messages interleaved.

### Local Process Task Runner

The local process runner, instance of `TKTLocalProcessTaskRunner`, is a task runner that executes a task in the caller process. In other words, this task runner does not run concurrently. Executing the following piece of code:
```smalltalk
aRunner := TKTLocalProcessTaskRunner new.
future := aRunner schedule: [ 1 second wait ].
```
is equivalent to the following piece of code:
```smalltalk
[ 1 second wait ] value.
```
or even:
```smalltalk
1 second wait.
```

While this runner may seem a bit naive, it may also come in handy to control and debug task executions. Besides, the power of task runners is that they offer a polymorphic API to execute tasks.

### The Worker Runner

The worker runner, instance of `TKTWorker`, is a task runner that uses a single process to execute tasks from a queue. The worker's single process removes one-by-one the tasks from the queue and executes them sequenceally. Then, schedulling a task into a worker means to add the task inside the queue.

A worker manages the life-cycle of its process and provides the messages `start` and `stop` to control when the worker thread will begin and end.

```smalltalk
worker := TKTWorker new.
worker start.
worker schedule: [ 1 + 5 ].
worker stop.
```

By using workers, we can control the amount of alive processes and how tasks are distributed amongst them. For example, in the following example three tasks are executed sequenceally in a single separate process while still allowing us to use an asynchronous style of programming.

```smalltalk
worker := TKTWorker new start.
future1 := worker future: [ 2 + 2 ].
future2 := worker future: [ 3 + 3 ].
future3 := worker future: [ 1 + 1 ].
```

Workers can be combined into *worker pools*. Worker pools are discussed in a later section.

### Managing Runner Exceptions

As we stated before, in TaskIT the result of a task can be interesting for us or not. In case we do not need a task's result, we will schedule it usign the `schedule` or `schedule:` messages. This is a kind of fire-and-forget way of executing tasks. On the other hand, if the result of a task execution interests us we can get a future on it using the `future` and `future:` messages. These two ways to execute tasks require different ways to handle exceptions during task execution.

First, when an exception occurs during a task execution that has an associated future, the exception is forwarded to the future. In the future we can subscribe a failure callback using the `onFailureDo:` message to manage the exception accordingly.

However, on a fire-and-forget kind of scheduling, the execution and results of a task is not anymore under our control. If an exception happens in this case, it is the responsibility of the task runner to catch the exception and manage it gracefully. For this, each task runners is configured with an exception handler in charge of it. TaskIT exception handler classes are subclasses of the abstract `TKTExceptionHandler` that defines a `handleException:` method. Subclasses need to override the `handleException:` method to define their own way to manage exceptions.

TaskIt provides by default a `TKTDebuggerExceptionHandler` that will open a debugger on the raised exception. The `handleException:` method is defined as follows:

```smalltalk
handleException: anError 
	anError debug
```

Changing a runner's exception handler can be done by sending it the `exceptionHandler:` message, as follows:

```smalltalk
aRunner exceptionHandler: TKTDebuggerExceptionHandler new.
```

## The Worker pool

A worker pool is our implementation of a threads pool. Its main purpose is to provide with several worker runners and decouple us from the management of threads/processes. Worker pools are built on top of TaskIT, inside the PoolIT package. A worker pool, instance of `PITWorkersPool`, manages several worker runners. All runners inside a worker pool shared a single task queue. We can schedule a task for execution using the `dispatch:` message.

```smalltalk
dispatcher := PITWorkersPool instance. 
future := dispatcher dispatch: [ 1+1 ] asTask.
future value = 2
```

By default, a worker pool spawns two workers during it initialization (which is lazy). We can add more workers to the pool with the `addWorker` message and remove them with the `removeWorker` message.

```smalltalk
dispatcher := PITWorkersPool instance. 
dispatcher addWorker.
dispatcher
    removeWorker;
    removeWorker
```

The `removeWorker` message send will fail if there is no workers available to remove. The removed worker will stop after it finishes any task it is running, and it will not be available for usage any more. The last remaining reference to this worker is given as return of the message.

Finally, there is a fancy way to schedule tasks into the singleton pool of workers.

```smalltalk
future := [ 2 + 2 ] scheduleIt. 
```

## Advanced Futures



# To Review

### Synchronous result retrieval

The simplest way to interact with a future is synchronously. That is, when asking for a future's value, it will block the actual process until the value is available. We can do that by sending our future the message `value`.

```smalltalk
future := [ 2 + 2 ] shootIt.
self assert: future value equals: 4.
```

However, it could have happened that the finished in an erroneous state, with an exception. In such case, the exception that was thrown inside the task's execution is forwarded to the sender of `value`.

```smalltalk
future := [ SomeError signal ] shootIt.
[ future value ] on: SomeError do: [ :error | "We handle the error" ].
```

A future can also tell us if the task is already finished or not, by sending it the message `isValueAvailable`. The `isValueAvailable` message, opposedly to the `value` message, will not block the caller's process but return immediately a boolean informing if the task has finished.

```smalltalk
future := [ 2 + 2 ] shootIt.
future isValueAvailable.
```

However, waiting synchronously or polling for the task to be finished can be a waste of CPU time sometimes. For those cases when completely synchronous execution does not fit, TaskIT provides an alternative of retrieving a value with a timeout option, using the `valueTimeoutMilliseconds:` message. When we specify a timeout, we can also provide a block to handle the timeout case using the `valueTimeoutMilliseconds:ifTimeout:`. If we choose not to provide such a block, the default behavior in case of timeout is to throw a `TKTTimeoutError` exception.

```smalltalk
future := [ (Delay forMilliseconds: 100) wait ] shootIt.

future
    valueTimeoutMilliseconds: 2
    ifTimeout: [ "if it times out we execute this block"].

future valueTimeoutMilliseconds: 2.
```

### Lazy result resolution

A third way to work with futures is to ask them for a lazy result. A lazy result is an object that represents, almost transparently, the value of the task execution. This lazy result will be (using some reflective Pharo facilities) the value of the result once it is available, or under demand (for example, when a message is sent to it). Lazy results support a style of programming that is close to the synchronous style, while performing asynchronously if the result is not used. 

```smalltalk
future := [ employee computeBaseSallary ] shootIt.
result := future asResult.

subTotal := employee sumSallaryComponents

result + subTotal
```

@@comment explain the code

Note: Lazy results are to be used with care. They use Pharo's `become:` facility, and so, it will scan the system to update object references.

Lazy results can be used to easily synchronize tasks. One task running in parallel with another one and waiting for it to finish can use a lazy result object to perform transparently as much work as it can in parallel and then get blocked waiting for the missing part. Only when the result object is sent a message the 

```smalltalk
future := [ employee computeBaseSallary ] shootIt.
baseSallary := future asResult.

[ employee sumSallaryComponents + baseSallary ] shootIt value.
```

## Customizing TaskIT

### Custom Tasks

When you need to customize a task, the most important thing is to mind the main invocation method. 
	
```smalltalk 
runOnRunner: aRunner withFuture: aFuture

	| value |
	self setUpOnRunner: aRunner withFuture: aFuture.
	[
		[
			value := self executeWithFuture: aFuture. 
		] on: Error do: [ : exception |
			^ aRunner deployError: exception intoFuture: aFuture.
		].
		aRunner deployValue: value intoFuture: aFuture.
	] ensure: [
		self tearDownOnRunner: aRunner withFuture: aFuture.
		aRunner noteFutureHasFinished: aFuture.
	].
```

The task execution life cycle is defined here. 
	
It has a setup, execution and teardown	 time that is always executed. 
In this method we also have two important parts the deploy of the result (success or error) and the notification of a future as finished. (The future window is not just the task running, it is all the task execution life time. From the setup to the teardown).

So, if you need a task to setUp resources, or have some cleanup post processing, in the same process line, do not hesitate in subclassing and using this prepared hooks.  

```smalltalk
TKTSubClassedTask>>#setUpOnRunner: aRunner withFuture: aFuture.
TKTSubClassedTask>>#tearDownOnRunner: aRunner withFuture: aFuture.
```

By other side, if what you need is to change the execution it self (Maybe the main invocation method is not really suitable for you), remember always to notice the runner about the finishing of an execution, by sending the proper notification inside your overridden method.

```smalltalk
TKTSubClassedTask>>#runOnRunner: aRunner withFuture: aFuture
	"..."
	aRunner noteFutureHasFinished: aFuture.
	"..."
```
### Custom Task Runners

## Conclusion
	
In this chapter we present TaskIT framework for dealing with common concurrent architecture problems. We covered how to start a create a task from a block, how does that task run into a runner. We covered also futures as way to obtain a value, and to have a gate to synchronise your threads explicitly, and covered lazy results for synchronising your threads implicitly.

Finally we explain also how TaskIT deal with thread pools, explaining how to use it, and the impact in the global system performance.  


%!!ActIT: A Simple Actor Library on top of TaskIT



%!!TODOs

%- Discuss with Santi: What should be a good behavior if an error occurrs during a callback?
%- Does it make sense to put callbacks on a task (besides or instead putting it on the future)?
%- What about implementing lazy results with proxies (and do just forwarding?)?
%- ExclusiveVariable finalize is necesary?
%- Lazy result can be cancelled?
%- interruptCurrentTask

%	currentTask ifNotNil: [ 
%		currentTask value isProcessFinished ifFalse: [
%			currentTask  priority: 10.
%			workQueue do: currentTask.
%		].
%	].
%- cleanup wtF?
%- por que hay que ejecutar esto en un task?
%self scheduleTask: [ keepRunning set: false ] asTask.


% Local Variables:
% eval: (flyspell-mode -1)
% End:
https://docs.oracle.com/javase/tutorial/essential/concurrency/index.html

`Thread.sleep` causes the current thread to suspend execution for a specified period. It can also be used for pacing:
```java
for (int i = 0; i < importantInfo.length; i++) {
    //Pause for 4 seconds
    Thread.sleep(4000);
    //Print a message
    System.out.println(importantInfo[i]);
}
```
Method using `sleep` can throw `InterruptedException`, which can happen when another thread interrupts the current thread with “sleep” is active.

An `interrupt` is an indication to a thread that it should stop what it is doing and do something else. It is very common for the thread to terminate.

Thread can support its interruption by catching `InterruptedException`.

If thread goes a long time without calling method that can throw `InterruptedException`, then it must periodically invoke `Thread.interrupted()` to check it.

One can also do the following:
```java
if (Thread.interrupted()) {
    throw new InterruptedException();
}
```

This throws exception and allows `try-catch` take care of interruption.

(Note: another thread can try to interrupt the thread, and it might not necessarily finish right away – depends on whether methods throwing InterruptedException are called or if thread itself is checking the interruption flag).

Interesting: calling `Thread.interrupt` sets interrupt status. When a thread checks for interrupt by invoking the static method `Thread.interrupted()` – the flag is cleared. The non-static method `isInterrupted()` is read-only and is for other threads to check this thread’s status. Also, any method that exists by throwing an `InterruptedException` clears the flag.


The `Thread.join` method makes one thread wait for the completion of another. The thread that calls `t.join()` pauses its execution until `t` is terminated.

One can specify for how long to pause execution – but it’s anyway OS specific.

Happen-before relationships:
-	Main thread defines variables, then `t.start()` is called – the actions performed before `t.start()` will be visible to thread `t`
-	Main thread calls `t.join()` – the stuff that was done by thread `t` before the successful join will be visible to the following actions.
`synchronized` cannot be used on constructors, as it does not make sense for constructor to be accessed by multiple threads

`final` fields can be safely read through non-synchronized methods, once the object is constructed

Example of fine-grained synchronization:
```java
public class MsLunch {
    private long c1 = 0;
    private long c2 = 0;
    private Object lock1 = new Object();
    private Object lock2 = new Object();

    public void inc1() {
        synchronized(lock1) {
            c1++;
        }
    }

    public void inc2() {
        synchronized(lock2) {
            c2++;
        }
    }
}
```

if `c1` and `c2` do not interfere/depend on each other, they can use their own locks, instead of locking the same instance every time, reducing unnecessary blocking.

Atomic action is one that effectively happens all at once. It cannot stop in the middle: either happens completely, or does not happen at all.

Liveness is the application’s ability to execute in a timely manner. There are 3 liveness problems: deadlocks, starvation, and livelock.

Deadlock is when two or more threads are blocked forever, waiting for each other.

Starvation is when thread is unable to gain access to a shared resource, and is unable to make progress. Example: some synchronized method that takes long to execute.

Livelock is similar, but it’s more of a chain of dependencies of one thread on another and so on.

### Guarded blocks
Suppose that thread must wait until some flag is set. It could potentially do `while(true)`, but it’s wasteful. Instead, we can use `Object.wait()` which will wait until some significant action happens. When wait is called, the thread releases the lock and suspends execution. At some future time, another thread will acquire the same lock and invoke `Object.notifyAll()`, informing all threads waiting on that lock that something important has happened. Some time after the second thread releases the lock, the first thread reacquires the lock and resumes by returning from the invocation of wait. 

`notify()` will wake up only one thread – useful when you don’t care which thread it is.

### Lock objects
The biggest advantage of `Lock` objects over implicit locks is their ability to back out of an attempt to acquire a lock. The `tryLock` method backs out if the lock is not available immediately or before a timeout expires (if specified)

Note: avoiding deadlock:
-	First acquire lock for itself
-	Then, try acquire lock for the other object
-	Only if both locks were acquired – proceed
-	Don’t forget to release locks when done
Private final Lock lock = new ReentrantLock(); //how to create a lock

3 main methods:
-	`tryLock(`)
-	`lock()`
-	`unlock()`

### Executors
Three main executor interfaces:
- `Executor` – supports launching new tasks
  * Has single method – `e.execute(r)` that replaces `(new Thread(r)).start())`, and launches created thread immediately 
- `ExecutorService` – subinterface of the former, adds features that help manage the lifecycle of individual tasks and of executor itself
  *	Adds `submit()` method, which does not only accept a Runnable, but also `Callable` objects, which allow a task to return a value
  *	Provides methods to submit large collections of `Callable` objects
  *	Provides methods to manage shutdown of the executor
- `ScheduledExecutorService` – subinterface of the former, supports future and/or periodic execution of tasks
  * Adds `schedule()` method, which executes `Runnable` or `Callable` after a delay
  *	Defines `scheduleAtFixedRate` (repeated tasks) and `scheduleWithFixedDelay`

Most executor implementations use thread pools, consisting of worker threads. Worker threads is often used to execute multiple tasks. Worker threads minimize the overhead due to thread creation.
One common type of thread pool is the fixed thread pool which has a specified number of threads. If a thread is somehow terminated while it is still in use – it is replaced with a new thread. Tasks are submitted to the pool via an internal queue.

Fixed thread pools allow application to degrade gracefully. Example: if every HTTP request is handled by a separate thread.

-	`Executors.newFixedThreadPool`
-	`Executors.newCachedThreadPool` – creates threads on demand, reuses previously created threads, terminates threads that were not used for more than 60 seconds
-	`newSingleThreadExecutor` – executor with single worker; tasks are guaranteed to execute sequentially
Fork/Join
Fork/join framework is implementation of `ExecutorService` that helps you take advantage of multiple processors. The fork/join framework uses thread pool, but allows work-stealing: worker threads that run out of tasks can steal tasks from other threads that are still busy.

if (my portion of the work is small enough)
  do the work directly
else
  split my work into two pieces
  invoke the two pieces and wait for the results

Wrap the split work into ForkJoinTask subclass (such as RecursiveTask which can return an action, or RecursiveAction). Then this task can be passed to the invoke() (or invokeAll()) method of a ForkJoinPool instance.

Java SE 8 uses fork/join in `Arrays.parallelSort()`

Fork/join is also used by methods in the `java.util.streams` package, which are part of Lambda expressions???

Concurrent collections

-	`BlockingQueue` – FIFO data structure that blocks or times out when you attempt to add to a full queue, or retrieve from an empty queue
-	`ConcurrentMap` interface – defines useful atomic operations on maps:
  - `computeIfAbsent(key, mappingFunction)` – if key is not in the mapping, compute it and put into map if not null
  - `getOrDefault(key, defaultValue)`
  - `putIfAbsent(key, value)`
  - `replace(key, oldValue, newValue)` – replaces only if key is currently mapped to a given value


`ThreadLocalRandom`
 Available  since Java 7, for applications that expect to use random numbers from multiple threads or `ForkJoinTasks`

For concurrent access, instead of using `Math.random()`, results in less contention and better performance.
```java
Int r = ThreadLocalRandom.current().nextInt(1, 10);
```

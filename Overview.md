# MultithreadedTC #

William Pugh, Nat Ayewah

([HTML version of this introduction](http://www.cs.umd.edu/~ayewah/projects/multithreadedtc/introduction.html))

## What is MultithreadedTC? ##

  * _MultithreadedTC is a framework for testing concurrent applications. It features a metronome that is used to provide fine control over the sequence of activites in multiple threads._

Many failures in concurrent applications are not deterministic. They do not occur every time. Different interleavings of application threads yield different behaviors.

Concurrent application designers often want to run many (unrelated or loosely related) threads of activity to maximize throughput. Sometimes it is useful for test designers to mimic this style and run multiple threads, generating as many interleavings as possible. Many test frameworks support this paradigm (e.g. Contest).

MultithreadedTC is different. It supports test cases that exercise a _specific_ interleaving of threads. This is motivated by the principle that concurrent applications should be built using _small concurrent abstractions_ such as bounded buffers, semaphores and latches. Separating the concurrency logic from the rest of the application logic in this way makes it easier to understand and test concurrent applications. Since these abstractions are small, it should be possible to deterministically test every (significant) interleaving in separate tests.

But how can one guarantee a specific interleaving of different threads in the presence of blocking and timing issues? Consider the following example:

```
	[Initialize]

	ArrayBlockingQueue buf = new ArrayBlockingQueue(1);

	[Thread 1]		[Thread 2]

	buf.put(42);
	buf.put(17);
				assertEquals(42, buf.take());
				assertEquals(17, buf.take());
```

In this test, the second `put()` should cause Thread 1 to block. It should remain blocked until the first `take()` occurs in Thread 2. In order words, the test must guarantee that the first `take()` statement does not occur until _after_ the second `put()` statement. How could a designer guarantee this interleaving of the two threads?

One approach is to use `Thread.sleep()` in Thread 2 to delay its first statement long enough to "guarantee" that Thread 1 has blocked. But this approach makes the test timing-dependent -- timing can be thrown off by, say, an ill-timed garbage collector. This also does not work well when stepping through the code in a debugger.

Another common approach for coordinating activities in two threads is to use a `CountDownLatch`. A `CountDownLatch` will not work in this example as illustrated by the following diagram:

```
	[Initialize]

	ArrayBlockingQueue buf = new ArrayBlockingQueue(1);
	CountDownLatch c = new CountDownLatch(1);

	[Thread 1]		[Thread 2]

	buf.put(42);		c.await();
	buf.put(17);
	c.countDown();
				assertEquals(42, buf.take());
				assertEquals(17, buf.take());
```

Of course the problem is that the statement `c.countDown()` cannot be executed until after Thread 1 unblocks... which will not occur until Thread 2 `take()`s. In other words, this test is deadlocked!

MultithreadedTC provides an elegant solution to this problem, illustrated in the following example:

```
	class MTCBoundedBufferTest extends MultithreadedTestCase {
		ArrayBlockingQueue<Integer> buf;
		@Override public void initialize() {
			buf = new ArrayBlockingQueue<Integer>(1); 
		}

		public void thread1() throws InterruptedException {
			buf.put(42);
			buf.put(17);
			assertTick(1);
		}

		public void thread2() throws InterruptedException {
			waitForTick(1);
			assertEquals(Integer.valueOf(42), buf.take());
			assertEquals(Integer.valueOf(17), buf.take());
		}

		@Override public void finish() {
			assertTrue(buf.isEmpty());
		}
	}
```

Multithreaded has an internal metronome (or clock). But don't try and use it to set the tempo for your jazz band. **The clock only advances to the next tick when all threads are blocked**.

The clock starts at tick 0. In this example, the first statement in Thread 2, `waitForTick(1)`, makes it block until the clock reaches tick 1 before resuming. Thread 1 is allowed to run freely in tick 0, until it blocks on the second put. At this point, all threads are blocked, and the clock can advance to the next tick.

In tick 1, the first `take()` in Thread 2 is executed, and this frees up Thread 1. The final statement in Thread 1 asserts that the clock is in tick 1, in effect asserting that the thread blocked on the second `put()`.

This approach does not deadlock like the `CountDownLatch`, and is more reliable than `Thread.sleep()`. Some other high level observations are:

  * The test is encapsulated in a class that extends MultithreadedTestCase. Each of the threads is represented by a method whose name starts with "`thread`", returns `void`, and has no arguments. The `initialize()` method is invoked first; then the thread methods are invoked simultaneously in different threads; finally the `finish()` method is invoked when all threads have completed.

  * This test can be run using the following JUnit test:

```
	public void testMTCBoundedBuffer() throws Throwable {
		TestFramework.runOnce( new MTCBoundedBufferTest() );
	}
```
> This creates an instance of the test class and passes it to the `TestFramework`. The `TestFramework` creates the necessary threads, manages the metronome, and runs the test.

  * All the components of the test are represented using classes and methods, constructs that are recognizable to Java programmers.

  * The framework handles exceptions thrown by any of the threads, and propagates them up to JUnit. This solves a problem with anonymous Threads, whose exceptions are not detected by JUnit without some extra scaffolding provided by the test designer. (See Example XXX).

  * The clock is not necessarily incremented by units of one. When all threads are blocked it advances to the next requested tick specified by a `waitForTick()` method. If none of the threads are waiting for a tick, the test is declared to be in deadlock (unless one of the threads is in state `TIMED_WAITING`).

## So how does this work? ##

The class `TestFramework`, provides most of the scaffolding required to run **MultithreadedTC** tests. It uses reflection to identify all relevant methods in the test class, invokes them simultaneously in different threads. It regulates these threads using a separate _clock thread_.

The clock thread checks periodically to see if all threads are blocked. If all threads are blocked and at least one is waiting for a tick, the clock thread advances the clock to the next desired tick. The clock thread also detects deadlock (when all threads are blocked, none are waiting for a tick, and none are in state `TIMED_WAITING`), and can stop a test that is going on too long (a thread is in state `RUNNABLE` for too long.)

The test threads are placed into a new thread group, and any threads created by these test cases will be placed in this thread group by default. All threads in the thread group will be considered by the clock thread when deciding whether to advance the clock, declare a deadlock, or stop a long-running test.


## Cool! How do I use this? ##

**MultithreadedTC** tests are created by extending one of two classes:

  * class `MultithreadedTestCase` extends `junit.framework.Assert` and provides the base functionality required all tests. (NOTE: `MultithreadedTestCase` does NOT extend `junit.framework.TestCase`). A test using this class consists of:
    * an optional `initialize()` method,
    * one or more "thread" methods which are invoked in different threads,
    * an optional `finish()` method that is run after all threads have completed,
> In addition to the `MultithreadedTestCase` subclass, an additional JUnit test method is used to call one of the "run" methods in `TestFramework`. The run methods receive an instance of a `MultithreadedTestCase` and test it in different ways. The primary run methods are:
    * `runOnce(MultithreadedTestCase)` - run a test sequence once.
    * `runManyTimes(MultithreadedTestCase, int)` - run a test sequence as many times as specified by the int parameter, until one of the test runs fails
  * class `MultithreadedTest` extends `MultithreadedTestCase` and implements `junit.framework.Test`. So it can be added to a `junit.framework.TestSuite` and run directly. This class includes a `runTest()` method that calls: `TestFramework.runOnce(this)`. To change the way a test is run, override the `runTest()` method.
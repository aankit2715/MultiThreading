## Synchronization
Synchronization is a mechanism used in multithreading to control access to shared resources and prevent multiple threads from modifying data simultaneously in an unsafe manner.

- Its main purpose is to avoid:
    - Race Conditions
    - Data Inconsistency
    - Deadlocks
    = Thread Interference
    - Memory Visibility Issues

## Why Synchronization is Needed?
Consider two threads accessing the same bank account.

- Scenario Without Synchronization:
```java
class BankAccount {
    int balance = 1000;

    void withdraw(int amount) {
        balance = balance - amount;
    }
}
Two Threads
- Thread A withdraws ₹500
- Thread B withdraws ₹400

Expected Balance: 100

Possible Actual Result: (Without Synchronization)
500 or 600 or 100

Because both threads may read and update the balance simultaneously.
This problem is called a Race Condition.
```

## Race Condition:
A race condition occurs when;
- Multiple threads access shared data.
- At least one thread modifies the data.
- Final output depends on execution timing.

- Example of Race Condition:
```java
class Counter {

    int count = 0;

    public void increment() {
        count++;
    }
}

public class Main {

    public static void main(String[] args) throws Exception {

        Counter counter = new Counter();

        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 100000; i++) {
                counter.increment();
            }
        });

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 100000; i++) {
                counter.increment();
            }
        });

        t1.start();
        t2.start();

        t1.join();
        t2.join();

        System.out.println("Final Count = " + counter.count);
    }
}

Final Output can be: 0 or 5000 or 23976 or 2000000
```

## Critical Section:
A critical section is the portion of code that accesses shared resources. and Only one thread should execute the critical section at a time.


## Java provides synchronization using:

1. Synchronized Method: Entire method is locked.
- Ex:
```java
class Counter {
    int count = 0;

    synchronized void increment() {
        count++;
    }
}
```

* Advantages
- Easy to implement
- Thread-safe

* Disadvantages
- Lower performance
- Entire method locked

* Internal Working of Synchronized
Every Java object has a "Monitor Lock" and When thread enters synchronized code Acquire Lock, Execute Code, Release Lock means Only one thread can hold monitor lock at a time.


2. Synchronized Block:
Instead of locking whole method, lock only required code.
- Ex:
```java
public void process() {

    loadData();

    synchronized(this) {
        updateDatabase();
    }

    sendEmail();
}
Only database update is locked.
```

* Benefits
- Better performance
- Reduced lock duration
- More concurrency

3. Static Synchronization:
Static synchronization is used when multiple threads need to access a static (class-level) resource safely.
A static synchronized method acquires the Class-level lock, not the object lock.

- Ex.
```java
class Bank {
    static int transactionCount = 0;
    static synchronized void increment() {
        transactionCount++;
    }
}
Output:
Thread A enters
Thread B waits

A exits
B enters
```
Here, if we use simple synchronization then multiple objects of Bank class can use by multiple thread and as it will be instance level lock then multiple thread can manipulate transactionCount and we loss data consistancy.


4. Lock Interface:
More flexible than synchronized.
- Ex:
```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

class Counter {

    private Lock lock = new ReentrantLock();
    int count = 0;

    void increment() {

        lock.lock();

        try {
            count++;
        }
        finally {
            lock.unlock();
        }
    }
}
```

- Advantages of Lock over Synchronized

## Try Lock:
```java
if(lock.tryLock()) {
    try {
        // critical section
    }
    finally {
        lock.unlock();
    }
}
```
Thread doesn't wait forever.

## Timed Lock:
```java
lock.tryLock(5, TimeUnit.SECONDS);
```
Waits specific time.

## Fair Locking:
```java
Lock lock = new ReentrantLock(true);
```
Threads acquire lock in FIFO order.

## Reentrant Lock:
A thread can acquire the same lock multiple times.
```java
class Example {

    synchronized void A() {
        B();
    }

    synchronized void B() {
        System.out.println("Inside B");
    }
}
```
Same thread entering synchronized methods repeatedly is allowed.

5. Semaphore:
Controls number of threads accessing a resource.

- Ex.
```java
Semaphore semaphore = new Semaphore(3);
```
Only 3 threads can enter simultaneously.

```java
semaphore.acquire();
try {
    // resource usage
}
finally {
    semaphore.release();
}
```

- Useful for:
    1. Database connection pools
    2. API rate limiting
    3. Resource management


6. Atomic Variables:
They provide thread-safe operations without using synchronized blocks or explicit locks.
Think of them as:
Variables that can be updated safely by multiple threads using low-level CPU operations.

- Example:
```java
import java.util.concurrent.atomic.AtomicInteger;

class Counter {

    AtomicInteger count =
            new AtomicInteger(0);

    void increment() {
        count.incrementAndGet();
    }
}
```
Now multiple threads can update safely without explicit locking.

AtomicInteger
AtomicLong
AtomicBoolean
AtomicReference

- When to use Atomic Variables?
    1. A single variable is shared by multiple threads.
    2. Operations like increment/decrement must be thread-safe.
    3. You want better performance than synchronized.
    4. Lock-free programming is sufficient.


7. Volatile Keyword:
volatile provides visibility, but not atomicity.

- Why Do We Need Volatile?
In a multithreaded environment, each thread may keep a local copy (cache) of variables.
A thread may read a value from its cache instead of main memory.
As a result, one thread's updates might not be visible immediately to another thread.

- Ex. Without Volatile
```java
class Task {

    boolean stop = false;

    void run() {
        while (!stop) {
            // do work
        }

        System.out.println("Stopped");
    }
}

- Main Thread
Task task = new Task();

Thread t1 = new Thread(() -> task.run());

t1.start();

Thread.sleep(2000);

task.stop = true;
```

- Expectation:
After 2 seconds: stop becomes true and loop exits

- output:
Stopped

- What Can Actually Happen?
```java
Thread t1 reads stop=false
Stores it in CPU cache

Main thread updates stop=true

Thread t1 keeps using cached false

Result: Loop never stops (This is called a visibility problem.)
```
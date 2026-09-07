## Inter-Thread Communication (ITC)
Inter-Thread Communication (ITC) is the mechanism through which multiple threads within the same process exchange information, coordinate tasks, and synchronize access to shared resources.
Since threads share the same process memory space, they can communicate directly through shared variables, objects, buffers, queues, and synchronization mechanisms.

## Why Do Threads Need Communication?
- Imagine an e-commerce application:
    - Thread 1 receives customer orders.
    - Thread 2 validates payment.
    - Thread 3 updates inventory.
    - Thre`ad 4 sends confirmation emails.

- These threads must exchange information. If they don't communicate properly:
    - Orders may be processed twice.
    - Inventory may become inconsistent.
    - Confirmation emails may be sent before payment succeeds.

- Inter-thread communication ensures that all threads work together correctly.

## Ways of Threads Communication

1. Shared Memory:
Threads share the same memory space.

- Ex:
```java
class SharedData {
    int count = 0;
}
```
Multiple threads can read and update count.

- Problem
Two threads updating the same variable simultaneously can lead race condition.
```java
Expected Count = 2000
Actual Count = 1587
```

2. Wait(), Notify(), NotifyAll():
Used when one thread needs to wait until another thread completes some work.

## Example:
Producer-Consumer Scenario

- Producer Thread-
Produces data and places it in a buffer.

- Consumer Thread-
Consumes data from the buffer.

- Without communication:
    - Consumer may try to read before data exists.
    - Producer may overload buffer capacity.

## Working Flow
- Consumer:
```java
synchronized(buffer){
    while(buffer.isEmpty()){
        buffer.wait();
    }
}
```
Consumer sleeps until data becomes available.

- Producer:
```java
synchronized(buffer){
    buffer.add(item);
    buffer.notify();
}
```
Producer wakes consumer after adding data.

3. Blocking Queue (Modern Approach):
In enterprise Java applications, developers rarely use wait/notify directly.

- Instead:
```java
BlockingQueue<Order> queue =
       new LinkedBlockingQueue<>();
```

- Producer:
```java
queue.put(order);
```

- Consumer:
```java
Order order = queue.take();
```

- Benefits:
    - Thread-safe
    - No manual synchronization
    - Easy maintenance

4. Semaphore
Used to limit simultaneous access to a resource.

- Ex:
```java
Semaphore dbConnections =
          new Semaphore(10);
Only 10 threads can access database at a time.
```
- When one thread finishes:
- dbConnections.release();
- Another waiting thread gets permission.

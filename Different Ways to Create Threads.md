
## Ways to create Threads:

## 1. Extending Thread Class
- Override the run() method in a subclass of Thread.
```java
class MyThread extends Thread {
    public void run() {
        System.out.println("Thread running...");
    }
}
public class Demo {
    public static void main(String[] args) {
        MyThread t1 = new MyThread();
        t1.start();
    }
}
```

- Drawback: Java doesn't support multiple inheritance so after extending thread class, we can't extend another class.


## 2. Implementing Runnable Interface:
- Most recommended traditional approach.
```java
class MyRunnable implements Runnable {
    public void run() {
        System.out.println("Runnable thread running...");
    }
}
public class Demo {
    public static void main(String[] args) {

        MyRunnable task = new MyRunnable();

        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);
        t1.start();
        t2.start();
    }
}
```

- Advantage: (Major difference from Thread class)
1. We can still extends another class.
2. Multiple threads can execute the same task object.
3. Increase reusablity as Runnable task can be reused mutliple times.
Ex:
```java
class DepositTask implements Runnable {
    private int accountBalance = 100;
    @Override
    public void run() {
        
            accountBalance += 10;
            System.out.println(Thread.currentThread().getName() +
                               " deposited, balance: " + accountBalance);
         
    }
}

public class BankDemo {
    public static void main(String[] args) {
        DepositTask task = new DepositTask();

        Thread t1 = new Thread(task, "Customer-1");
        Thread t2 = new Thread(task, "Customer-2");
        Thread t3 = new Thread(task, "Customer-3");

        t1.start();
        t2.start();
        t3.start();
    }
}

Posible output:
Customer-1 deposited, balance: 110
Customer-3 deposited, balance: 120
Customer-2 deposited, balance: 130
```

- But in above example there is posiblity of overriding changes as multiple threads are manipulating accountBalance or same data so they can override each other changes, leading to a RACE Condition.
- we can prevent race condition by using Synchronization technique, AtomicInteger and Locks.

4. Thread with Lambda:
- As runnable contains only one abstract method run(), it is s Functional Interface. and Java allows functional interfaces to be implemented using lambdas.
Ex:
```java
public class LambdaThreadDemo {
    public static void main(String[] args) {
        // Creating a thread using lambda
        Thread t1 = new Thread(() -> {
            System.out.println("Thread 1 running with lambda!");
        });

        Thread t2 = new Thread(() -> {
            System.out.println("Thread 2 running with lambda!");
        });

        Thread t3 = new Thread(() -> {
            System.out.println("Thread 3 running with lambda!");
        });

        t1.start(); // start the thread
        t2.start();
        t3.start();
    }
}

Posible Output:
Thread 3 running with lambda!
Thread 1 running with lambda!
Thread 2 running with lambda!

order is not guraranteed because threads run concurrently.
```

- Why are Lambdas Prefered:
- 1. Less Code
- 2. Better Readability

## Two Approaches:

1. Assigning to a Variable:
```java
Thread t1 = new Thread(() -> {
    System.out.println("Thread 1 running with lambda!");
});
t1.start();
```
- You store the thread object in t1.
- This gives you control the thread later (e.g., t1.join(), t1.isAlive(), t1.interrupt()).
- Useful when you need to manage or monitor the thread.

2. Anonymous Thread (No Variable):
```java
new Thread(() -> {
    System.out.println("Thread 1 running with lambda!");
}).start();
```
- You don’t store the thread object.
- The thread starts immediately and runs its task.
- You cannot reference it later (no way to interrupt, join, or check status).
- Useful for fire-and-forget tasks where you don’t care about the thread after it starts.

## 4. Using ExecutorService (Industry Standard):

- Manage a pool of reuable threads instead of creating a new thread for each task.

## - Problem Without ExecutorService:
Suppose an e-commerce application receives 10,000 order requests, and if you do
```java
for(int i=1; i<=10000; i++) {
    new Thread(() -> processOrder()).start();
}
```
- so here we are creating 10000 Threads. Problem,
1. Huge memory consumption
2. Expensive thread creation
3. Excessive context switching
4. Posible OutOfMemoryError

## - Better Approach: Thread Pool
```java
ExecutorService executor = Executors.newFixedThreadPool(3);
```
- This creates 3 thread only.

Example:
```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ExecutorDemo {
    public static void main(String[] args) {
        // Create a fixed thread pool with 3 threads
        ExecutorService executor = Executors.newFixedThreadPool(3);

        // Submit tasks using lambda
        for (int i = 1; i <= 5; i++) {
            int taskId = i;
            executor.submit(() -> {
                System.out.println("Task " + taskId + " executed by " +
                                   Thread.currentThread().getName());
            });
        }

        // Shutdown executor after tasks are submitted
        executor.shutdown();
    }
}

- Possible output:
Task 1 executed by pool-1-thread-1
Task 2 executed by pool-1-thread-2
Task 3 executed by pool-1-thread-3
Task 4 executed by pool-1-thread-1
Task 5 executed by pool-1-thread-2
```

## Explanation

- Executors.newFixedThreadPool(3) → Creates a pool of 3 reusable threads.
- executor.submit(...) → Submits tasks to be executed by available threads.
- Threads are reused — notice how tasks 4 and 5 are executed by threads that already handled earlier tasks.
- shutdown() → Gracefully stops accepting new tasks and finishes existing ones.

## ThreadPoolExecutor:

- Behind the scenes:
Executors.newFixedThreadPool(5);

- actually creates:
ThreadPoolExecutor

- Production-grade applications often configure it directly.
```java
ThreadPoolExecutor executor =
        new ThreadPoolExecutor(
                5,      // core pool size
                10,     // max pool size
                60,
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(100)
        );
```		
		
- Meaning of corePoolSize = 5
The pool tries to keep 5 threads available.

- Meaning of maximumPoolSize = 10
1. This is the upper limit of threads that can exist in the pool.
2. It does not mean 10 threads are created immediately.
3. Additional threads (6 to 10) are created only when:
	3.1 All core threads are busy, AND
	3.2 The queue is full
4. What Happens After 10 Threads?
10 active threads
+
Queue full (100)

Now another task arrives. No thread available. Queue full. Maximum threads reached. then Executor rejects task, will throws "RejectedExecutionException".

- What About keepAliveTime = 60 seconds?
Extra threads beyond corePoolSize are temporary.

- What is LinkedBlockingQueue<>(100) meaning?
Means max 100 task can queue if all the threads are busy at same time.

## Best Practices (Industry Standard)
✅ Use Fixed Thread Pool:
- Executors.newFixedThreadPool(n)

## ExecutorService with Runnable vs Callable vs Future

1. Runnable: (No return statement):
```java
ExecutorService executor = Executors.newFixedThreadPool(1);
executor.submit(() -> {
    System.out.println("Hello");
});
```

2. Callable: (Has a return value):
- Callable is a functional interface in java.util.concurrent.
- Similar to Runnable, but:
- 1. Returns a value (V call() instead of void run()).
- 2. Can throw checked exceptions.
-  Perfect for tasks where you need a result (e.g., database query, computation).
```java
ExecutorService executor = Executors.newFixedThreadPool(1);
executor.submit(() -> {
    return "Hello";
});
```

3. Future: (If You Need the Result Later):
1. Future represents the result of an asynchronous computation.
2. Returned when you submit a Callable to an ExecutorService.
3. Provides methods:
- 1. get() → Retrieve result (blocks until ready).
- 2. isDone() → Check if task finished.
- 3. cancel() → Cancel task execution.
- 4. isCancelled() → Check if task was cancelled
```java
ExecutorService executor = Executors.newFixedThreadPool(1);
Future<String> future = executor.submit(() -> {
    return "Hello";
});
String result = future.get();
System.out.println(result);
executor.shutdown();
```

Example: Callable + Future:
```java
import java.util.concurrent.*;

public class CallableFutureDemo {
    public static void main(String[] args) throws Exception {
        // Create a thread pool with 2 threads
        ExecutorService executor = Executors.newFixedThreadPool(2);

        // Define Callable tasks
        Callable<Integer> task1 = () -> {
            System.out.println("Task 1 running...");
            Thread.sleep(1000); // simulate work
            return 10;
        };

        Callable<Integer> task2 = () -> {
            System.out.println("Task 2 running...");
            Thread.sleep(2000); // simulate work
            return 20;
        };

        // Submit tasks and get Future objects
        Future<Integer> future1 = executor.submit(task1);
        Future<Integer> future2 = executor.submit(task2);

        // Retrieve results (blocking until done)
        System.out.println("Result of Task 1: " + future1.get());
        System.out.println("Result of Task 2: " + future2.get());

        executor.shutdown();
    }
}

Posible Output:
Task 1 running...
Task 2 running...
Result of Task 1: 10
Result of Task 2: 20

```

2. Multiple ExecutorService Pools.
Example:
```java
import java.util.concurrent.*;

public class MultiplePoolsDemo {
    public static void main(String[] args) {
        // Pool for CPU-intensive tasks (fixed size)
        ExecutorService cpuPool = Executors.newFixedThreadPool(4);

        // Pool for I/O tasks (cached, grows/shrinks dynamically)
        ExecutorService ioPool = Executors.newCachedThreadPool();

        // Submit tasks to CPU pool
        for (int i = 1; i <= 3; i++) {
            int taskId = i;
            cpuPool.submit(() -> {
                System.out.println("CPU Task " + taskId + " executed by " +
                                   Thread.currentThread().getName());
            });
        }

        // Submit tasks to IO pool
        for (int i = 1; i <= 3; i++) {
            int taskId = i;
            ioPool.submit(() -> {
                System.out.println("IO Task " + taskId + " executed by " +
                                   Thread.currentThread().getName());
            });
        }

        // Shutdown both pools
        cpuPool.shutdown();
        ioPool.shutdown();
    }
}

Posible Output:
CPU Task 1 executed by pool-1-thread-1
CPU Task 2 executed by pool-1-thread-2
CPU Task 3 executed by pool-1-thread-3
IO Task 1 executed by pool-2-thread-1
IO Task 2 executed by pool-2-thread-2
IO Task 3 executed by pool-2-thread-3
```

## Advantages of Multiple pool:
- Workload Separation → Prevents I/O-heavy tasks from starving CPU-bound tasks.
- Resource Control → Each pool can have its own size and configuration.
- Isolation → Failures or overload in one pool don’t affect the other.
- Performance Tuning → You can optimize thread counts differently for CPU vs I/O tasks.

## Customize thread pool name along with Id:

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class CustomThreadPoolDemo {
    public static void main(String[] args) {
        // Atomic counter for thread IDs
        AtomicInteger counter = new AtomicInteger(1);

        // Create ExecutorService with custom factory
        ExecutorService executor = Executors.newFixedThreadPool(3,
             r -> new Thread(r, "MyPool-Thread-" + counter.getAndIncrement())
             );

        // Submit tasks
        for (int i = 1; i <= 5; i++) {
            int taskId = i;
            executor.submit(() -> {
                System.out.println(Thread.currentThread().getName() +
                                   " executing Task " + taskId);
            });
        }

        executor.shutdown();
    }
}

Posible Output:

MyPool-Thread-1 executing Task 1
MyPool-Thread-2 executing Task 2
MyPool-Thread-3 executing Task 3
MyPool-Thread-1 executing Task 4
MyPool-Thread-2 executing Task 5

```
 
## 6. Using CompletableFuture (Modern Java):
- Used heavily in microservices and async programming.
- Nowadays you often see:

```java
CompletableFuture<String> future =
    CompletableFuture.supplyAsync(() -> "Hello");
```

- instead of:
```java
Future<String> future =
    executor.submit(() -> "Hello");
```

because CompletableFuture provides:
```java
future.thenApply(...)
      .thenAccept(...)
      .exceptionally(...);
```
which makes async workflows easier.

## Example: (Real Industry)
- Suppose you're building an API:  GET /customer-dashboard/123
- The UI needs:
	1. Customer Details
	2. Orders
	3. Payment History
	4. Reward Points

- If you call them sequentially:
```java
	Customer customer = customerService.getCustomer(id); // 2 sec
	List<Order> orders = orderService.getOrders(id); // 3 sec
	List<Payment> payments = paymentService.getPayments(id); // 2 sec
	Reward reward = rewardService.getReward(id); // 1 sec

Total: 2 + 3 + 2 + 1 = 8 seconds
```

- Using CompletableFuture
Run everything in parallel.

```java
ExecutorService executor =
        Executors.newFixedThreadPool(4);

CompletableFuture<Customer> customerFuture =
        CompletableFuture.supplyAsync(
                () -> customerService.getCustomer(id),
                executor
        );

CompletableFuture<List<Order>> ordersFuture =
        CompletableFuture.supplyAsync(
                () -> orderService.getOrders(id),
                executor
        );

CompletableFuture<List<Payment>> paymentFuture =
        CompletableFuture.supplyAsync(
                () -> paymentService.getPayments(id),
                executor
        );

CompletableFuture<Reward> rewardFuture =
        CompletableFuture.supplyAsync(
                () -> rewardService.getReward(id),
                executor
        );
```		
		
- Wait for all:
```java
CompletableFuture.allOf(
        customerFuture,
        ordersFuture,
        paymentFuture,
        rewardFuture
).join();
```

- Build response:
```java
DashboardResponse response =
        new DashboardResponse(
                customerFuture.join(),
                ordersFuture.join(),
                paymentFuture.join(),
                rewardFuture.join()
        );
		
Execution time: ~3 seconds
```

- Chaining Operations Using thenApply():
Suppose after getting customer data, you want to transform it.
```java
CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() ->
                customerService.getCustomer(id))
        .thenApply(customer ->
                customer.getName().toUpperCase());
```				

- Calling Another Service Using thenCompose():
This is one of the most asked interview questions.
```java
CompletableFuture<Order> future =
        CompletableFuture.supplyAsync(() ->
                        customerService.getCustomer(id))
                .thenCompose(customer ->
                        CompletableFuture.supplyAsync(
                                () -> orderService
                                        .getLatestOrder(
                                                customer.getId())));
												
```												
												
- Combining Independent Results Using thenCombine():
Suppose, Price and Tax comes from different systems.
```java
CompletableFuture<Double> priceFuture =
        CompletableFuture.supplyAsync(
                () -> productService.getPrice());

CompletableFuture<Double> taxFuture =
        CompletableFuture.supplyAsync(
                () -> taxService.getTax());
				
Combine:
CompletableFuture<Double> totalFuture =
        priceFuture.thenCombine(
                taxFuture,
                (price, tax) -> price + tax
        );
		
Result:
Double total = totalFuture.join();  //join() is used to wait for the completion of a CompletableFuture and get its result.
```



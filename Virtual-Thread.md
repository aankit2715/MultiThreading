# What is a Virtual Thread?
A Virtual Thread is a lightweight thread managed by the JVM rather than directly by the operating system.  

- Traditional Thread (Platform Thread - Before Java 21):
```java
Thread thread = new Thread(() -> {
    System.out.println("Hello");
});
thread.start();

Each Java thread maps to an OS thread: 
Java Thread 1 --> OS Thread 1 
Java Thread 2 --> OS Thread 2
Java Thread 3 --> OS Thread 3
```

- Problems:
    1. High memory consumption
    2. Expensive creation/destruction
    3. Context switching cost
    4. Difficult to scale beyond a few thousand threads


- With Virtual Threads:
```java
Thread.startVirtualThread(() -> {
    System.out.println("Hello from Virtual Thread");
});
```

**Mapping becomes:**  
1,000,000 Virtual Threads ---> JVM Scheduler ---> 8 Platform Threads  
Many virtual threads share a small number of OS threads.

- Why Virtual Threads Were Needed:  
Using thread for DB

**Traditional Approach:**  

Thread
   |
   |---- waiting for DB Response (idle)
   |
   V
OS thread remains blocked and cannot do other work.

** With Virtual Threads:** 
Virtual Thread
     |
     |---- waiting
     |
     V
JVM parks virtual thread

Platform thread becomes free  

The JVM suspends the virtual thread and reuses the platform thread for another task. This dramatically increases scalability.  

- Virtual Thread Architecture:

    1. Platform Thread (Managed by Operating System):  
    Java Thread
        |
    Operating System Thread  

    Heavyweight.

    2. Virtual Thread (Managed by JVM Scheduler):  
    Virtual Thread
        |
        V
    JVM Scheduler
        |
    Carrier Thread
        |
    OS Thread  
    
    The actual OS thread executing virtual threads is known as a Carrier Thread.  


- Creating Virtual Threads:

**1. startVirtualThread():**
```java
Thread.startVirtualThread(() -> {
    System.out.println("Task");
});
```  

**2. Thread Builder:**  
```java
Thread vt = Thread.ofVirtual()
                  .name("worker-1")
                  .start(() -> {
                      System.out.println("Running");
                  });
```
				  
**3. Executor Service:**
```java
try (ExecutorService executor =
        Executors.newVirtualThreadPerTaskExecutor()) {

    executor.submit(() -> {
        System.out.println("Task 1");
    });

    executor.submit(() -> {
        System.out.println("Task 2");
    });
}
```




 
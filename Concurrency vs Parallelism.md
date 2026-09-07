## Concurrency vs Parallelism:

- Concurrency = dealing with multiple tasks
- Parallelism = doing multiple tasks at the exact same time

1. Concurrency:
    - In concurrency, multiple tasks make progress during the same period of time, but they may not be running simultaneously.
    - A single CPU core can achieve concurrency through context switching.

# Example:
```java
Thread t1 = new Thread(() -> {
    for(int i=1; i<=5; i++) {
        System.out.println("Task A");
    }
});

Thread t2 = new Thread(() -> {
    for(int i=1; i<=5; i++) {
        System.out.println("Task B");
    }
});

Posible outpur: (The JVM switches between threads.)
Task A
Task B
Task A
Task A
Task B
Task B
```

2. Parallelism
    - In parallelism, tasks execute literally at the same time.
    - This requires multiple CPU cores.

## Example: (Threads are executing at same time)
```java
Core 1 --> Thread A
Core 2 --> Thread B
Core 3 --> Thread C
Core 4 --> Thread D
```
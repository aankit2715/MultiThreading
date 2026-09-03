## Process:
A program under execution knows as process.
- Each process has its own address space, stack, and heap.
- Process are isolating from each other. Means process A is runnning Task A then process B can't access Task A directly. So isolation procvides security and stability.
- Communication between processes requires Inter-Process Communication (IPC) mechanisms like sockets or shared memory
- Crash doesn’t affect others.

## Thread:
A smallest unit of execution inside a process knows as Thread.
- Threads exist within a same process share its memory and resources.
- Less isolation, risk of race conditions.
- Crash may affect parent process

## - When we open an application, the opertaing system creates a process for it.
- Modern applications have multiple processes and threads at same time.
Ex: Google Crome:
Chrome Main Process
|
+--- Tab Process (Gmail)
|     |
|     +--- Rendering Thread
|     +--- JavaScript Thread
|
+--- Tab Process (Youtube)
|     |
|     +--- Rendering Thread
|     +--- JavaScript Thread

- So for each tab of crome os creates a process so that if any of the tab got crash it doesn't imapct on browser.

## Context Swicthing:
CPU can execute only one instruction per core at a time so it switches btw threads or processes.

- Process Context Switch: Switching btw processes requires,
1. Save Process A state
2. Load Process B state
3. Change memory mapping
4. Change resourses 
- Expensive Operation

- Thread Context Switch: Switching btw threads of the same process requires,
1. Save Thread 1 state
2. Load Thread 2 state 
- Memory is already shared (Mush faster)

- Program counter, Registers and Stack pointer are the pieces of information the CPU needs to know where a thread was interrupted and how to resume it later.
- Program counter: Stores the address of next instruction to execute.
- Registers: Are tine memory inside the CPU.
- Stack Pointer: Each thread has its own stack and stack pointer contains the current top of the stack is here.

## Ways to create Threads:

1. Extending Thread Class
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


2. Implementing Runnable Interface:
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

- Advantage: 
1. We can still extends another class.
2. Multiple threads can execute the same task object.
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
```
- But in above example there is posiblity of overriding as multiple threads are manipulating accountBalance or same data so they can override each other changes and that's called RACE Condition..

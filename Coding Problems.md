# 1. Reentrant Lock:
- Producer waits on notFull when the buffer is full.
- Consumer waits on notEmpty when the buffer is empty.
- Producer signals notEmpty after producing.
- Consumer signals notFull after consuming.

```java
package com.test;

import java.util.LinkedList;
import java.util.Queue;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

class SharedBuffer {
	
	private final Queue<Integer> buffer = new LinkedList<>();
	private final int capacity;
	
	private final ReentrantLock lock = new ReentrantLock();
	private final Condition notFull = lock.newCondition();
	private final Condition notEmpty = lock.newCondition();
	
	public SharedBuffer(int cap) {
		this.capacity = cap;
	}
	
	public void produce(int item)throws InterruptedException {
		lock.lock();
		try {
			while(buffer.size() == capacity)
				notFull.await();
			
			buffer.offer(item);  // Add data into end/tail of queue
			System.out.println(item + " Data Produced by " + Thread.currentThread().getName());
			notEmpty.signalAll();
		} finally {
			lock.unlock();
		}
	}
	
	public int consume() throws InterruptedException {
		lock.lock();
		try {
			while(buffer.isEmpty())
				notEmpty.await();
			
			int data = buffer.poll(); 	// Extract data from head/front of queue
			System.out.println(data + " Data Consumed by " + Thread.currentThread().getName());
			notFull.signal();
			return data;
		} finally {
			lock.unlock();
		}
	}
}

public class ReentrantLockExample {
	
	private static final int POSION_PILL = -1;
	private static final int CONSUMER_COUNT = 3;
	
	public static void main(String[] args) throws Exception{
		
		SharedBuffer sb = new SharedBuffer(10);
		
		ExecutorService executor = Executors.newFixedThreadPool(5);
		
		// Here 2 means I am waiting for 2 events/tasks to complete.
		CountDownLatch producerLatch = new CountDownLatch(2);
		
		// Producer
		executor.submit(() -> producer(1, 5, sb, producerLatch));
		executor.submit(()-> producer(6, 10, sb, producerLatch));
			 
		// Consumer
		executor.submit(() -> consumer(sb));
		executor.submit(() -> consumer(sb));
		executor.submit(() -> consumer(sb));
		
		// Main thread waits for producers
		producerLatch.await();
		System.out.println("All Producer threads are finished");
		
		// Send poison pill to each consumer
		for(int i=0; i<CONSUMER_COUNT; i++)
			sb.produce(POSION_PILL);
		 
		
		executor.shutdown();
		
	}
	
	public static void producer(int start, int end, SharedBuffer sb, CountDownLatch latch) {
		try {
			for(int i=start; i<=end; i++) {
				sb.produce(i);
				Thread.sleep(1000);
			}			 			
		} catch(Exception e) {
			System.out.println("Error while Producing data");
		} finally {
			latch.countDown();
		}
		
	}
	
	public static void consumer(SharedBuffer sb) {
		try {
			while(true) {
				int data = sb.consume();
				if(data == POSION_PILL) {
					System.out.println("Stopped Thread is " + Thread.currentThread().getName());
					break;
				}				 
			}			 
		} catch(Exception e) {
			System.out.println("Error while consuming data");
		}
	}

}

```


# 2. Blocking Queue:

```java
package com.test;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.LinkedBlockingQueue;

public class BlockingQueueExample {
	
	private static BlockingQueue<Integer> bq = new LinkedBlockingQueue<>(5); 
	
	public static void main(String[] args) {
		
		ExecutorService executor = Executors.newFixedThreadPool(4);
		
		//Producer
		executor.submit(() -> {	
			for(int i=1; i<=10; i++) {
				try {
					bq.put(i);
					System.out.println("Data Produced " + i + " by " + Thread.currentThread().getName());
					Thread.sleep(2000);
				}catch(Exception e) {
					System.out.println("Error while processing producer "+ Thread.currentThread().getName());
				}
			}
			
		});
		
		//Consumer 1
		executor.submit(() -> Consumer());
		executor.submit(() -> Consumer());
		executor.submit(() -> Consumer());
	}
	
	public static void Consumer() {
		try {
			while(true) {
				Integer data = bq.take();
				System.out.println("Data Consumed " + data + " by " + Thread.currentThread().getName());
				Thread.sleep(1000);
			}			 
		} catch(Exception e) {
			System.out.println("Error while processing consumer "+ Thread.currentThread().getName());
		}
	}

}
```

# 3. Wait() and Notify():

```java
package com.test;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

class Buffer {
	
	private int data;
	private boolean flag = false; //  false: Empty and true: Data
	
	public synchronized void produce(int value) throws Exception {
		
		while(flag)
			wait();
		
		data = value;
		System.out.println("Produced Data: " + value + " by " + Thread.currentThread().getName());
		flag = true;
		notifyAll();			
	}
	
	public synchronized void consume() throws Exception {
		
		while(!flag)
			wait();
		
		System.out.println("Consumed Data: " + data + " by " + Thread.currentThread().getName());
		flag = false;
		notify();
	}
	
}


public class NotifyAllExample {

     public static void main(String[] args) throws Exception{
    	 
    	 Buffer bf = new Buffer();
    	 
    	 ExecutorService executor = Executors.newFixedThreadPool(5);
    	 
    	 //Producer 1
    	 executor.submit(() -> {
    		 
    		 for(int i=1; i<=10; i++) {
    			 try {
					bf.produce(i);
					Thread.sleep(2000);
				} catch (Exception e) {
					// TODO Auto-generated catch block
					System.out.println("Error while processing producer " + Thread.currentThread().getName());
				}
    		 }
 				 
    	 });
    	 
 
    	 // Consumer 1
    	 executor.submit(()-> consumer(bf));
    	 executor.submit(()-> consumer(bf));
    	 executor.submit(() -> consumer(bf));
    	 executor.submit(() -> consumer(bf));
    	 
    	 executor.shutdown();
    	 
    	 
     }
     
     public static void consumer(Buffer buffer) {
    	 
    	 try {
    		 while(true) {
    			 buffer.consume();
    			 Thread.sleep(1000);
    		 }
    	 } catch(Exception e) {
    		 System.out.println("Error while processing consumer " + Thread.currentThread().getName());
    	 }
    	 
     }
}
```

# 4. Print numbers in order using N threads:

```java
package com.test;

import java.time.LocalTime;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicInteger;

class AlternativeNumbers {
	
	private int i=1;
	private final int limit;
	
	public AlternativeNumbers(int limit) {
		this.limit = limit;
	}
	
	public synchronized void evenNumber() throws InterruptedException {
		try {
			while(i <= limit) {
				
				while(i % 2 != 0)
					wait();
				
				if(i > limit) {
					notifyAll();
					break;
				}
				
				System.out.println(Thread.currentThread().getName() +" : " + i++);
				notifyAll();
			}
		} catch(InterruptedException e) {
			Thread.currentThread().interrupt();
		}
		 
	}
	
	public synchronized void oddNumber() throws InterruptedException {
		
		try {
			while(i <= limit) {
				
				while(i % 2 == 0)
					wait();
				
				if(i > limit) {
					notifyAll();
					break;
				}
				
				System.out.println(Thread.currentThread().getName() + " : " + i++);
				notifyAll();
			}
		} catch(InterruptedException e) {
			Thread.currentThread().interrupt();
		}
		
		 
	}
}
 

public class TestThread {
	 
	public static void main(String[] args) throws InterruptedException {
		
		AlternativeNumbers AltNum = new AlternativeNumbers(20);
		
		ExecutorService executor = Executors.newFixedThreadPool(2);
		
		executor.submit(()-> {
			try {
				AltNum.oddNumber();
			} catch(InterruptedException e) {}		 
		});
		
		executor.submit(()-> {
			try {
				AltNum.evenNumber();
			} catch(InterruptedException e) {}		 
		});		 
		
		executor.shutdown();
		 	 
	}

}

```

# 5. Race Condition:

```java
package com.test;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class RaceConditionExample {
	
	private static int counter=0;
	
	public static void main(String[] args) throws InterruptedException{
		
		ExecutorService executor = Executors.newFixedThreadPool(4);
		
		executor.submit(() -> increment());
		executor.submit(() -> increment());
		executor.submit(() -> increment());
		executor.submit(() -> increment());
		
		// Don't accept any new tasks, but let the already submitted tasks finish.
		executor.shutdown();
		
		// Main thread, wait up to 1 minute for all executor tasks to complete.
		executor.awaitTermination(1, TimeUnit.MINUTES);
		
		System.out.println("Final Counter: " + counter);
	}
	
	public static void increment() {	
		for(int i=0; i<20000; i++)
			counter++;	
	}

}
```

# 6. How to make Singleton threads safe?
- Singleton: A Singleton is a design pattern that ensures:
	1. Only one instance of a class is created.
	2. That instance is globally accessible throughout the application.
	
- Why use Singleton?
Common use cases:
	1. Logger
	2. Configuration manager
	3. Cache manager
	4. Database connection pool manager
	5. Application-wide settings
	
- Example:
```java
public class Singleton {

    private static Singleton instance;

    private Singleton() {
        // private constructor prevents object creation using new
    }

    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

- Why is the above Singleton not thread-safe?  
Imagine two threads, Thread A and Thread B, call getInstance() at exactly the same time.

**Initially:**  
instance = null  

**Thread A executes:**  
if(instance == null)  

Condition is true. Before Thread A creates the object, CPU switches to Thread B.

**Thread B executes:**  
if(instance == null)  

Condition is still true. Now Thread B creates an object;  
instance = new Singleton();  

CPU switches back to Thread A. Thread A also creates another object:  
instance = new Singleton();  

**Result:**  
Singleton object 1  
Singleton object 2  
Two different objects are created, which violates the Singleton principle.  

- How to make it thread-safe?
Example:
```java
package com.test;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

class Singleton {
	
	private static Singleton instance;
	
	private Singleton() {
		System.out.println("Singleton Instance Created by " + Thread.currentThread().getName());
	}
	
	public static Singleton getInstance() throws InterruptedException {
		if(instance == null) {
			Thread.sleep(2000);
			instance = new Singleton();
		}  
		
		return instance;
	}
}

public class SingletonExample {
	
	public static void main(String[] args) throws Exception{
		
		ExecutorService executor = Executors.newFixedThreadPool(3);
		
		for(int i=0; i<3; i++) {
				executor.submit(() -> {
					try {
						Singleton singleton = Singleton.getInstance();
						System.out.println(Thread.currentThread().getName() + " : " + singleton.hashCode());
					} catch(Exception e) {}	 
					
				});				
		}
		
		executor.shutdown();		 
	}
}

Posible Output:
Singleton Instance Created by pool-1-thread-3
pool-1-thread-3 : 68884669
Singleton Instance Created by pool-1-thread-2
pool-1-thread-2 : 1739213138
Singleton Instance Created by pool-1-thread-1
pool-1-thread-1 : 2128527197
```  
In above example, we just need to make getInstance() thread safe and will able to achive threads safe singleton pattern.

# 7.
# 8.
# 9.


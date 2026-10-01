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

# 7. Rate Limiter with Token Bucket:
- **Requirements**
	- Maximum N requests allowed.
	- Tokens refill at a fixed rate.
	- Multiple threads can call allowRequest() concurrently.
	- No race conditions.
	
- **capacity** → Maximum tokens bucket can hold.
- **refillRatePerSecond** → Number of tokens added every second.
- **tokens** → Current available tokens.
- **lastRefillTime** → Timestamp used to calculate how many new tokens should be added since the previous refill.

```java
package com.test;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.locks.ReentrantLock;

class TokenBucketRateLimiter {
	
	
	private final long capacity;
	private final long refillRatePerSec;
	
	private double token;
	private long lastRefillTime;
	
	private final ReentrantLock lock = new ReentrantLock();
	
	public TokenBucketRateLimiter(long capacity, long refillRate) {
		this.capacity = capacity;
		this.refillRatePerSec = refillRate;
		this.token = capacity;
		this.lastRefillTime = System.nanoTime();
		
	}
	
	public boolean allowRequest() throws InterruptedException {
		lock.lock();
		try {
			Thread.sleep(200);
			refill();
			if(token >= 1) {
				token--;
				return true;
			}
		} finally {
			lock.unlock();
		}
		
		return false;
	}
	
	public void refill() {
		
		long now = System.nanoTime();		
		long timePassed = (now - lastRefillTime)/1000000000;
		
		long newToken = timePassed * refillRatePerSec;
		
		if(newToken > 0) {
			token = Math.min(capacity, token+newToken);
			lastRefillTime = now;
		}
	}
	
}

public class RateLimiterExample {
	
	public static void main(String[] args) {
		
		TokenBucketRateLimiter tbr = new TokenBucketRateLimiter(10, 2);
		
		ExecutorService executor = Executors.newFixedThreadPool(5);
		
		for(int i=0; i<5; i++) {
			executor.submit(()-> {
				while(true) {
					try {
						if(tbr.allowRequest())
							System.out.println("Request allowed of " + Thread.currentThread().getName());
						else
							System.out.println("Request not allowed of " + Thread.currentThread().getName());
					} catch(Exception e) {}
				}		 
			});
		}
		
		executor.shutdown();
		 
	}

}

Posible Output:
Request allowed of pool-1-thread-3
Request allowed of pool-1-thread-5
Request not allowed of pool-1-thread-1
Request not allowed of pool-1-thread-2
Request not allowed of pool-1-thread-4
Request allowed of pool-1-thread-3
Request allowed of pool-1-thread-5
Request not allowed of pool-1-thread-1
Request not allowed of pool-1-thread-2
Request not allowed of pool-1-thread-4
Request allowed of pool-1-thread-3
			|
			|
			|
			|
			|
			
```

# 8. User-Based Token Bucket Rate Limiter:
- ConcurrentHashMap for per-user buckets
- ReentrantLock for thread safety
- ExecutorService for concurrent requests

```java
package com.test;

import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.locks.ReentrantLock;

class TokenBucket {
	
	private final long capacity;
	private final long refillRatePerSec;
	private long token;
	private long lastRefillTime;
	
	private final ReentrantLock lock = new ReentrantLock();
	
	TokenBucket(long cap, long refillRate) {
		this.capacity = cap;
		this.refillRatePerSec = refillRate;
		this.token = cap;
		this.lastRefillTime = System.nanoTime();
	}
	
	public boolean allowRequest() throws InterruptedException {	
		lock.lock();
		try {
			refillToken();
			if(token>0) {
				token--;
				return true;
			}
			
		} finally {
			lock.unlock();
		}
		
		return false;
	}
	
	public void refillToken() {
		
		long now = System.nanoTime();
		long passedTime = (now - lastRefillTime) / 1000000000;
		
		long newToken = passedTime * refillRatePerSec;
		
		if(newToken > 0) {
			token = Math.min(capacity, token+newToken);
			lastRefillTime = System.nanoTime();
			
		}	
	}
		
}

class UserRateLimiter {
	
	private final long capacity;
	private final long refillRate;
	
	private final ConcurrentHashMap<String, TokenBucket> userBucket = new ConcurrentHashMap<>();
	
	public UserRateLimiter(long cap, long refillRate) {
		this.capacity =  cap;
		this.refillRate = refillRate;
	}
	
	public boolean allowRequest(String userId) throws InterruptedException {
		
		TokenBucket bucket = userBucket.computeIfAbsent(userId, id -> new TokenBucket(capacity, refillRate));
		return bucket.allowRequest();
	}
	
}

public class UserRateLimiterExample {
	
	public static void main(String[] args) {
		
		UserRateLimiter limiter = new UserRateLimiter(3, 2);
		
		String[] users = {"user1", "user2", "user3"};
		
		ExecutorService executor = Executors.newFixedThreadPool(5);
		
		for(int i=0; i<30; i++) {
			
			final String userId = users[i%users.length];
			executor.submit(()-> {
				try {	
					boolean allowed = limiter.allowRequest(userId);
					System.out.println(Thread.currentThread().getName()+ " | " + userId + " | " + (allowed ? "Allowed" : "Rejected"));
					Thread.sleep(300);
					 
				} catch (InterruptedException e) {
	                Thread.currentThread().interrupt();
	            } 
				
			});
		}	 
		
		executor.shutdown();
	}

}

**Posible Output:**

pool-1-thread-3 | user3 | Allowed
pool-1-thread-1 | user1 | Allowed
pool-1-thread-2 | user2 | Allowed
pool-1-thread-4 | user1 | Allowed
pool-1-thread-5 | user2 | Allowed
pool-1-thread-2 | user3 | Allowed
pool-1-thread-3 | user1 | Allowed
pool-1-thread-1 | user2 | Allowed
pool-1-thread-4 | user3 | Allowed
pool-1-thread-5 | user1 | Rejected
pool-1-thread-2 | user3 | Rejected
pool-1-thread-3 | user2 | Rejected
pool-1-thread-1 | user1 | Rejected
pool-1-thread-4 | user2 | Rejected
pool-1-thread-5 | user3 | Rejected
pool-1-thread-2 | user1 | Rejected
pool-1-thread-4 | user3 | Rejected
pool-1-thread-1 | user1 | Rejected
pool-1-thread-3 | user2 | Rejected
pool-1-thread-5 | user2 | Rejected
pool-1-thread-2 | user3 | Allowed
pool-1-thread-4 | user1 | Allowed
pool-1-thread-1 | user3 | Allowed
pool-1-thread-3 | user2 | Allowed
pool-1-thread-5 | user1 | Allowed
pool-1-thread-2 | user2 | Allowed
pool-1-thread-4 | user3 | Rejected
pool-1-thread-3 | user1 | Rejected
pool-1-thread-1 | user2 | Rejected
pool-1-thread-5 | user3 | Rejected
```

# 9. Dinning Philoshper Problem:
The Dining Philosophers Problem is one of the most famous synchronization problems in Operating Systems and Java multithreading interviews.  

It demonstrates how multiple threads compete for limited resources and how improper locking can lead to deadlock, starvation, and resource contention.  

- **Problem Statement**  
    - 5 philosophers are sitting around a circular dining table.
    - Between every pair of philosophers there is one fork.
    - Therefore there are 5 philosophers and 5 forks.

- **Each philosopher repeatedly:**  
    - Thinks
    - Gets hungry
    - Picks up left fork
    - Picks up right fork
    - Eats
    - Puts down both forks

- **Constraint**  
A philosopher needs both forks to eat.

**Solution:**  
Here's a complete Dining Philosophers implementation using ExecutorService and ReentrantLock.tryLock().  

This version avoids deadlock because a philosopher only eats if both forks are acquired; otherwise, it releases any acquired fork and tries again later.

```java
package com.test;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.locks.ReentrantLock;

class Philosopher {
	
	private int id;
	private ReentrantLock leftFork;
	private ReentrantLock rightFork;
	
	Philosopher(int id, ReentrantLock left, ReentrantLock right) {
		this.id = id;
		this.leftFork = left;
		this.rightFork = right;
	}
	
	public void dine() throws InterruptedException {
		
		try {
			
			while(!Thread.currentThread().isInterrupted()) { //Keep running until someone interrupts this thread.
				
				think();
				
				if(leftFork.tryLock()) { // Unlike: leftFork.lock(); --> which waits forever, leftFork.tryLock(); --> returns immediately:	
					
					try {
						System.out.println("Philosopher " + id + " is picked left fork !!!");	
						
						if(rightFork.tryLock()) {
							
							try {
								System.out.println("Philosopher " + id + " is picked right fork !!!");
								eat();
							} finally {
								rightFork.unlock();
								System.out.println("Philosopher " + id + " is released right fork!!!");
							}
						}
					} finally {
						leftFork.unlock();
						System.out.println("Philosopher " + id + " is released left fork !!!");
					}
				
				}
			}
			Thread.sleep(100);
		} catch(InterruptedException e) {
			Thread.currentThread().interrupt();
		}
			 	
		
	}
	
	public void think() throws InterruptedException {	
		System.out.println("Philosopher "+ id + " is thinking !!!");
		Thread.sleep(1000);	
	}
	
	public void eat() throws InterruptedException{
		System.out.println("Philosopher " + id + " is eating !!!");
		Thread.sleep(500);
	}
}

public class DinningPhilosopher{
	
	public static void main(String[] args) throws InterruptedException{
		
		int philosopherCount = 5;
		ReentrantLock[] fork = new ReentrantLock[philosopherCount];
		
		for(int i=0; i<philosopherCount; i++)
			fork[i] = new ReentrantLock();
		
		ExecutorService executor = Executors.newFixedThreadPool(philosopherCount);
		
		for(int i=0; i<philosopherCount; i++) {
			
			Philosopher philosopher = new Philosopher(i, fork[i], fork[(i+1)%philosopherCount]);
			
			executor.submit(() -> {
				try {
					philosopher.dine();
				} catch (InterruptedException e) {
					// TODO Auto-generated catch block
					e.printStackTrace();
				}
			});		
			
		}
		
		Thread.sleep(10000);	 
		executor.shutdownNow();
		
		System.out.println("Dinner is over");
	}

}

**Posible Output:**

Philosopher 0 is thinking !!!
Philosopher 1 is thinking !!!
Philosopher 4 is thinking !!!
Philosopher 2 is thinking !!!
Philosopher 3 is thinking !!!
Philosopher 1 is picked left fork !!!
Philosopher 4 is picked left fork !!!
Philosopher 0 is picked left fork !!!
Philosopher 2 is picked left fork !!!
Philosopher 0 is picked right fork !!!
Philosopher 4 is released left fork !!!
Philosopher 1 is released left fork !!!
Philosopher 4 is thinking !!!
Philosopher 0 is eating !!!
Philosopher 2 is picked right fork !!!
Philosopher 1 is thinking !!!
Philosopher 2 is eating !!!
Philosopher 3 is thinking !!!
Philosopher 0 is released right fork!!!
Philosopher 0 is released left fork !!!
Philosopher 2 is released right fork!!!
Philosopher 0 is thinking !!!
Philosopher 2 is released left fork !!!
Philosopher 2 is thinking !!!
Dinner is over
```

# 10.  Custom Thread Pool:

```java
package com.test;

import java.util.LinkedList;
import java.util.Queue;

class CustomThreadPool {
	
	private final WorkerThread[] workers;
	private final Queue<Runnable> taskQueue;
	
	private volatile boolean isShutdown = false;  // volatile keyword ensures that changes made to a variable by one thread are immediately visible to all other threads.
	
	public CustomThreadPool(int poolSize) {
		
		workers = new WorkerThread[poolSize];
		taskQueue = new LinkedList<>();
		
		for(int i=0; i<poolSize; i++) {
			workers[i] = new WorkerThread();
			workers[i].start();
		}
	}
	
	public void submit(Runnable task) {		
		synchronized(taskQueue) {
			if(isShutdown) {
				throw new IllegalStateException("Thread pool is shutdown");
			}
			taskQueue.offer(task);
			taskQueue.notifyAll();
		}
		
	}
	
	public void shutdown() {
		synchronized(taskQueue) {
			isShutdown = true;	
			taskQueue.notifyAll();
		}	 
	}
	
	public class WorkerThread extends Thread {
		
		@Override
		public void run() {
			
			while(true) {				
				Runnable task;
				synchronized(taskQueue) {
					while(taskQueue.isEmpty() && !isShutdown) {
						try {
							taskQueue.wait();
						} catch (InterruptedException e) {
							 Thread.currentThread().interrupt();
							 return;
						}
					}
					if(isShutdown && taskQueue.isEmpty()) {
						System.out.println(Thread.currentThread().getName() + " stopped");
						return;
					}
					task = taskQueue.poll();
				}	 	
				try {
					task.run();
				} catch(Exception e) {
					e.printStackTrace();
				}
			}
			
		}
		
	}
	
}
 

public class ThreadPoolExample {
	
  public static void main(String[] args) throws InterruptedException {
	  
	  CustomThreadPool pool = new CustomThreadPool(3);
	  
	  for(int i=1; i<=10; i++) {
		  int taskId = i;
		  pool.submit(()-> {
			  System.out.println(Thread.currentThread().getName() + " Executed task " + taskId);
			  try {
				  Thread.sleep(500);
			  } catch(InterruptedException e) {
				  Thread.currentThread().interrupt();
			  }
		  });
	  }
	  
	  Thread.sleep(5000);
	  pool.shutdown();
	  System.out.println("Thread pool shutdown initiated");
	  
  }	 

}
```

# 11. Read-Write Lock Implementation

```java
package com.test;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

class ReaderWriterLock {
	
	private int reader = 0;
	private int writer = 0;
	private int writeRequest = 0;
	
	public synchronized void lockRead() throws InterruptedException {
		while(writer > 0 || writeRequest > 0)
			wait();
		
		reader++;
	}
	
	public synchronized void unlockRead() {
		reader--;
		notifyAll();
	}
	
	public synchronized void lockWrite() throws InterruptedException {
		writeRequest++;
		while(writer > 0 || reader > 0)
			wait();
		
		writeRequest--;
		writer++;
	}
	
	public synchronized void unlockWrite() {
		writer--;
		notifyAll();
	}
	
}

class ManipulatingSection {
	
	private int data = 0;	
	ReaderWriterLock lock = new ReaderWriterLock();
	
	public void read() {
		boolean locked = false;
		try {
			lock.lockRead();
			locked = true;
			System.out.println(Thread.currentThread().getName() + " Reading Value: " + data);
			Thread.sleep(500);
		} catch (InterruptedException e) {
			Thread.currentThread().interrupt();
		} finally {
			if(locked)
				lock.unlockRead();
		}
	}
	
	public void write(int val) {
		boolean locked = false;
		try {
			lock.lockWrite();
			locked = true;
			System.out.println(Thread.currentThread().getName() + " Writing Value: " + val);
			data = val;
			Thread.sleep(1000);
		} catch (InterruptedException e) {
			Thread.currentThread().interrupt();
		} finally {
			if(locked)
				lock.unlockWrite();
		}
	}
	
}

class ReaderWriterExample {
	
public static void main(String[] args) throws InterruptedException {
		
	ManipulatingSection resource = new ManipulatingSection();
		
		ExecutorService executor = Executors.newFixedThreadPool(4);
		
		for(int i=0; i<3; i++) {	 
			executor.submit(()-> {
				System.out.println("Reader Thread - " + Thread.currentThread().getName());
				for(int j=1; j<=3; j++) {
					resource.read(); 		
				}
					 		 
			});
		}
		
		executor.submit(()-> {
			System.out.println("Writer Thread - " + Thread.currentThread().getName());
			for(int i=1; i<=3; i++) {
				resource.write(i*10);
			}
		});
		
		Thread.sleep(5000);
		
		executor.shutdown();
		
	}

}
```

 


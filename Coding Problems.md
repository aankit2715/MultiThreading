# Reentrant Lock:
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


# Blocking Queue:

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


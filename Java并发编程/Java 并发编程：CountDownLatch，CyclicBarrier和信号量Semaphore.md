# Java 并发编程：CountDownLatch，CyclicBarrier和信号量Semaphore



描述：CountDownLatch，CyclicBarrier和信号量Semaphore都是java并发包concurrent下提供的并发工具类，是比synchrorized关键字更高效的同步结构。

## CountDownLatch

CountDownLatch的用法顾名思义有一个计数器的概念，其应用场景是，如果B组线程要等A组线程全部执行完后再执行，那A组中每个线程都会调用其countDown方法计数，即A组中每跑一个线程计数加一。而B组中每个线程都会调用其await()方法。只有当计数达到预设值后，因调用await()方法而处于等待状态的线程全部执行。CountDownLatch可以用于高并发测试，即计数积累了一定数量的线程后再一起执行。

例子：

```java
package com.example.concurrent;

import java.util.concurrent.CountDownLatch;

/**
 * <p>
 *  CountDownLatch 使用例子
 * </p>
 *
 * @author wanxinabo
 * @date 2021/4/8
 */
public class CountDownLatchDemo {
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(6);

        for (int i = 0; i < 5; i++) {
            Thread t = new Thread(new FirstBatchWorker(countDownLatch));
            t.start();
        }

        for (int i = 0; i < 5; i++) {
            Thread t = new Thread(new SecondBatchWorker(countDownLatch));
            t.start();
        }

        while (countDownLatch.getCount() != 1) {
            Thread.sleep(100L);
        }

        System.out.println("Wait for first batch finish");
        countDownLatch.countDown();
    }
}

class FirstBatchWorker implements Runnable {

    private CountDownLatch countDownLatch;

    public FirstBatchWorker(CountDownLatch countDownLatch) {
        this.countDownLatch = countDownLatch;
    }

    @Override
    public void run() {
        System.out.println("First Batch Executed!");
        countDownLatch.countDown();
    }
}

class SecondBatchWorker implements Runnable {
    private CountDownLatch countDownLatch;

    public SecondBatchWorker(CountDownLatch countDownLatch) {
        this.countDownLatch = countDownLatch;
    }

    @Override
    public void run() {
        try {
            countDownLatch.await();
            System.out.println("Second Batch Executed!");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

## CyclicBarrier

CyclicBarrier和CountDownLatch有点像，不过他不是计数。CyclicBarrier在初始化的时候可以定义一个参与线程的数量，即parties同伴数量，有点抽象，接下来线程中会先调用一个await()方法使线程处于等待状态。只有当有预设值parties个线程调用了await()方法后，那这parties个线程才能继续执行。简单点说就是凑齐了一拨人一起走。这里就有一个generation概念，即同一代线程，也就是前面说的同一拨人。值得注意的是当同一代线程一起执行后，generation会被自动重置，开始下一代的互相await()和一起执行。这也是和CountDownLatch区别最大的地方，CountDownLatch不会自己重置。

CyclicBarrier的await()方法底层是通过再入锁RecentLock实现的，简单点说，每个调用await()方法的线程会先lock()获得再入锁，最后必须unlock()释放锁以供其他同伴线程获得，同伴全部都获得再入锁然后一起执行的时候，不需要再次获得锁了。让同伴线程一起执行是通过一个notifyAll()方法一起唤醒的。

例子：

```java
package com.example.concurrent;

import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

/**
 * <p>
 *  CyclicBarrier 的例子
 * </p>
 *
 * @author wanxinabo
 * @date 2021/4/8
 */
public class CyclicBarrierDemo {

    public static void main(String[] args) {
        final CyclicBarrier cyclicBarrier = new CyclicBarrier(5, () -> {
            System.out.println("Action... GO again!");
        });

        for (int i = 0; i < 5; i++) {
            Thread t = new Thread(new CyclicWorker(cyclicBarrier), String.valueOf(i));
            t.start();
        }
    }

    static class CyclicWorker implements Runnable {
        private CyclicBarrier cyclicBarrier;

        public CyclicWorker(CyclicBarrier cyclicBarrier) {
            this.cyclicBarrier = cyclicBarrier;
        }

        @Override
        public void run() {
            try {
                for (int i = 0; i < 3; i++) {
                    System.out.println(Thread.currentThread().getName() + "Executed!");
                    cyclicBarrier.await();
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
        }
    }
}
```

## Semaphore

信号量Semaphore用得比较多，核心作用是控制并发执行的线程数量。定义信号量的时候会初始化一个预设值n，即最多有n个线程同时执行，如果将n设为1，就达到了互斥锁的效果。

信号量是通过acquire()和release()来控制线程的。最多允许n个线程同时执行即有n个通行证，线程会先acquire()即去获得许可证，只有成功获得许可证才能继续执行，否则处于等待状态，最后通过release()把许可证释放掉以供其他线程获取，通过这种方式去控制线程最大并发量。

acquire()和release()都可以传参，acquire(n)/release(n)是获取/释放n个许可证，如果不传参即默认获取/释放1个许可证。有时候最大并发量需要计算一下，举个例子，如果预设值10个许可证，线程acquire(3)就是一次获取3个许可证才能执行，那么最大并发量就是10/3=3个。信号量还有一个availablepermiles()方法顾名思义就是返回可用许可证的数量。

例子：

```java
package com.example.concurrent;

import com.google.common.base.Strings;

import java.util.concurrent.Semaphore;

/**
 * <p>
 *  Semaphore 的例子
 * </p>
 *
 * @author wanxinabo
 * @date 2021/4/8
 */
public class SemaphoreDemo {
    public static void main(String[] args) {
        System.out.println("Action... GO!");
        Semaphore semaphore = new Semaphore(5);

        for (int i = 0; i < 3; i++) {
            new Thread(new SemaphoreWorker(semaphore), String.valueOf(i)).start();
        }
    }

}

class SemaphoreWorker implements Runnable {
    private String name;
    private Semaphore semaphore;

    public SemaphoreWorker(Semaphore semaphore) {
        this.semaphore = semaphore;
    }

    @Override
    public void run() {
        try {
            log("is waiting for a permit");
            semaphore.acquire(2);
            log("acquire a permit");
            log("executed!");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            log("release a permit");
            semaphore.release(2);
        }

    }

    private void log(String msg) {
        if (Strings.isNullOrEmpty(msg)) {
            name = Thread.currentThread().getName();
        }

        System.out.println(name + "\r" + msg);
    }
}
```


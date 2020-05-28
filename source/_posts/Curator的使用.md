---
title: Curator的使用（TODO:源码、原理）
date: 2019-02-13 13:55:19
tags: [zookeeper, 分布式锁]
categories: 分布式锁
---
#### Curator使用 Demo

代码
```java
public class CuratorDemo {

    public static void main(String[] args) throws Exception {

        //重试机制
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
        //客户端
        CuratorFramework client = CuratorFrameworkFactory.newClient("localhost:2181", retryPolicy);
        client.start();

        //线程池
        ThreadPoolExecutor threadPoolExecutor =
                new ThreadPoolExecutor(Runtime.getRuntime().availableProcessors() * 2, 10, 10,
                        TimeUnit.SECONDS, new LinkedBlockingQueue<>(1000),
                        //com.google.guava的ThreadFactoryBuilder
                        new ThreadFactoryBuilder().setNameFormat("test-pool-%d").build());
                        
        for (int i = 0; i < 5; i++) {
            threadPoolExecutor.execute(() -> {
                //分布式可重入排它锁
                InterProcessMutex mutex = new InterProcessMutex(client, "/test");
                try {
                    String threadName = Thread.currentThread().getName();
                    System.out.println("创建线程："+threadName);

                    mutex.acquire();
                    System.out.println(threadName + " 加锁成功！");
                    Thread.sleep(1000);

                    mutex.acquire();
                    System.out.println(threadName + " 重入锁成功！");
                    Thread.sleep(1000);

                    mutex.release();
                    mutex.release();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            });
        }

        threadPoolExecutor.shutdown();
        if (threadPoolExecutor.isTerminated()) {
            client.close();
        }
    }
}
```
输出
```java
创建线程：test-pool-1
创建线程：test-pool-2
创建线程：test-pool-0
创建线程：test-pool-3
创建线程：test-pool-4
test-pool-0 加锁成功！
test-pool-0 重入锁成功！
test-pool-4 加锁成功！
test-pool-4 重入锁成功！
test-pool-3 加锁成功！
test-pool-3 重入锁成功！
test-pool-1 加锁成功！
test-pool-1 重入锁成功！
test-pool-2 加锁成功！
test-pool-2 重入锁成功！
```

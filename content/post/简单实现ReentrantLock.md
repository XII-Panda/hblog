---
title: "简单实现ReentrantLock"
description: 
date: "2022-04-15"
lastmod: "2022-04-15"
image: 
math: 
license: 
hidden: false
comments: true
draft: false
categories: ["Java"]
tags: ["多线程"]
---

# 简单实现ReentrantLock

## 实现思路

锁的基本执行流程：

1. 抢到锁，继续执行
2. 抢不到锁，阻塞等待
3. 释放锁，唤醒等待线程抢锁

所以需要一个互斥量

释放锁时需要唤醒等待线程，所以需要一个存储等待线程的集合：

1. 使用线程安全的集合
2. 使用链表

## 代码实现

```java
public class MyReentrantLock implements Lock {
  
  /**
   * 锁的基本使用流程
   * <br>
   * 抢到锁,继续执行
   * <br>
   * 抢不到锁,阻塞等待
   * <br>
   * 释放锁,唤醒等待线程执行
   */
  private volatile int status;
  private volatile Thread ownThread;
  
  /**
   * 链表节点
   */
  private static class Node {
    Thread thread;
    volatile Node pre;
    volatile Node next;
    
    public Node(Thread thread) {
      this.thread = thread;
    }
  }
  
  /**
   * 头结点
   */
  private volatile Node head;
  /**
   * 尾结点
   */
  private volatile Node tail;
  
  private boolean fair;
  
  public MyReentrantLock() {
  }
  
  public MyReentrantLock(boolean fair) {
    this.fair = fair;
  }
  
  /**
   * 使用线程安全的List记录等待线程
   */
  private volatile CopyOnWriteArrayList<Thread> waitThreads = new CopyOnWriteArrayList<>();
  
  private static final AtomicIntegerFieldUpdater<MyReentrantLock> ATOMIC_INTEGER_FIELD_UPDATER
         = AtomicIntegerFieldUpdater.newUpdater(MyReentrantLock.class, "status");
  private static final AtomicReferenceFieldUpdater<MyReentrantLock, Node> HEAD_ATOMIC_FIELD
         = AtomicReferenceFieldUpdater.newUpdater(MyReentrantLock.class, Node.class, "head");
  private static final AtomicReferenceFieldUpdater<MyReentrantLock, Node> TAIL_ATOMIC_FIELD
         = AtomicReferenceFieldUpdater.newUpdater(MyReentrantLock.class, Node.class, "tail");
  
  @Override
  public void lock() {
    //可重入判断
    if (Thread.currentThread() == ownThread) {
      status++;
      return;
    }
    //判断是否为公平模式
    if (fair) {
      Node node = waitThread();
      doPark(node);
    } else {
      if (!tryLock()) {
        //加入等待队列
//       waitThreads.add(Thread.currentThread());
        Node node = waitThread();
        //使用循环,如果抢到锁,结束,没有继续循环阻塞等待
        doPark(node);
      }
    }
  }
  
  private void doPark(Node node) {
    while (true) {
      //如果是第一个节点,尝试获取锁
      //防止
        /*
        head=null;tail=null;
        t1 lock成功
        t1 unlock 此时head=null      t2 waitThread()
        t1 unlock return            此时t2线程不会被唤醒(除非有其他线程来抢锁)
         */
      if (node.pre == head) {
        if (tryLock()) {
          //抢到锁,从等待队列中删除该线程
          head.next = null;
          head = node;
          node.thread = null;
          node.pre = null;
          //waitThreads.remove(Thread.currentThread());
          return;
        }
      }
      LockSupport.park(this);
      //被唤醒后,清除中断状态,否则循环再次进来park不成功
      Thread.interrupted();
    }
  }
  
  
  private Node waitThread() {
    Node node = new Node(Thread.currentThread());
    while (true) {
      Node t = tail;
      //如果为空,初始化链表
      if (t == null) {
        if (HEAD_ATOMIC_FIELD.compareAndSet(this, null, new Node(null))) {
          tail = head;
          //完成了链表的初始化,重复循环,将节点加入到尾
        }
      } else {
        node.pre = t;
        if (TAIL_ATOMIC_FIELD.compareAndSet(this, t, node)) {
          t.next = node;
          return node;
        }
      }
    }
  }
  
  @Override
  public void lockInterruptibly() throws InterruptedException {
  
  }
  
  @Override
  public boolean tryLock() {
    /*
      需要一个互斥量
      通过一个int属性,CAS操作compareAndSet(0,1)充当互斥量,抢到锁,设置成功
      方式1:使用字段属性的原子类
      方式2:使用原子类AtomicInteger
     */
    //方式1
    boolean b = ATOMIC_INTEGER_FIELD_UPDATER.compareAndSet(this, 0, 1);
    //记录抢到锁的线程
    if (b) {
      ownThread = Thread.currentThread();
    }
    return b;
  }
  
  @Override
  public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
    return false;
  }
  
  @Override
  public void unlock() {
    //unlock设置status为0,从等待队列中唤醒一个等待线程抢锁
    //等待队列:1.可以使用链表 2.可以使用线程安全的集合
    if (ownThread == Thread.currentThread()) {
      int s = status - 1;
      if (s == 0) {
        ownThread = null;
      }
      status = s;
      //使用链表
      Node h = head;
      if (h != null) {
        Node n = h.next;
        if (n != null) {
          LockSupport.unpark(n.thread);
        }
      }
    }
  }
  
  @Override
  public Condition newCondition() {
    return null;
  }
}
```


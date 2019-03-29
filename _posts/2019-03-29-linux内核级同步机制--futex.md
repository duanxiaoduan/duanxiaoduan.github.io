---
layout:     post
title:      linux内核级同步机制--futex
subtitle:   同步机制
date:       2019-03-29
author:     duanxiaoduan
header-img: img/post-bg-design-linux.jpg
catalog: true
tags:
    - 多线程 底层实现
---

>linux内核级同步机制--futex

理想的同步机制应该是没有锁冲突时在用户态利用原子指令就解决问题，而需要挂起等待时再使用内核提供的系统调用进行睡眠与唤醒。换句话说，在用户态的自旋失败时，能不能让进程挂起，由持有锁的线程释放锁时将其唤醒？  
如果你没有较深入地考虑过这个问题，很可能想当然的认为类似于这样就行了（伪代码）：

    void lock(int lockval) {
    	//trylock是用户级的自旋锁
    	while(!trylock(lockval)) {
    		wait();//释放cpu，并将当期线程加入等待队列，是系统调用
    	}
    }
    
    boolean trylock(int lockval){
    	int i=0; 
    	//localval=1代表上锁成功
    	while(!compareAndSet(lockval,0,1)){
    		if(++i>10){
    			return false;
    		}
    	}
    	return true;
    }
    
    void unlock(int lockval) {
    	 compareAndSet(lockval,1,0);
    	 notify();
    }
    

上述代码的问题是trylock和wait两个调用之间存在一个窗口：  
如果一个线程trylock失败，在调用wait时持有锁的线程释放了锁，当前线程还是会调用wait进行等待，但之后就没有人再唤醒该线程了。

为了解决上述问题，linux内核引入了futex机制，futex主要包括等待和唤醒两个方法：`futex_wait`和`futex_wake`，其定义如下

    //uaddr指向一个地址，val代表这个地址期待的值，当*uaddr==val时，才会进行wait
    int futex_wait(int *uaddr, int val);
    //唤醒n个在uaddr指向的锁变量上挂起等待的进程
    int futex_wake(int *uaddr, int n);
    

futex在真正将进程挂起之前会检查addr指向的地址的值是否等于val，如果不相等则会立即返回，由用户态继续trylock。否则将当期线程插入到一个队列中去，并挂起。

在[关于同步的一点思考-上](https://github.com/farmerjohngit/myblog/issues/6)文章中对futex的背景与基本原理有介绍，对futex不熟悉的人可以先看下。

本文将深入分析futex的实现，让读者对于锁的最底层实现方式有直观认识，再结合之前的两篇文章（[关于同步的一点思考-上](https://github.com/farmerjohngit/myblog/issues/6)和[关于同步的一点思考-下](https://github.com/farmerjohngit/myblog/issues/7)）能对操作系统的同步机制有个全面的理解。

**下文中的进程一词包括常规进程与线程**。

### futex_wait

在看下面的源码分析前，先思考一个问题：如何确保挂起进程时，val的值是没有被其他进程修改过的？

代码在kernel/futex.c中

    static int futex_wait(u32 __user *uaddr, int fshared,
    		      u32 val, ktime_t *abs_time, u32 bitset, int clockrt)
    {
    	struct hrtimer_sleeper timeout, *to = NULL;
    	struct restart_block *restart;
    	struct futex_hash_bucket *hb;
    	struct futex_q q;
    	int ret;
    
    	...
    
    	//设置hrtimer定时任务：在一定时间(abs_time)后，如果进程还没被唤醒则唤醒wait的进程
    	if (abs_time) {
    	    ...
    		hrtimer_init_sleeper(to, current);
    		...
    	}
    
    retry:
    	//该函数中判断uaddr指向的值是否等于val，以及一些初始化操作
    	ret = futex_wait_setup(uaddr, val, fshared, &q, &hb);
    	//如果val发生了改变，则直接返回
    	if (ret)
    		goto out;
    
    	//将当前进程状态改为TASK_INTERRUPTIBLE，并插入到futex等待队列，然后重新调度。
    	futex_wait_queue_me(hb, &q, to);
    
    	/* If we were woken (and unqueued), we succeeded, whatever. */
    	ret = 0;
    	//如果unqueue_me成功，则说明是超时触发（因为futex_wake唤醒时，会将该进程移出等待队列，所以这里会失败）
    	if (!unqueue_me(&q))
    		goto out_put_key;
    	ret = -ETIMEDOUT;
    	if (to && !to->task)
    		goto out_put_key;
    
    	/*
    	 * We expect signal_pending(current), but we might be the
    	 * victim of a spurious wakeup as well.
    	 */
    	if (!signal_pending(current)) {
    		put_futex_key(fshared, &q.key);
    		goto retry;
    	}
    
    	ret = -ERESTARTSYS;
    	if (!abs_time)
    		goto out_put_key;
    
    	...
    
    out_put_key:
    	put_futex_key(fshared, &q.key);
    out:
    	if (to) {
    		//取消定时任务
    		hrtimer_cancel(&to->timer);
    		destroy_hrtimer_on_stack(&to->timer);
    	}
    	return ret;
    }
    

在将进程阻塞前会将当期进程插入到一个等待队列中，需要注意的是这里说的等待队列其实是一个类似Java HashMap的结构，全局唯一。

    struct futex_hash_bucket {
    	spinlock_t lock;
    	//双向链表
    	struct plist_head chain;
    };
    
    static struct futex_hash_bucket futex_queues[1<<FUTEX_HASHBITS];
    

着重看`futex_wait_setup`和两个函数`futex_wait_queue_me`

    static int futex_wait_setup(u32 __user *uaddr, u32 val, int fshared,
    			   struct futex_q *q, struct futex_hash_bucket **hb)
    {
    	u32 uval;
    	int ret;
    retry:
    	q->key = FUTEX_KEY_INIT;
    	//初始化futex_q
    	ret = get_futex_key(uaddr, fshared, &q->key, VERIFY_READ);
    	if (unlikely(ret != 0))
    		return ret;
    
    retry_private:
    	//获得自旋锁
    	*hb = queue_lock(q);
    	//原子的将uaddr的值设置到uval中
    	ret = get_futex_value_locked(&uval, uaddr);
    
       ... 
    	//如果当期uaddr指向的值不等于val，即说明其他进程修改了
    	//uaddr指向的值，等待条件不再成立，不用阻塞直接返回。
    	if (uval != val) {
    		//释放锁
    		queue_unlock(q, *hb);
    		ret = -EWOULDBLOCK;
    	}
    
       ...
    	return ret;
    }
    

函数`futex_wait_setup`中主要做了两件事，一是获得自旋锁，二是判断*uaddr是否为预期值。

    static void futex_wait_queue_me(struct futex_hash_bucket *hb, struct futex_q *q,
    				struct hrtimer_sleeper *timeout)
    {
    	//设置进程状态为TASK_INTERRUPTIBLE，cpu调度时只会选择
    	//状态为TASK_RUNNING的进程
    	set_current_state(TASK_INTERRUPTIBLE);
    	//将当期进程（q封装）插入到等待队列中去，然后释放自旋锁
    	queue_me(q, hb);
    
    	//启动定时任务
    	if (timeout) {
    		hrtimer_start_expires(&timeout->timer, HRTIMER_MODE_ABS);
    		if (!hrtimer_active(&timeout->timer))
    			timeout->task = NULL;
    	}
    
    	/*
    	 * If we have been removed from the hash list, then another task
    	 * has tried to wake us, and we can skip the call to schedule().
    	 */
    	if (likely(!plist_node_empty(&q->list))) {
    		 
    		 //如果没有设置过期时间 || 设置了过期时间且还没过期
    		if (!timeout || timeout->task)
    			//系统重新进行进程调度，这个时候cpu会去执行其他进程，该进程会阻塞在这里
    			schedule();
    	}
    	//走到这里说明又被cpu选中运行了
    	__set_current_state(TASK_RUNNING);
    }
    

`futex_wait_queue_me`中主要做几件事：

1.  将当期进程插入到等待队列
2.  启动定时任务
3.  重新调度进程

#### 如何保证条件与等待之间的原子性

在`futex_wait_setup`方法中会加自旋锁；在`futex_wait_queue_me`中将状态设置为`TASK_INTERRUPTIBLE`，调用`queue_me`将当期线程插入到等待队列中，然后才释放自旋锁。也就是说检查uaddr的值的过程跟进程挂起的过程放在同一个临界区中。当释放自旋锁后，这时再更改addr地址的值已经没有关系了，因为当期进程已经加入到等待队列中，能被wake唤醒，不会出现本文开头提到的没人唤醒的问题。

#### futex_wait小结

总结下`futex_wait`流程：

1.  加自旋锁
2.  检测*uaddr是否等于val，如果不相等则会立即返回
3.  将进程状态设置为`TASK_INTERRUPTIBLE`
4.  将当期进程插入到等待队列中
5.  释放自旋锁
6.  创建定时任务：当超过一定时间还没被唤醒时，将进程唤醒
7.  挂起当前进程

### futex_wake

futex_wake

    static int futex_wake(u32 __user *uaddr, int fshared, int nr_wake, u32 bitset)
    {
    	struct futex_hash_bucket *hb;
    	struct futex_q *this, *next;
    	struct plist_head *head;
    	union futex_key key = FUTEX_KEY_INIT;
    	int ret;
    
    	...
    	//根据uaddr的值填充&key的内容
    	ret = get_futex_key(uaddr, fshared, &key, VERIFY_READ);
    	if (unlikely(ret != 0))
    		goto out;
    	//根据&key获得对应uaddr所在的futex_hash_bucket
    	hb = hash_futex(&key);
    	//对该hb加自旋锁
    	spin_lock(&hb->lock);
    	head = &hb->chain;
    	//遍历该hb的链表，注意链表中存储的节点是plist_node类型，而而这里的this却是futex_q类型，这种类型转换是通过c中的container_of机制实现的
    	plist_for_each_entry_safe(this, next, head, list) {
    		if (match_futex (&this->key, &key)) {
    			...
    			//唤醒对应进程
    			wake_futex(this);
    			if (++ret >= nr_wake)
    				break;
    		}
    	}
    	//释放自旋锁
    	spin_unlock(&hb->lock);
    	put_futex_key(fshared, &key);
    out:
    	return ret;
    }
    

`futex_wake`流程如下：

1.  找到uaddr对应的`futex_hash_bucket`，即代码中的hb
2.  对hb加自旋锁
3.  遍历fb的链表，找到uaddr对应的节点
4.  调用`wake_futex`唤起等待的进程
5.  释放自旋锁

`wake_futex`中将制定进程状态设置为`TASK_RUNNING`并加入到系统调度列表中，同时将进程从futex的等待队列中移除掉，具体代码就不分析了，有兴趣的可以自行研究。

### End

Java中的ReentrantLock,Object.wait和Thread.sleep等等底层都是用futex进行线程同步，理解futex的实现能帮助你更好的理解与使用这些上层的同步机制。另外因篇幅与精力有限，涉及到进程调度的相关内容没有具体分析，不过并不妨碍理解文章内容，
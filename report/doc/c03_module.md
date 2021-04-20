# Modules
In this section we will discuss what modules the OSv has and the relationship between those modules.

OSv modules include shared memory, networking channels, and thread scheduler. At first, we have processes that make a call down into the “kernel”. The “kernel” provides a classifier associated with a channel which is a single producer/single consumer queue. Then the channel transfers packets to the application thread. It saves much processes in the traditional network stack and removes the lock and cache-line contention. Also, during this progress OSv allows only one thread to access the data which is stored in a single address space. Compared to the traditional thread scheduler, OSv’s thread scheduler does not use spin-locks and sleeping mutex. Moreover, based on the fairness criteria of threads on the CPU’s run-queue, the scheduler chooses the most appropriate thread to do the next operation which keeps the queue of one CPU not much longer than others’, which is efficient.

Let's dive into the modules a little bit now.

## 1. Virtual Memory

At first, OSv doesn't have multiple spaces. OSv runs an application with the kernel and threads sharing a single space. It means the threads and kernel use the same tables, which make system calls as efficient as function calls and also make context switches quicker. Also, because OSv share a single address space, it doesn't maintain different permissions for the kernel and applications. Therefore, the isolation is managed by the hypervisor. This way achieves simpler code, better performance and also reduces the frequency of TLB misses.

## 2. Networking channels

OSv provides a new network channel so that only one thread could access the data which simplifies the locking. Most of TCP/IP is moved from kernel to the application level, while a tiny packet classifier is running in an interrupt handling thread. Therefore, it could reduce the run time and context switches overhead. The code is implemented in net_channel.cc. It shows that after finding the ipv4 packet which has the same item in the hash table, it will use the pre-channel and wake it up.
```c
bool classifier::post_packet(mbuf* m)
{
    WITH_LOCK(osv::rcu_read_lock) {
        if (auto nc = classify_ipv4_tcp(m)) {
            log_packet_in(m, NETISR_ETHER);
            if (!nc->push(m)) {
                return false;
            }
            // FIXME: find a way to batch wakes
            nc->wake();
            return true;
        }
    }
    return false;
}
```

<p align="center"> <img src="./resources/OSv-Channel.png" width="350" height="425" /> </p>

## 3. Thread Scheduler



Relationship: At first, we have processes that make a system call down into the “kernel”. The “kernel” provides a classifier associated with a channel. The channel which is a single producer/single consumer queue transfers packets to the application thread. It saves much processes in the traditional network stack and removes the lock and cache-line contention. Also, in this progress OSv allows only one thread thread to access the data which is stored in a single address space. Compared to the traditional thread scheduler, OSv’s thread scheduler does not use spin-locks and sleeping mutex. Based on the fairness criteria of threads on the CPU’s run-queue, the scheduler chooses the most appropriate thread to do the next operation which keeps the queue of one CPU is not much longer than others’. 

At first, OSv doesn't have multiple spaces. OSv runs an application with the kernel and threads sharing a single space. It means the threads and kernel use the same tables, which make system calls as efficient as function calls and also make context switches quicker. Also, because OSv shares a single address space, it doesn't maintain different permissions for the kernel and applications. Therefore, the isolation is managed by the hypervisor.

For performance, when OSv calls down to the data, it shares a single address space and avoids context switches. Also, by using the “channel”, OSv saves much processes in the traditional networking stack. OSv avoids the socket locks and TCP/IP locks. According to the performance result, the network stack’s performance for TCP and UDP consistently outperforms Linux and latency is about 25% less than Linux. Also, the context switching (thread switching) is much faster in OSv than in Linux - between 3 and 10 times faster.

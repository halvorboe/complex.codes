---
title: "Types of non-blocking algorithms"
description: "Non blocking algorithms are hard."
pubDate: "May 31 2021"
---

When working with threads, locks are widely used to protect critical sections. Algorithms utilising locks tend to straightforward to reason about, but from a throughput perspective they fall short. Lock-free algorithms solve this issue by replacing locks with cheaper CAS (compare-and-swap) instructions. A downside is that lock-free algorithms often do not guarantee that threads will finish in a finite amount of time. Wait-free algorithms are a subset of lock-free algorithms that don’t utilise locks, so not threads are waiting for each other, but that give back the guarantee that the algorithm will finish in a finite amount of time. As with the run-time of an algorithm, there are different guaranteed finish times.

Algorithms utilising locks are generally straightforward to reason about.

```python
class Stack:

    ...

    def push(self, it):
        with self.mutex:
            self.queue.append(it)

    def pop(self):
        with self.mutex:
            return self.queue.pop()
```

Whenever running PUSH or POP for the stack, a thread waits for a lock to be acquired, then modifies the stack, and returns the lock. Because the parts of the code happening inside the mutex are never running concurrently, we can easily reason about what will happen.

Considering only the work done by each thread the algorithm is efficient, but the throughput not high.

1.  When there is a lot of contention, most of the threads will be waiting rather than doing work.

2.  The amount of time spent executing the instructions is low, but the overhead is high. Locks require system calls and context switches.

For these reasons, most a lot of concurrent data-structure implementations are using CAS instructions to avoid blocking other threads. A CAS instruction is an instruction that changes a value in memory given that it matches a certain expected value.

A push to the stack above could be implemented using a linked list and the following algorithm.

```plaintext
1. Read the current head of the stack.
2. CAS the current head to your value with a pointer to the expected value.
3. If the CAS instruction did not succeed, try again.
```

The difference from the previous approach is that threads are always attempting to make progress, instead of waiting for the other threads. The only reason why one thread would end up not making progress is that another thread made progress. Also, the CAS instructions is much less expensive than acquiring a lock.

A problem with most lock-free algorithms is that they have no guarantee for each thread finishing. For a mutex, the threads enter a queue, so threads that have been waiting for a long time get scheduled before threads that just came in. With lock-free algorithms this is not the case. A CAS instruction has the same chance of succeeding no matter what thread is executing it. When doing batch-compute this is usually not a problem, because the ordering does not matter. In a real time system this can on the other hand cause issues because a thread might usually spend 3 milliseconds to complete a task and then spend 3 seconds or more.

A subset of lock-free algorithms are also what’s called wait-free, which means that they are guaranteed to finish in a finite amount of time. The way this is achieved is most of the time with threads helping other threads. So in the stack example above, if a thread is not able to finish it’s working within a certain number of tries it will ask for help and other threads drop what they’re doing to help the thread finish. Another common approach thread doing work in a non-blocking manner that requires cleanup after. For example to delete an item from a linked list the item is marked as deleted, which can be done in a single operation, and the node is left to be deleted later.

For example, let’s say we wanted the stack above to be wait-free. We might do the following.

```plaintext
If there are any threads asking for help, help them.
1. 10 times
    Read the current head.
    Try to update the head with your value. Return.
2. Add your value to the help queue and help threads asking for help.
```

In this case there are three paths:

1.  The fast path: No threads are asking for help, it updates the head, returns.

2.  The slow path: Updates fail 10 times, adds value to help queue, helps other threads, get’s help, returns.

3.  The helping path: A thread is asking for help, update the head to that threads value, update the head to its own value, returns.

Important note is that a thread can end up helping itself, but it will always help the thread that asked for help first. Also, this approach requires a wait-free queue. Lastly, the operating system scheduler has to be “fair” and give all the threads time.

Reasoning about these algorithms is hard, because you have to make sure that no thread will ever be blocked by other threads, all updates happen correctly and you have to be careful with choosing memory ordering constraints to not get terrible performance.

There are also different guarantees when it comes to wait-free algorithms. Some might finish in an amount of instructions related to the number of threads, while others will finish in a constant amount of instructions.

In summary, lock-free algorithms can help with throughput. If you need throughput and latency guarantees wait-free algorithms can provide both. Implementing a wait free-algorithm is a huge endeavour given the complexity in both assuring correctness and the wait-free guarantees.

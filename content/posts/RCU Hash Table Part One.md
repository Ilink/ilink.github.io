---
title: RCU Hash Table Part One
date: 2023-11-29
draft: false
---
# Introduction
I've been interested in RCU and had a vague need for it in a personal project. By vague need, I mean, I was using it as an excuse to learn about something cool. I also wanted to see if I could write my own RCU userspace algorithm without looking at existing implementations.

My project really needed a threadsafe hash table, so I went with that. This series covers my journey of learning about RCU and implementing my ideas in Rust.

# What is RCU?
RCU is a useless acronym for a very useful algorithm for handling heavy-read, low-write concurrency. Imagine there's a linked list with occasional writes, but a very heavy concurrent read workload. Sure, you could use reader-writer locks, but that blocks readers when there are concurrent writes.

Reader side:
* A reader must obtain a RCU lock, then obtain the root pointer from the list atomically
	* It cannot hold onto that pointer - it must relinquish it after unlocking!
* The reader iterates through the list, taking pointers atomically as it goes
* The reader unlocks its RCU lock

Writer side:
* Only one writer can be active at a time
* Iterate through the list, looking for some value to update
* The old data is copied into a new node. The new node is updated.
	* This is where RCU gets its name - "read-copy-update"
* The new node is inserted into the old position by rewriting the adjacent pointer(s).

Note that the writer does not actually free anything. Since all of these operations are atomic, there's no way for a reader to get invalid data. However, there does need to be some form of garbage collection. The GC aspect is actually the most interesting part of RCU!

Additionally, while most examples you'll find easily use linked lists, RCU can really be used to protect a variety of data structures. My own project is to create an algorithm which applies to a hash table.

# Kernel Space GC
After a deletion/edit has occurred, no subsequent readers will be able to see the old data. Therefore, once the current set of readers are done, nothing can see the data. The data can be deleted at that point.

After writing, the kernel waits for all readers running from before it started to finish: 
```
old_node = do_update()
wait_for_current_readers_to_finish()
delete(old_node)
```

Determining if a thread is still using data is conceptually simple for the kernel, since it's in charge of actually running threads. The unlock method is just a context switch and nothing more. When the kernel gets control of the thread again, it knows that it is done reading and is therefore unable to use any old data.

This is hand-waiving the details, mostly because I was more interested in the userspace side. If you want to know more, I found [this lwn article](https://lwn.net/Articles/262464/#Wait%20For%20Pre-Existing%20RCU%20Readers%20to%20Complete) and [this talk from Fedor Pikus](https://www.youtube.com/watch?v=rxQ5K9lo034) to be very informative.

# Userspace GC
Since we can't cheat and completely control thread execution, the userspace algorithm(s) I came up with are a bit more involved.

## Generation ID
We still need to find a way of tracking which threads are using old data. This algorithm uses the idea of a "generation ID". This is simply a counter that is incremented every time an edit is performed. Essentially, this tracks a point in time. If every reader sees the latest generation ID, then everything replaced by an edit can be deleted. We can go one step further than this:
* Find the minimum generation ID across all readers
* Any old item from a lower generation ID can be deleted. No reader is using it, because they're all on a greater generation ID.

In subsequent articles, we'll go more in-depth into how this works exactly. I have already implemented this algorithm and it seems correct so far. I still need to test on a weakly-ordered CPU (I'll be using ARM64) but it seems correct on x86.

## Reference Counting
The list looks node looks something like this:
```rust
struct Entry {
	ref_cnt: AtomicUsize,
	ptr: AtomicPtr<T>,
	next: AtomicPtr<Entry>,
	deleted: AtomicBool,
}
```
The reader must atomically increment the counter before reading `ptr` (or `next`). That way, the data cannot disappear before it dereferences the pointer. Note that because the ref_cnt might actually be for a different ptr than what was originally there when it incremented it!

There's quite a bit more trickyness to this algorithm which we'll get into in future articles. I haven't implemented this one yet, so some of that trickyness is yet-unknown to me. It might also be that this design as presented doesn't work at all!



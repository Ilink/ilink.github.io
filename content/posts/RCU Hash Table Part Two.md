---
title: RCU Hash Table - Part Two
date: 2023-12-04
draft: false
---
# Intro
Let's dive more into the generation ID-based RCU hash table algorithm. I had some basic principles which guided me through this process:
* I did not want to make a [fast hash table](https://www.youtube.com/watch?v=ncHmEUmJZf4). I wanted to make an extremely basic hash table with interesting and maybe-fast lock-free concurrency.
* The API can be extremely simple - it's just not my focus.
* The algorithm(s) must be correct.
* I wanted to learn more Rust along the way. Otherwise, I would have just written this much more quickly in C++!

# Data Structures
The hash table itself is a simple open-addressing design. The table can go up to a load factor of 0.5 before it resizes. 
The underlying data is a vector of pointers:
```rust
struct HashTableVal<T> {
    key: String,
    val: T,
}

struct HashTableEntry<T> {
    ptr:AtomicPtr<T>
}

type EntryVec<T> = Vec<HashTableEntry<HashTableVal<T>>>;

// This has the same length as EntryVec<T>
type Storage<T> = Vec<Option<Box<HashTableVal<T>>>>
```
Pointers must be used because the data has to be stable in the face of updates. Remember, readers can see old versions of the entries! 

These atomic pointers point to the underlying data within `Storage<T>`. The writer, which is single threaded, sets the values there. Then it atomically sets the pointers within `EntryVec<T>`. The readers only interact with `EntryVec<T>`. This is just to make things threadsafe, since I don't think there's a way of using `Box` with `AtomicPtr` otherwise. The size of both `EntryVec<T>` and `Storage<T>` are always identical. The indexing scheme is also the exact same. Logically, the same data is in the same relative index on both vectors.

When data is updated, the old entries are moved into a separate vector called `OldEntries<T>`:
```rust
struct GenerationEntry<T> {
	val: Box<HashTableVal<T>>,
	generation_id: usize,
}
type OldEntries<T> = Vec<GenerationEntry<T>>
```
The `OldEntries<T>` vector is iterated over by the garbage collector before finally being deleted. This is covered more in the [garbage collection section](#garbage-collection).

# Insert
Insertion is simply a matter of hashing the value and finding the first matching/empty slot. As mentioned in the [Data Structures](#data-structures) section, inserting a new value is done into 2 vectors. The process is essentially:
1. The writer takes the write lock
2. The new value is inserted into the slot within `EntryVec<T>`.
3. The pointer is atomically set in `Storage<T>`.

If the slot is empty, there's not much to do. The new element is allocated and placed into the slot. If an insertion updates an old entry, the old entry must be moved to `OldEntries<T>`. As part of this, a new `GenerationEntry<T>` is created. This takes ownership of the old data. Additionally, the field `GenerationEntry<T>::generation_id` is set to the current generation id. If all readers are on a higher generation id, that old entry can be freed. See more details in the [Garbage Collection](#garbage-collection) section.

Finally, and I do stress finally, the generation id is incremented. This is done as a [store-release operation](https://preshing.com/20120913/acquire-and-release-semantics/). It must not get reordered and happen before the other operations. If that were to happen, the id could be updated before the new element has been inserted. Let's step through what could happen:
* The generation ID is updated to N
* A reader begins and sees generation ID N, but grabs the old entry from generation N-1
* The data is updated atomically and the insert is finished
* A new insertion begins and updates the generation ID to N+1
* Garbage collection occurs. This notes that the lowest reader generation is N. The item at N-1 is freed. The reader is still using that, so memory corruption/a crash/fiery doom occurs.

Another way of looking at this is that it's ok if an entry is associated with an earlier generation ID. Remember that only entries earlier than the minimum generation ID across all readers can be freed. The entry may be on ID=N while the reader saw ID=N-1. That's ok. It won't be freed because N > N-1. 

## Resize
When the hash table reaches load factor (too many entries), it must be resized. Since this is concurrent, the old hash table vector must be preserved in case readers are still using it. Logically, this is done in the exact same way as handling old entries in the hash table itself.

The rest is fairly straightforward:
* Create 2 new vectors - one for the atomic pointers, one for the backing storage.
* Re-hash every current hash table entry and insert them into the new hash table vectors.
* Place the old hash table vector into "old" storage.
* Update the generation ID.
	* This must occur at the end and is done with acquire-release semantics.

# Garbage Collection
The crux of the algorithm is how garbage collection is performed. When an entry has a generation ID less than the minimum generation ID across all readers, it can be deleted. 

To simplify this concept, let's say there's only one reader at generation 2. There's one entry in old storage and it has generation ID 1. Since the reader is beyond 1, it can be deleted.
* The hash table pointers are updated atomically, so there is no way for the reader to see the old data after an update has occurred.
* Furthermore, there's no way to see the latest generation ID with old data. This is because of the acquire-release semantics on generation ID as described in the [insertion section](#insert).

When an insertion/update is performed, the existing entry is moved into a secondary data structure. The current generation ID is recorded on the entry at that time.
```rust
struct GenerationEntry<T> {
	val: Box<HashTableVal<T>>,
	generation_id: usize,
}
type OldEntries<T> = Vec<GenerationEntry<T>>
```
`GenerationEntry<T>::generation_id` is what's compared against the readers minimum generation ID.

Let's go into a little more detail about why a greater generation ID enables this. Conceptually, a generation ID is like a point in time. It monotonically increases.
To illustrate this with a more concrete example:
* A reader begins and sees generation ID 1, then atomically reads the entry (pointer) at slot 10.
* The writer inserts a new entry atomically at slot 10 and updates the generation ID to 2.
	* After this point, there is no way to get the old entry again
* The reader releases its lock
* The reader comes back and reads slot 10 again, this time seeing generation ID 2. It gets the latest entry.
	* There is no way for it to get the old entry at this point because it was atomically updated.
* Garbage collection runs. There's no way the reader can see the old entry, and it is therefore deleted. Remember that all readers have now seen a higher generation ID. Additionally, no reader performing a new read can see any old entries.

# Read/Get
This is simple because all of the complexity is within the write path:
* The atomic generation ID is read and stored in the reader state
	* One nice thing about this algorithm is that there's no need for atomic fetch_add, which is a more expensive atomic operation.
* The key is hashed and turned into an index
* A linear scan of the hash table is performed, starting from the hashed-to index. If there's an empty slot before the key is found, `None` is returned. Otherwise `Some()` is returned with a reference to the data in that slot.

Since the algorithm ensures the readers can always access entries, there's nothing else to be done! The hash table is designed for a read-heavy workload. That's why all of the complexity is stuffed into the writer. Acquiring a lock on the hash table only requires 1 atomic read.


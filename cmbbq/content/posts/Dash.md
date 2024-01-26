+++
title = "Dash: Scalable Hashing"
date = "2024-01-26"
tags = ["sys"]
description = "The main focus of the Dash paper was on the once fashionable `persistent memory`, but in reality, any `memory bandwidth`-limited scenario can benefit from it. With Intel killing off its `pmem` business, the significance of the `Dash` approach has shifted to regular DRAM applications."
showFullContent = false
+++

The main focus of the Dash paper was on the once fashionable `persistent memory`, but in reality, any `memory bandwidth`-limited scenario can benefit from it. With Intel killing off its `pmem` business, the significance of the `Dash` approach has shifted to regular DRAM applications.

# Dynamic Hashing
`Dashtable`, the proposed scalable hashtable, evolves from `extendible hashing`. 

`Extendible hashing` is a hash system that uses the first $N$ bits of hashed values to look up buckets in a trie-structured `directory`.

A `directory` with `global depth` $N$ can hold $2^N$ buckets. It means $N$ is the key size that maps the `directory`.

Each bucket also has a `local depth` $M(M \le N)$, which is the key size that previously mapped the `directory`. Any bucket having a `local depth` $M = N$ is pointed-to by exactly one `directory` entry. Any bucket having a `local depth` $M \lt N$ is pointed-to by more than one `directory` entries. 

The minimal $N$ needed to ensure every item has a unique bucket index is 1 for 2 items. The minimal $N = 2$  for 4 items. Everytime a new  item added into a bucket, if the number of items in the bucket exceeds a certain threshold, a rehashing operation happens by splitting the bucket into 2 parts. Hence rehashing in this scheme doesn't have to stop the world and do a full-table scan & copy, but instead is done incremental.

Similar to `extendible hashing`, `linear hashing` also uses a `directory` to orgranize and address buckets. The distinction lies in split control. In `linear hashing`, a split typically occurs only if the load factor exceeds a threshold and the bucket to be split is chosen in a "linear" manner.  

# Dash for Extendible Hashing
## Overview
![dash_eh](https://cmbbq.github.io/img/dash_eh.png)

In `Dash-EH`, each `directory` entry points to a `segment` which consists of a fixed number of normal buckets and stash buckets. A `segment` can be viewed as a sub-hashtable of constant size. The so-called stash buckets shares the same layout as the normal buckets, responsible for storing overflow records. 

![dash_eh](https://cmbbq.github.io/img/dash_eh_bucket.png)

The core idea of `Dash-EH` is to pay a little bit extra space in metadata to buy faster probing with fingerprints and lightweight concurrency control with version locks.

Inside a `Dash-EH` bucket, as shown in the figure above, the first 32 bytes are metadata, including version lock, counter, alloc bitmap, membership bitmap for load balancing, and 18 one-byte fingerprints for bucket probing(14 for slots in the bucket, 4 for overflow records originally hashed to this bucket). It is followed by $16(Bytes) \times 14 (records) = 224 Bytes$ payload, which stores 14 16-byte records. 

## Fingerprinting
Bucket probing, which refers to searching for a slot in a bucket, is a basic operation of hashtables, needed by search, insert and delete operations to locate a particular key. Traditionally probing requires a linear scan, which is naturally slow on `PMem` and could be completely unnecessary when a searched key doesn't exist. `Dash-EH` employs fingerprinting to reduce unnecessary scans. Fingerprints are the least significant byte of hashes of keys. To probe for a key, the probing thread first checks if any fingerprint in the bucket's metadata matches the key's, so it can skip buckets without any fingerprint match.

Fingerprinting primarily benefits negative(key-not-found) searches. The `Dash` paper also claims fingerprinting enables using larger buckets spanning more than 2 cachelines. But I have to take it with a grain of salt. The paper itself uses a 256B setup. So does the DragonFly implementation[^1]. In theory, larger buckets can indeed tolerate more collisions and improve the load factor; however, this may come at the cost of compromising locality to a certain degree - you don't want to load multiple times for a single bucket access in a hashtable.

## Bucket Load Balancing
Segmentation reduces cache misses on `directory` by reducing its size. In the `extendible hashing` scheme, if any bucket in a segment is full, the entire segment needs to be split, even though other buckets might have much free space. 

Thus the `Dash-EH` algorithm design takes bucket load balancing into consideration. To insert a record, `Dash-EH` probes both bucket $B_b$ and $B_{b+1}$, and inserts into the bucket that is less full. If both $B_b$ and $B_{b+1}$ are full, `Dash-EH` tries to displace a "native record" from $B_{b+1}$ to $B_{b+2}$, or move a "rebalanced record" from $B_b$ back to $B_{b-1}$ where it originally belongs.  

The per-bucket membership bitmap is used to decide whether a record is rebalanced or native. If a bit is set in the membership bitmap, then the corresponding key was not directly hashing into this bucket(native) but placed here due to re-balancing(rebalanced).

If both insert and displacement failed, `Dash-EH` turns to the last resort - stashing. Each segment has a fixed number of stash buckets to hold these overflow records. Probing stash buckets introduces significant overhead to negative searches and inserts(needs uniqueness check). To address this issue, each normal bucket reserves certain metadata fields. 4 overflow fingerprints are reserved for overflow records stored in stsh buckets. A overflow bit indicates if there exists an overflow at all. So if there isn't an overflow in a bucket, the search/insert operation doesn't have to probe stash buckets. Anyway, it is still advisable to maintain a small number of stash buckets. The paper claims "using 2–4 stash buckets per segment can improve load factor to over 90% without imposing significant overhead". In Dragonfly's Dashtable, each segment has 56 regular buckets, and 4 stash buckets.

## Lightweight Concurrency Control
The lightweight concurrency control in `Dash-EH` naturally scales well on today's `many-core` architectures, out-performing traiditional bucket-level shared locks. 

Write operations follow traditional bucket-level locking to lock the affected buckets, using CAS over a lock bit. If a write is done, the writer thread resets the lock bit and increments the per-bucket version number by one.

On the other hand, read operations are designed to be lock-free. Before a read, the reader thread first fetches a snapshot of the lock word, waits until the lock is released, then proceed to read without holding any lock. After reading, it will check the lock word again to verify the version number stays unchanged. If the version is changed, it retries the entire operation.

# Dash for Linear Hashing
The Dash paper also presents `Dash-LH`, a Dash-enabled linear hashing approach built upon building blocks used in `Dash-EH`, such as balanced insert, displacement, fingerprinting and optimistic concurrency; they are pretty much orthogonal after all. The main difference is that `Dash-LH` split the segment pointed to by a pointer in a linear manner. 

Traditional `linear hashing` link overflow records with a linklist. In `Dash-LH`, it's done more cache-friendly with stash buckets. It still needs to chain these stash buckets though. Still it's much better than chaining individual records. 



[^1]: [Dashtable in Dragonfly](https://github.com/dragonflydb/dragonfly/blob/main/docs/dashtable.md)






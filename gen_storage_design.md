Generational Tail-Append Storage Design Specification
=======================================

# INTRODUCTION

## Purpose of this Document

This document describes the design, integration and implementation considerations of Generational Tail-Append Storage in Couchbase Server.

## Motivation

Generational Storage (GS) is an evolution of the tail append design of CouchDB and Couchstore (the Couchbase storage subsystem).

The CouchDB and Couchstore storage system uses a tail append design, where all updates to a database result in appending new or updated information to storage file. Even deletes would grow the storage files until compaction.

Some advantages of this design are:
* Extremely efficient and fast durable updates, minimizing disk head seeks on HDDs and minimizing write amplification on SSDs.
* Fast and simple recovery from crashes, simply scan backwards from the end of the file to the first good header and start using the file.
* Zero cost MVCC snapshots. Simply use the most recent version of a file header and get a completely consistent storage file concurrent with updates and compaction.

The biggest disadvantage is that recovering wasted file space means copying all live to another file. Even if all the wasted space is due to small portion of the documents being updated, it will still require copying all live documents to a new file.

An improvement of this new design over CouchDB and previous Couchstore designs are them optimization for Zipfian/Zeta distributions of document update frequencies.

Empirically, most interactive database workloads follow a Zeta distribution (aka Zipfian or Exponential), where a small percentage of records are updated frequently, a larger percentage are updated less frequently, and most are updated rarely or never. A storage system that is optimized for these frequently occurring distributions can be an orders of magnitude faster and more efficient for updates, scans, and reads, than a system designed for more uniform distributions.

## Scope of this Document

This document will describe the high level algorithms, data structures, and components necessary to implement Generational Tail- Append Storage in Couchbase server, and the motivations behind the design decisions.

#Background and Current Implementation

This new storage design is an evolution of the CouchDB and Couchbase implementations. Both share many of the same core design aspects, with CouchDB being more full featured and Couchstore having faster and more efficient implementations.

## Components of a Couchbase Storage File
A storage file contains all of the following:

* Document bodies - Each document is written to contiguous region of a file, and can be interleaved with any other component of the file.
* by_id index - A copy-on-write btree arranged by doc Id/key and contains metadata and file location of each document
* by_seq index - A copy-on-write btree arranged by the latest update sequence for a documents and contains metadata and file locations of each document.
* Headers(s) - They contains pointers to the roots of the btrees at a particular point in time, and other metadata about the database file. The header closest to the end of the file represents the most recent state of the database. They can also be called a "trailer", since it always occurs after all valid data and metadata in a file.

### 4k Header Marker Boundaries
To differentiate between headers and data, especially data that might look like a header, every 4k in the file, a single byte takes the value of either 0x00 or 0x01. 0x01 means that what follows the byte is a database header. Non-header, or data blocks, have 0x00.

When opening the file, the file is scanned backward to find the first intact header. It's assumed if an intact header is found, all data preceding it is intact. It ensures this when performing a update by first fsync'ing all data and meta, then writing and fsync'ing the header, making it impossible for a failed write or even power failure to allow a header to incomplete data. Also CRC32 integrity markers are always written and validated for each object on read, to detect if the media itself becomes corrupted or the OS doesn't actually correctly perform the fsync.

For datablocks (documents, btrees nodes or anything not a header), any data that spans across the 4k marker will have the 0x00 added when written, and stripped out when read. Otherwise Couchbase is not aware of any other block boundaries.

### All Mutations Are Appends

When an insert or update occurs, the document is written to the end of the file, and the btrees and the contained metadata are updated and written to the end of the file. If it's a deletion, then typically only the btrees are updated. Then the header is written.

The tail-append design means additional seeks are necessary to write the data (excluding file system metadata), which minimizes write amplification in SSDs and disk seeks on HDDs.

"Write amplification" is the additional low level storage blocks that must be rewritten for any successful write on a SSD. Sequential writes/tail append will minimize write amplification, random internal writes tend to maximize it. Lowering write amplification increases SSD write performance and operational life.

<http://en.wikipedia.org/wiki/Write_amplification>

<http://en.wikipedia.org/wiki/Write_amplification#Sequential_writes>

<http://en.wikipedia.org/wiki/Write_amplification#Random_writes>

### Recovery Oriented Design

The tail append design ensures data integrity of committed data regardless of orderly shutdown or crash. Should a write fail do to complete because of any kind of crash, even power loss, previous data written and indexes written to the file are still intact and untouched. And incomplete writes and index updates simply appear as garbage data at the end of the file, and are ignored on database open. This is due to the pure tail append design, which means special recovery code, such as consistency checks or fixups, are never needed.

## Organized by Id and by Update Sequence

The 2 btrees for indexing data are by\_id and by\_seq.

The by\_id btree is used to find a document by id/key.

The by\_seq index is primarily used for range scans to find document mutations since a particular point in time. Each document mutation (insert/update/delete) in the database is assigned a sequence number, starting with 1 and incremented by 1 for each mutation. For new documents, only a new by\_seq entry is added to the index. For updates or deletes, the previous by\_seq entry for the document is deleted and the new one is added in a single operation.

When walking the by\_seq index, the nodes won't necessarily be well ordered or display good locality (that is to say over time the nodes can approach random file locations), but the documents are disk will be read in the order they appear on disk. For HDDs, this will greatly reduce the number and cost of head seeks, and typically benefits from read-ahead optimization on HDDs and SSDs.

Also, the most recently written document bodies and btree nodes will be clustered near the end of the file and will have good data locality, which means greater efficiency serving continuously active UPR clients (which read documents in by\_seq order) that are likely to be only behind by a small number of updates and therefore only requiring nodes and data near the end of the file.

## Immutable, COW Btrees

The Couchstore btrees are a variant of traditional b+trees, where only the leaf nodes contain key/value pairs, and inner nodes contain key/pointer pairs, where the pointer is the offset in the file for a subnode.

The btree's are immutable and all updates are Copy-On-Write. This means for an update occurred in the btree, no existing btree nodes (root, inner or leaf) are  modified, but rather updated in memory and copied to the end of the file. Therefor to update a leaf node, it's rewritten and it's parent node must also be rewritten to point to the new node. And it's parent, and so on, until finally the root is recopied. Only the nodes from the modified leaf to the root are written, all other nodes remain in the original locations.

Until compaction happens, this can have higher space requirements than update-in-place btrees (which can also suffer from internal fragmentation), but is still extremely fast as each node written is appended to the end of the file, one after another. This is efficient both for spinning HDDs as no extra seeks are necessary to commit, and also SSDs, as it has the least cost of write-amplification.

Batching many updates into a single commit will also reduce the amount of btree nodes rewritten per document, and becoming more even efficient the more tightly grouped the updates are in the leaf nodes.

Unlike update in-place btrees, Couchstore btree's do not self balance. For the by\_id btree, since it doesn't "lose values", all operations are guaranteed to keep the btree balanced. So long as all updates are inserts, the by\_seq btree will remain balanced. But certain update patterns can cause the by\_seq btree to become unbalanced. However, the process of database compaction, necessary in the update case, will regenerate the btrees from the ground up and restore optimal locality and rebalance.

## Crash/Powerloss tolerant

Because of the tail-append nature, any update to that fails to complete will also fail to write a valid header. When the file is reopened, the incompletely written data is seen as garbage at the end of the file and skipped over until the system finds the first valid header, and all data preceding it will be valid, assuming the physical media has not corrupted.

Currently Couchstore and CouchDB do 2 fsyncs to ensure the database can never be corrupted. First they fsync all data and btree writes to disk, and then write the new header (at the end of the file) to point to the newly updated btrees.

## MVCC

When updates occurs, any existing on-disk data and metadata aren't modified. This means MVCC snapshotting of the entire database file is automatic and essentially free, requiring no explicit locks or resources beyond what a single threaded access require, as reads to the database can happen concurrently with writes, with no hard limit on the number of concurrent read snapshots. With a shared file cache, each concurrent reader often uses less resources per-snapshot (file system cache memory and disk fetches).

The efficiency of snapshotting is particularly important when performing range scans of by\_seq index for UPR, which requires UPR clients (backup utlities, indexers, replicators, etc) are able to get a consistent and complete snapshot to ensure fast recovery points in failover scenarios.

## Compaction

Compaction is the process of recovering wasted space in a storage file and arranging all the live data in a and btrees to be optimized for size and arrangement. It copies all data and btree indexes from the old file to a new file. It does this without without stopping or pausing persistence and fetches, but can result in duplicate IO for writes that occurred during the compaction.

The compaction doesn't block reads or writes, letting them proceed normally, though possibly slowed through increased io utilization of the copying process.

The compaction process scans the by\_seq index and load all the live documents from the database and copying them into a new file. It creates a by\_seq index bottom up in the new file by building the node the leafs up to the root node, the sorts the same information by key/id with on-disk heap sort, and then builds up the new by\_id btree from the bottom up.

Once it's completed a snapshot, it checks to see if it's caught up with the old storage file, as new mutations could have been committed to it during the compaction. It then walks the by_seq index of the old file, picking up from the sequence where it left off, copying values into the new database file.

It repeats this process until it's fully caught up with all mutations to the main file. It's theoretically possible the updates to the main file happen faster than compaction can keep up with, but this has never been observed in practice.

### Shortcomings

In CouchDB, all documents for a database are written to a single storage file, meaning compaction requires at least as much free diskspace as the newly compacted file will require once complete, which must hold the entire data set. This means a filesystem without a fairly large amount of spare storage cannot complete a compaction.

In Couchbase 2.0 all documents are written to a partition specific storage file, and with 1024 partitions it effectively solves the free space problem with compaction as it is now far more incremental (each unit of compaction work is 1/1024 of the total dataset) and therefore needs much less temporary disk space or memory.

It also makes it easy to read, write or delete many or all documents from a partition more efficiently, as is the case for rebalances. Documents grouped by partition have better locality for scans, writing in bulk doesn't cause extra compaction or migration IO in other partitions, and deleting a partition is simply deleting the storage file as a single operation.

But for Couchbase 2.0 storage this introduces a new problem, to durably committing a large number of documents to different partitions requires at a minimum 1 fsync per partition, greatly increasing latency for a durable commit.

The cost of compaction per document updated is 0((N*log(N))/M) where N is the total number of document and M is total number of unique documents updated between compactions.

The worst case cost of compaction is where a single document in a large partition is updated repeatedly. Recovering the garbage space from the updates means copying all documents in the partition to a new file, even the documents that haven't been updated in a very long time.

The best case for compaction is when there is a distribution of single updates across all documents, so as M becomes large and approaches N, the cost become O(log(N)) per unique document update.

## Generational Storage

Generational Append-Only Storage is a new design that is heavily based on the current Couchstore design, and addresses many of the shortcomings, particularly for interactive workloads.

It has been empirically observed that most interactive workloads have approximations of a Zipfian/Zeta distribution of retrievals and updates to documents. This means the most popular document is retrieved/updated 2x more than the next most popular, 3x more than the next after that, 4x more the one after that, and so on.

The generational storage system attempts to group documents into generations of "hotness", where the most updated documents stay in the smallest, hottest and most frequently compacted generation, colder documents tend to migrate down to the next exponentially larger, colder generation (which because it's updated less frequently is compacted less frequently), and the colder still documents migrate down to the next exponentially larger, colder, less frequently compacted generation, and so on. The larger and colder the generation, the less frequently it's compacted, greatly reducing compaction costs.

Generational storage also has the advantage of larger bulk write for each colder generation, reducing btree fragmentation and garbage, and improving locality of documents, especially useful when performing by\_seq scans as documents are more likely to be contiguous, making read ahead highly advantageous for SSD and especially for HDDs. Also it's possible to preallocate the storage needed to support the bulk transaction, reducing file system fragmentation, again highly advantageous requiring fewer block "extents" mappings and less seeks for HDDs.

When workloads where many documents have widely ranging update frequencies (such as Zipfian) the frequency of update of a document (it's "hotness") determines the generation the document migrates to, and how therefore how frequently it gets compacted.

Some key properties we want to maintain when working with a interactive/Zipfian workloads:

* Consistent low latency durable writes/updates and fetches
* Durable storage immune to corruption due to process/OS crash or power loss
* Fast startup and warmup time
* High throughput, restartable by\_seq scans when indexing, replicating, rebalancing or backing-up a large amounts of changes
* Partition level MVCC for reads and consistency of indexing and replication (necessary for UPR), and whole database MVCC for point-in-time backups.
* Efficient partition rebalance, both in and out of a node.
* Cost of rebalancing a partition in and out of a node is the same regardless the number/size of other partitions on a node
* Low overhead and frequency of compaction.
* Pauseless compaction (files are compacted, reorganized, and defragmented without pausing durable writes or fetches)
* Low diskspace overhead (disk space usage isn't dominated by internal fragmentation or index overhead)

# System Architecture

## Write Ahead Log

All commits to storage are written to an index-less, tail-append write ahead log (WAL) file. There are a series of these files numbered from 0 and up (0.log, 1.log ... _N_.log), and all new mutations happen to the highest numbered file. Until the data is moved to partition specific storage, the most recent update to a document is kept in memory (as dirty) and cannot be ejected. However, once written and persisted to the log, a client waiting for the write to be durable will get the success response from the server.

On startup, the WAL file or files are read from beginning to end into memory, with subsequent updates to the same document replacing in the in-memory representation of the earlier mutation. These newly read documents are still marked un-ejectable from RAM until they are written and committed to partition specific storage.

The advantage of the WAL is to reduce the latency of a durable commit, as it only has to wait on a single fsync of a sequential write for all partitions, instead of an fsync per partition, which can increase latency dramatically.

The downside is the every document commit in the worst case will be written to disk 2 times, once to the log and at least once in the partition specific generational storage, increasing disk contention and reducing throughput from it's theoretical maximum. For subsets of documents that are updated constantly, the number of times rewritten can be much smaller than 2 (but always greater than 1), as only the most recent commit is copied the partition storage, the rest are ignored and discarded.

### Async Copy from WAL to Partition Storage

Asynchronous to WAL commits, a separate process or thread will copy the most recent versions of data from memory to the youngest generation of partition specific storage. When this process starts, writes to the most recent (highest numbered) log file cease and new writes happen to a new high numbered log file. Once the partition copy process has copied all the live data from that log file to their specific partition storage, that log file is deleted. The process then begins again with next highest numbered log file, until no more mutations remain, or happening continuously if there is a continuous write load.

### Natural Write Throttle

While it is copying files from the log to storage, it might need to recopy data from a younger generation to an older generation and/or in-place compact any of the generational files. When this happens, the copy/compaction process may get behind the update/insert rate of the log files. If this happens for too long, all the bucket memory quota will be dedicated to un-ejectable items (even though they may be safe on disk) and it will pause client updates as the bucket will not be able to accept any client writes, until the items in memory are marked as permanently persisted (moved from the WAL to partition storage).

## Generational Storage for Partitions

We now introduce the concept of Generational Storage (GS) For each partition that a bucket is responsible for, either as a master or replica/backup, it will contain a series of Append-Only storage files, which independently look and behave very much like 2.0 Couchstore partition storage. Unlike 2.0 storage, there isn't one big file that contains all data for that partition, but instead contain a series of files arranged in generations, each being exponentially larger than previous generation.

Initial writes to GS occur at the smallest and youngest generation, until that generation exceeds it's live data size threshold, then the coldest documents (least recently or least frequently updated) documents are copied to the next colder generation, and the hottest documents are compacted in place for the same generation, or dropped if it can be determined a more recent version is in a hotter generation. This processes is repeated recursively until a generation is reached that is under it's live data threshold.

### Zipfian Update Workloads

GS write performance is optimizing for workloads where there is a Zipfian distribution of update frequency. That is, over any given period of time, the 1st most updated document is updated 2x more times than the 2nd, and 3x more than the 3rd, and 4x more than the forth, and so on.

For a perfect zipfian update distribution, and a perfect "hotness-detection" of documents, GS will be a nearly ideal tradeoff, keeping IO costs per document update to the bare minimum. Even for workloads where there distribution isn't completely zipfian, but instead there are varied groupings of documents with similar update frequencies, GS will still tend to group documents of similar hotness into one of more generations.

Consider a database where 50% of the documents are consistently updated 2x more often than the remaining 50%, those documents will tend to stay in the hotter generations that can contain them (with perhaps a poor distribution among those generations), and the coldest will tend to stay in the coldest generation, thereby reducing the compaction frequency of the cold documents and improving IO costs substantially.

### Zipfian Read Workloads

A large RAM cache will improve the performance for a working set that can fit into it entirely, but if not large enough and if there is a strong correlation between how often a document is updated and how often it's read, GS will also provide much cheaper disk based accesses.

However if there is little connection between updates and reads, generational storage will offer slightly less fetch efficiency over a single storage file (due to statistically more btree nodes reads necessary), or theoretically a storage scheme that optimizes disk layout for read frequency (such as traditional balanced btrees).

Combined with per-generation bloom filters, efficiencies similar to a single storage file can be obtained (with a memory and disk space overhead of maintaining the bloom filter), and will be always be in the order O(logN) as a single file.

### Optimizing "Coldness"-detection

A key to optimizing GS for zipfian workloads is to ensure the reasonable ranking of the "hotness" of documents and migrating and keeping at the appropriate generation. Perfect ranking requires future knowledge of update patterns, which is NP-hard (http://www.cse.msu.edu/~torng/Research/Pubs/offline-cache-paper.pdf), but heuristics will be employed and can be compared for  efficiency. The cost of the heuristic -- including space, IOPs, CPU and latency -- must be balanced for general server performance.

A simple but serviceable heuristic is a Least Recently Updated scheme. For any generation file, the documents are arranged by\_seq order, with the most recently updated documents having the highest seq. When exceeding the size threshold, simply copy the documents with the lowest seq (LRU) to the next generation, and keep rest in the current generation. Determining the optimal percentage of documents to move the larger generation vs. keeping in the current is also NP-hard. Reasonable defaults can be determined via empirical means and made configurable for different workloads.

The efficiency of the heuristic depends a great deal on the exact workload patterns, and how favorably it compares to other schemes for those patterns. Devising alternative heuristics is outside the scope of this document.

### Storage File Naming

All generation storage files for a node will be stored in a single directory, with naming convention as follows:

%PartitionNumber%-%GenerationLevel%-%CompactionVersion%.cb

ParitionNumber is an integer between 0 and N - 1, where N is the total number of partitions. It represents the partition the storage file belongs to.

GenerationLevel is a monotonically increasing integer 0 to G, where G is coldest, largest generation. There will never be gaps between generations, if generation 5 exists, so must 4, 3, 2, 1 and 0.

CompactionVersion is a monotonically increasing integer N that is the same as the number of times the generation has been compacted in-place. For a small window of time as compaction completes, there will be 2 files, the older file being N and the new file being N+1. Once version N+1 is in place, the older file N will be deleted.

### Generation File Degradation

As generations get insertions, updates, and deletions, they tend to degrade in performance and storage efficiency. This is due to several factors:

* Wasted Space - For every document update or deletion, there is now wasted space of a previous version that is occupying disk space.
* Fragmentation - The wasted space, when it occurs between 2 live documents, increases cost of reading the live documents when scanning the documents in by\_seq order, as reading the 2 live documents will either have to also read any wasted space at the file system block level at the end of the first document and at the beginning of the second document, or perform a costly seek to skip over the wasted space. For SSDs the seeks aren't as much a concern except for wasted read-ahead cost that's often performed at various FS cache levels, and for HDDs, the cost is often greater as it can involved a physical disk head arm movement  (though the ordered nature of the document on disk tend toward smaller arm movements).
* Fragmentation - The wasted space, when it occurs between 2 live documents, increases cost of reading the live documents when scanning the documents in by\_seq order, as reading the 2 live documents will either have to also read any wasted space at the file system block level at the end of the first document and at the beginning of the second document, or perform a costly seek to skip over the wasted space. For SSDs the seeks aren't as much a concern except for wasted read-ahead cost that's often performed at various FS cache levels, and for HDDs, the cost is often greater as it can involved a physical disk head arm movement  (though the ordered nature of the documents on disk tend toward smaller arm movements).
* Poorer btree node locality. Each update to the generation means btree nodes must rewritten at the end of the file, meaning poorer locality for some branches of the btree, and for nodes not in cache this can entail expensive disk head arm movements.
* Unbalanced Btree Nodes - For certain update patterns, the deletion of old by\_seq entries for new entries mean the btree nodes can  deteriorate from their ideal balance of the same depth for all leaf nodes with a average case access cost of O(logN) to all nodes, to an absolute worst case of linear cost O(N) to some leafs nodes.
* Higher false hit ratios of the bloom filters. When a generation is newly created, the bloom filters will have their lowest average false positive rate. As new documents are added to the generation, the false positive rate will increase.

However, the process of compaction will restore the storage generation to optimal btree layouts, an optimal bloom filter, with no wasted space from garbage documents or btree nodes. This limits the total degradation possible for a storage generation.

### Batch Inserts, Better data locality per generation

Another advantage of generational storage to a single file storage is to reduce the rate of fragmentation of documents and btree nodes.

This is because the rate at which documents and btree nodes become fragmented are highly affected by the batch size of documents (larger batches are better). The best case for batching is experienced with compaction, where all documents written to a new a file in a single batch, resulting in files that are sequentially packed together, and btree's nodes exhibiting ideal locality for random lookups and range scans.

When copying documents to a colder generation from hotter generation, the documents are batched in a single transaction, guaranteeing sequential packing of those new updates and excellent locality of the rewritten btree nodes, and minimizing wasted btree nodes.

Since this always happens in batches, this a big improvement over the single document non-batched updates that can be happening at the hottest level, which if happens on a single, monolithic storage file means each document is non-sequential to other updates, as each update places rewritten btree nodes and a header between each document. Single updates will also rewrite more btree nodes than batched updates.

## Partition Specific Meta-data

The smallest generation will contain the meta information pertaining to an entire partition, and will be read and stored in-memory on startup.

Statistics that will be stored in the partition header are total live documents, total deleted documents, total documents in conflict.

Each generation's file header will contain information about the total document count and size of live data and btree nodes in the generation. However, this data isn't useful in aggregate across generations, as data in hotter generations may overwrite or delete data in older generations. Per generation metadata is only useful for determining when to compact a generation due to fragmentation.

## Performing an Insert, Update or Deletion

All mutations to partition storage happen at the smallest generation. 

When performing a mutation, if the document's metadata isn't already in RAM cache, check each generation from the smallest to largest, first checking each generation's bloom filter, then fetching the document and meta data via the by\_id btree (which might not find a result due to a bloom filter false positive), repeating until it finds the first occurrence of the document, if it exists at all.

The most recent document metadata (the one in the smallest generation) will contain the revision tree history for all conflicts and be cached in memory. This meta data will contain:

* The revision history of each conflict
* For conflicts residing in the same file, the exact file offset to fetch the document.
* For conflicts residing in the colder generations, it will contain the conflict update seq.
* A flag to indicate if a conflict is deleted or live.

NOTE: Under many circumstances there will be no real conflicts. But consider even a single document with no other conflicts as a "conflict" in this context to make the algorithm easier to understand.

If using a CAS, the document mutation is the applied to the metadata using CAS to find the appropriate conflict. If it's not found or doesn't match, the document is rejected and an error is returned the client.

If the CAS matches or isn't used, the document and updated metadata is then written to the most recent WAL file. If the CAS is omitted, the mutation is applied to the "winner" conflict.

A background process then writes the updated document(s) and metadata into the smallest generation file. When the generation exceeds it's live data threshold, the coldest documents are copied to the next colder generation, the hotter documents are written to a new file of the same generation level.

### by\_seq Tombstone

If the mutation is an edit or deletion, the old by\_seq entry must be deleted. If the previous by\_seq entry is still in memory and WAL only and uncopied to generational storage, nothing needs to be done. If the previous generation is in generational storage or it's unknown if it's in generational storage (due to concurrent migration of the mutation from WAL to GS), then write a deletion "tombstone" for the by\_seq entry to be copied into generational storage.

The by\_seq tombstone will be copied and eventually be applied to the generation with the corresponding by\_seq entry, where it's then removed from the btree. If it makes it all the way to the coldest generation without finding it's corresponding by\_seq entry, the tombstone is dropped.

When walking the logical by\_seq index, all generation files will be walked simultaneously, and tombstones encountered in hotter generations will cause the actual entry a colder generation to be skipped over.

## Cumulative by\_id entry

The by\_id entry for the document in the hottest generation contains the cumulative revision tree history for all conflicts. Each generation with a version of the document also has a by\_id entry that is the cumulative history for all colder generations. This is necessary so when the conflict status or winner changes, we know it at update time and can update flags and rewrite winners into the by\_seq and by\_id entries.

### Updating Interim Winner and Conflict Markers

Each conflict in the database has its own by\_seq entry. Each by\_seq entry for a document should know if the document is in conflict, and if the particular version is the interim winner or a conflict loser. This is because indexes want to only index the interim winner, and may want to know if the document is in conflict to allow quickly finding documents in conflict via the secondary index.

For correctness purposes, it's good enough to ensure that interim winner always has a seq higher than any other conflict version, the indexer will overwrite any conflicts with the winner if it's always last. But for efficiency reasons, we'd like indexers to be able to skip conflict documents, to not waste resources computing and indexing on conflicts that will just be overwritten.

A problem is a conflicting version of a document might be a conflict loser, then the interim winner with a higher seq is deleted such that the conflict loser becomes the new winner. The indexer needs to see that old loser as a new winner. Another problem is that if the only conflict version is created or deleted (ie. resolved), we want to update the winner's by\_seq entry to indicate the document's updated conflict status.

That is solved with two different mechanisms:

1. For versions that were conflict losers and now are winners, we solve it by reassigning a new higher sequence to the winner and marking it as the winner and setting the conflict status flag. This ensures the indexers will see the correct document and conflict state. This will require rewriting the the new winner to the WAL with a new sequence.

2. For versions that previously weren't losers, but now are, we write a by\_seq "metadata update" entry into the WAL. It will eventually migrate to it's correct generation and be applied to the corresponding by\_seq entry it is updating. Until then, as the by\_seq index is walked, the btree in all generations is walked simultaneously with entries in hotter generations overriding colder generations. We will also already do this for by\_seq deletions.

## Single vs. Double fsync

### Reducing fsyncs

If for single or batched update, if we write all the data, btree updates and header and then do a single fsync, when the fsync successfully returns everything will correct on disk. 

However, there is a small possibility the OS will reorder how it durably writes the files to disk, and can write out the header before other data in the same commit. If a crash or powerloss occurs during this reordered writes, on restart it's possible the system will find and use this header that eventually points to incompletely written data, which results in the same problems a data corruption. It's possible a single document is corrupted, or the root of a btree, making it impossible to access any document in the database. It won't know the data is corrupted until sometime later when it tries to read these incomplete items.

A simple enhancement is use a single fsync scheme for a commit, and when a server opens a database file for the first time after startup, it scans and reads all data, meta, and btree nodes whose file location is _after_ the previous header, but _before_ the current header. If any data is detected as corrupt (fails the CRC32 check), is instead uses the previous header as the correct header and truncates the file directly after it, effectively detecting and removing the incomplete transaction.

The tradeoff is we now half the fsyncs, but we must scan all data and meta from the last transaction on startup. Fortunately all the data is in a single contiguous region on disk, greatly reduce the seek cost if the area is prefetched.

The single fsync enhancement for commits and headers can make commits to a storage file faster, with the cost that on start up, all contents written the last segment of a storage file, between the last and second to last header, must be checked for integrity.

But for generational storage, each commit to larger, colder generations is the result of a single large batched update from a smaller generation. If we have no need to read all the data on start-up on those files (like for 2.0 warmup), we will there be wasting unbounded amounts of disk IO and cpu and time checking the last commits of all generations, increasing startup time and reducing availability of a node. Therefore all generations, except perhaps the very youngest, should be use the double fsync method, with data fsync'd first, then the header.

## Alternative Scheme for Valid Header Finding

An alternative design that does not require 4k header markers to instead record the compaction version and file offset of the last valid header for every generational file in the WAL. The WAL will persist all the current valid headers forever, before an older WAL is deleted all the current headers must be persisted to the newer WAL file.

This will require the same number of fsyncs per commit (once to the storage file and once to the WAL).

For file systems that don't do completely honest fsyncs, is is new risk introduced by entirety of the data relying on the log files being available and uncorrupted. If the WAL is lost, all data in all partitions will be lost. For file systems that "lie" about fsync's this is possible on a machine crash, or when the file is . For example, we commit a new log file but the filesystem doesn't fsync the file inode or directory for the new file, losing the file contents.

To get around this, if we detect a WAL file is incorrect or missing, we can read the entire generational storage files from beginning to end, reading in each file's appended items, ignoring data items and noting headers until the end or the very first corrupt item (this requires everything in the file is integrity checked, which it currently is in Couchstore). We scan and we use the last encountered header as the correct header.

To make recovery work, it will also require us to truncate the generational storage files if on file open we detect there is garbage (data begun to be written but not completed) after the last valid header, so that we can continue to append to the end of the file and recover correct headers should the WAL become incorrect.

### Advantages

We no longer have to align the storage file header to any boundary, so we can pack headers contiguous with previous data with no gaps in between. This will save an average of 1/2 the block size (4k by default, so 2k on average) per commit. For larger generations, this will have small benefit as commits are infrequent and large.

We will also eliminate all marker "holes" that occur when items (data, metadata, btree nodes, etc) are written that spans a 4k boundary, we won't have to add or remove the boundary marker when writing or removing, potentially reducing memory copies.

### Disadvantages

The disadvantage (other than the additional engineering work to write file recovering code) are we now must fsync 2 different files to correctly commit data to a generation. On HDDs, this may be significantly slower on average than fsync'ing the same file twice, since it likely requires a physical seek to the different file.

## Compaction and Spillover

### Live Data Threshold Exceeded

As each storage generation keeps statistics about the size of all live btree nodes and documents, it's possible to know how much live data and metadata is in a file and if it's exceed the generation's threshold. If it does, the coldest documents will be copied to the next coldest generation and the remaining will be copied to a new storage file of the same generation.

We also copy all by\_seq tombstones and by\_seq updates to the next colder generation.

When this is done, the old file is still accessible until the all the documents are copied to the next colder generation and the current generation file is generated. It is then deleted and inaccessible to new readers, and current readers will still have access to the old file until they close their file descriptors, which then will cause the OS to completely unlink the file and recover the wasted space.

### Garbage Threshold Exceeded

To determine how much space is in a storage file is garbage is done simply by subtracting the live data size from the total file size. If the garbage exceeds the configurable ratio, then the file is compacted in place.

As with the Live Data Threshold, the coldest documents can be copied to the next colder generation. Or none at all. The amount of cold data copied to the colder generation during compaction should be configurable. Also all by\_seq tombstones and updates are always copied.

As with the Live Data Threshold process, the original file is kept around until all data is copied to the new file and colder generation, when it is then deleted.

## Bloom Filters

Each Generational Storage File will have a bloom filter generated from the hashes of all the doc ids/keys for a database. This is to improve the speed of disk fetches by avoid looking to files where the document doesn't exist.

When a write to a storage file occurs, the entire bloom filter is rewritten at the end of the file. It can be stored inside the header.

With the larger, colder generation files, the bloom filter can be very large. Approximately 9 bits per document stored for a bloom filter with 1% false positive rate. However, the cost of rewriting the whole header will generally be offset by the cost of the bulk writes to the file, since the cost each update will be due relatively large amount of documents migrating from hotter storage to colder. The colder the file, the larger the bulk update and the larger the bloom fliter, so the amortized cost of rewriting the bloom filter per doc migrated should remain consistent and relatively small at all levels.

### False Positive Degradation and Regeneration

As more unique documents are added to the file, the false positive rate of the bloom filter will increase. But when the file is compacted in-place, the entire bloom filter will be regenerated from scratch and brought up to maximal space v.s false positive efficiency.

## MVCC and Range Scans

When traversing the partition by\_seq all files will be scanned simultaneously and the btree index in each file will be walked simultaneously, and a merge sort will occur, with the hottest/newest entry used. The in-memory WAL data and snapshot-able in memory by\_seq entry (as specified in the UPR design specification) is also walked.

If duplicate entries are found, a common and normal occurrence, the entry from the hottest generation file will be used, with entries from colder generations discarded. If the hottest entry is a "metadata deletion" entry, all metadata for the same sequence will be skipped. If the hottest entries are "metadata update" entries, the updates are applied to the hottest concrete entry in order of coldest to hottest.

The process of opening and walking the storage will provide an MVCC snapshot of each file. To snapshot the entire database, the process of walking the indexes will open the files from order of hottest to coldest, an order that will ensure complete logical consistency with persistence state even as generation files are in-place compacted and spilled over to higher generations.

## Shrinking/Merging Generations

When the live data in the coldest generations N-1 and N drops below the live data threshold for generation N after spillover commit (due to deletion tombstones and shrinking edits migrating down), the data is merged into generation N or N+1 (whichever has less live data). If merged into N-1, then N is deleted. If merged into N, the file is renamed to be the new generation N-1 and the old N-1 is deleted.

## Conflict Management

## Differential Updates

### by_id Entry - Commulative

### by_seq Entry 











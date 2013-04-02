Generational Tail-Append Storage Design Specification
=======================================

# INTRODUCTION

## Purpose of this Document

This document describes the design, integration and implementation considerations of Generational Tail-Append Storage in Couchbase Server.

## Motivation

Generational Storage (GS) is an evolution of the tail append design of CouchDB and Couchstore (the Couchbase storage subsystem).

The CouchDB and Couchstore storage system uses a tail append design, where all updates to a database resulted in appending new or updated information to storage file. Even deletes would grow the storage files until compaction.

Some advantages of this design are:
* Extremely efficient and fast durable updates, minimizing disk head seeks on HDDs and minimizing write amplification on SSDs.
* Fast and simple recovery from crashes, simply scan backwards from the end of the file to the first good header and start using the file.
* Zero cost MVCC snapshots. Simply use the most recent version of a file header and get a completely consistent storage file.

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

### All writes are appends

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

### Reducing fsyncs

If for single or batched update, if we write all the data, btree updates and header and then do a single fsync, when the fsync successfully returns everything will correct on disk. 

However, there is a small possibility the OS will reorder how it durably writes the files to disk, and can write out the header before other data in the same commit. If a crash or powerloss occurs during this reordered writes, on restart it's possible the system will find and use this header that eventually points to incompletely written data, which results in the same problems a data corruption. It's possible a single document is corrupted, or the root of a btree, making it impossible to access any document in the database. It won't know the data is corrupted until sometime later when it tries to read these incomplete items.

A simple enhancement is use a single fsync scheme for a commit, and when a server opens a database file for the first time after startup, it scans and reads all data, meta, and btree nodes whose file location is _after_ the previous header, but _before_ the current header. If any data is detected as corrupt (fails the CRC32 check), is instead uses the previous header as the correct header and truncates the file directly after it, effectively detecting and removing the incomplete transaction.

The tradeoff is we now half the fsyncs, but we must scan all data and meta from the last transaction on startup. Fortunately all the data is in a single contiguous region on disk, greatly reduce the seek cost if the area is prefetched.

## MVCC

When updates occurs, any existing on-disk data and metadata aren't modified. This means MVCC snapshotting of the entire database file is automatic and essentially free, requiring no explicit locks or resources beyond what a single threaded access require, as reads to the database can happen concurrently with writes, with no hard limit on the number of concurrent read snapshots. With a shared file cache, each concurrent reader often uses less resources per-snapshot (file system cache memory and disk fetches).

The efficiency of snapshotting is particularly important when performing range scans of by\_seq index for UPR, which requires UPR clients (backup utlities, indexers, replicators, etc) are able to get a consistent and complete snapshot to ensure fast recovery points in failover scenarios.

## Compaction

Compaction is the process of recovering wasted space in a storage file and arranging all the live data in a and btrees to be optimized for size and arrangement. It copies all data and btree indexes from the old file to a new file. It does this without without stopping or pausing persistence and fetches, but can result in duplicate IO for writes that occurred during the compaction.

The compaction doesn't block reads or writes, letting them proceed normally, though possibly slowed through increased io bandwith of the copying process.

The compaction process scans the by\_seq index and load all the live documents from the database and copying them into a new file. It creates a by\_seq index bottom up in the new file by building the node the leafs up to the root node, the sorts the same information by key/id with on-disk heap sort, and then builds up the new by\_id btree from the bottom up.

Once it's completed a snapshot, it checks to see if it's caught up with the old storage file, as new mutations could have been committed to it during the compaction. It then walks the by_seq index of the old file, picking up from the sequence where it left off, copying values into the new database file.

It repeats this process until it's fully caught up with all mutations to the main file. It's theoretically possible the updates to the main file happen faster than compaction can keep up with, but this has never been observed in practice.

### Shortcomings

In CouchDB, all documents for a database are written to a single storage file, meaning compaction requires at least as much free diskspace as the newly compacted file will require once complete, which must hold the entire data set. This means a filesystem without a fairly large amount of spare storage cannot complete a compaction.

In Couchbase 2.0 all documents are written to a partition specific storage file, and with 1024 partitions it effectively solves the free space problem with compaction as it is now far more incremental (each unit of compaction work is 1/1024 of the total database) and therefore needs much less temporary disk space or memory.

It also makes it easy to read, write or delete many or all documents from a partition more efficiently, as is the case for rebalances. Documents grouped by partition have better locality for scans, writing in bulk doesn't cause extra IO in other partitions, and deleting a partition is simply deleting the storage file as a single operation.

But the Couchbase storage this introduces a new problem, to durably committing a large number of documents to different paritions requires at a minimum 1 fsync per partition, greatly increasing latency for a durable commit.

Also, there is worst case of compaction, where a single document in a large partition is updated repeatedly. Recovering the garbage space from the updates means copying all documents in the partition to a new file, even the documents that haven't been updated in a very long time. The cost of compaction is N for N total documents in the partition, even though only 1 document is being updated. The cost of recovering the space for 1 document is O(N\*log(N)), where N is the total number of documents.

The best case for compaction for M different document updates when there is a even distribution of single updates across all documents. In this best case, the cost of recovering the space for each of M documents is O((N/M)\*log(N)), where N is the total number of documents.

# Generational Storage

Generational Append-Only Storage is a new design that is heavily based on the current Couchstore design, and addresses many of the shortcomings, particularly for interactive workloads.

It has been empirically observed that most interactive workloads have approximations of a Zipfian/Zeta distribution of retrievals and updates to documents. This means the most popular document is retrieved/updated 2x more than the next most popular, 3x more than the next after that, 4x more the one after that, and so one.

The generational storage system attempts to group documents into generations of "hotness", where the most updated documents stay in the smallest, hottest and most frequently compacted generation, colder documents tend  to migrate down to the next exponentially larger, colder generation (which because it's updated less frequently is compacted less frequently), and the colder still documents migrate down to the next exponentially larger, colder, less frequently compacted generation, and so on.

When workloads where many documents have widely ranging update frequencies (such as Zipfian) the frequency of update of a document (it's "hotness") determines the generation the document migrates to, and how therefore how frequently it gets compacted.

Some key properties we want to maintain when working with a interactive/Zipfian workloads:

* Consistent low latency durable writes/updates and fetches
* Storage immune to corruption due to process/OS crash or powerloss
* Fast startup and warmup time
* High throughput, restartable by\_seq scans when indexing, replicating, rebalancing or backing-up a large amounts of changes
* Partition level MVCC for reads and consistency of indexing and replication (necessary for UPR)
* Efficient partition rebalance in and out.
* Cost of rebalancing a partition onto and off of a node is the same regardless the number/size of other partitions on a node
* Reduce the overhead necessary for compaction.
* Pauseless compaction (files are compacted, reorganized, and defragmented without pausing updates or fetches)
* Low diskspace overhead (disk space usage isn't dominated by internal fragmentation or index overhead)

# System Architecture

## Write Ahead Log

All commits to storage are written to an index-less, tail-append write ahead log (WAL) file. There are a series of these files numbered from 0 and up (0.log, 1.log ... _N_.log), and all new mutations happen to the highest numbered file. Until the data is moved to partition specific storage, the most recent update to a document is kept in memory (as dirty) and cannot be ejected. However, once written and persisted to the log, a client waiting for the write to be durable will get the success response from the server.

On startup, the WAL file or files are read from beginning to end into memory, with subsequent updates to the same document replacing in the in-memory representation of the earlier mutation. These newly read document are still marked dirty until they are written and committed to partition specific storage.

The advantage of the WAL is to reduce the latency of a durable commits, as it only has to wait on a single fsync of a sequential write for all partitions, instead of an fsync per partition, which can increase latency dramatically.

The downside is the every document commit in the worst case will written to disk at least 2 times, once to the log and at least once in the partition specific generational storage, increasing disk contention and reducing throughput from it's theoretical maximum. For documents that are updated constantly, the number of times a write is rewritten can be much smaller, as only the most recent commit is copied the partition storage, the rest are ignored.

## Async Copy from WAL to Storage

Asynchronous to WAL commits, a separate process or thread will copy the most recent versions of data from memory to the youngest generation of partition specific storage. When this process starts, writes to the most recent (highest numbered) log file cease and new writes happen to a new high numbered log file. Once the partition copy process has copied all the live data from that log file to their specific partition storage, that log file is deleted. The process then begins again with next highest numbered log file, until no more mutations remain.

### Natural Throttle

While it is copying files from the log to storage, it might need to recopy data from a younger generation to an older generation and/or in-place compact any of the generational files. When this happens, the copy/compaction process may get behind the update/insert rate of the log files. If this happens for too long, all the bucket memory quota will be dedicated to "dirty" items (even though they are safe on disk) and it will pause updates. which as the bucket will exceed it's memory quota therefore will not be able to accept any client writes, until the  

## File versioning



## Performing an Insert

## Performing an Update

## Single fsync

## Live Data Threshold


### Growth and copying

## MVCC

## Range Scans

## Bloom Filters

## Copy Oldest to Next Generation

## Compaction

### Throttle Updates

## Shrinking/Merging Generation


## Conflict Management

### by_id Entry - Commulative

### by_seq Entry 











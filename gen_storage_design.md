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

"Write amplification" is the additional low level storage blocks that must be rewritten for any successful write on a SSD. Sequential writes/tail will minimize write amplification, random internal writes tend to maximize it. Lowering write amplification increases SSD write performance and operational life.

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

Unlike update in-place btrees, Couchstore btree's do not self balance. So long as all updates are inserts, they are guaranteed to remain balanced. But certain update patterns (value resizing and deletes) can cause them to become unbalanced. However, the process of database compaction will regenerate the btrees from the ground up and optimally rebalance them.

## Crash/Powerloss tolerant

Because of the tail-append nature, any update to that fails to complete will also fail to write a valid header. When the file is reopened, the incompletely written data is seen as garbage at the end of the file and skipped over until the system finds the first valid header, and all data preceding it will be valid, assuming the physical media has not corrupted.

## MVCC

When updates occurs, any existing on-disk data and metadata aren't modified. This means MVCC snapshotting of the entire database file is automatic, as reads to the database can happen concurrently with writes, with no hard limit on the number of concurrent read snapshots. This is particularly important when performing range scans of by\_seq index for UPR, which requires UPR clients (backup utlities, indexers, replicators, etc) are able to get a consistent and complete snapshot to ensure fast recovery points in failover scenarios.

## Compaction


## Shortcomings

### Filesystem Cache

### Compaction

In CouchDB, all documents are written to a single storage file, meaning compaction requires at least as much free diskspace as the newly compacted file will require once complete. This means a filesystem without a fairly large amount of spare storage cannot complete a compaction.

In Couchbase 2.0 all documents are written to a partition specific storage file, and with 1024 partitions it effectively solves the free space problem with compaction as it is now far more incremental and needs much less space storage space. But this introduces a new problem where durably committing a large number of documents requires a large number of fsyncs, as each partition must be fsync'd before all the documents are considered committed.

Best case

Worst case

## Double FSync

## Generational Storage

# System Architecture

File versioning

## Performing an Insert

## Performing an Update

## Single FSync

## Live Data Threshold

## Write Ahead Log

### Growth and copying

## Async Copy from WAL to Storage

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











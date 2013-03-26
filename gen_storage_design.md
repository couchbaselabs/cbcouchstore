Generational Tail-Append Storage Design Specification
=======================================

# INTRODUCTION

## Purpose of this Document

This document describes the design, integration and implementation considerations of Generational Tail-Append Storage in Couchbase Server.

## Motivation

Zeta distribution
data locality

## Scope of this Document

#Background and Current Implementation

## Tail Append

## MVCC

## Compaction

## Shortcomings

### Filesystem Cache

### Compaction

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

## Shrinking


## Conflict Management

### by_id Entry - Commulative

### by_seq Entry 











# Specialized Data Structures & Memory Management

This document explores the trade-offs of caching, persistent storage formats, and probabilistic data structures used in the high-load monitoring system.

## 1. The Real Cost of Caching
Caching is not "free performance." It introduces significant architectural complexity:
* **Cache Inconsistency:** Managing stale data relative to the Source of Truth (PostgreSQL) is a major challenge.
* **Cache Warming & Stampede:** A cold start after a system reboot can overwhelm the database as all requests miss the cache simultaneously.
* **Resource Overhead:** RAM is significantly more expensive than disk space. Inefficient caching leads to inflated infrastructure budgets.
* **Reliability:** The system must handle "Cache Down" scenarios without cascading failures.

## 2. Disk Persistence: Why HashMap is Not Ideal
Persisting a **HashMap** directly to disk is generally inefficient for high-load systems.
* **Random I/O Bottleneck:** HashTables are optimized for RAM. On disk, this leads to non-sequential access, which drastically reduces performance even on modern SSDs.
* **Fragmentation:** Frequent updates/deletes leave "holes" in disk blocks, requiring expensive rehashing or compaction operations.
* **Alternative:** We prefer **B-Trees** (PostgreSQL) or **LSM-Trees** (Cassandra/RocksDB), which are optimized for block-based reading and sequential writing.

## 3. Probabilistic Structures: When NOT to use Bloom Filters
A Bloom Filter provides a definitive "No" or a probabilistic "Maybe." We avoid it in the following scenarios:
* **Zero-Tolerance for False Positives:** If a "Maybe" result cannot be cross-checked with a database and might lead to a critical error.
* **Element Deletion Required:** Standard Bloom Filters do not support deletion. For this, more complex *Cuckoo Filters* or *Counting Bloom Filters* would be necessary.
* **Small Data Volumes:** For small datasets, a standard `Set` in memory is faster and more accurate. Bloom Filters only provide value at the scale of billions of records.

## 4. RAM vs. Disk: Comparative Analysis

| Characteristic | RAM (In-Memory) | Disk (SSD/HDD) |
| :--- | :--- | :--- |
| **Access Latency** | Nanoseconds (Ultra-fast) | Milliseconds (Slow) |
| **Volatility** | Volatile (Lost on power-off) | Persistent |
| **Cost** | High price per GB | Low price per GB |
| **Access Pattern** | Uniform speed for any cell | Sequential is much faster than random |

## 5. SSTables & LSM-Trees: The Read/Write Trade-off
SSTables (Sorted String Tables) are the backbone of many high-performance databases, but they come with a major drawback: **Read Amplification**.

* **The Problem:** Data is written in immutable segments. A single record might exist in multiple layers (an old version in one file, a new one in another).
* **Impact:** To find the latest value, the system must search through multiple disk segments and merge them on the fly.
* **Maintenance:** Frequent background **Compaction** is required to merge files, which creates high disk pressure (**Write Amplification**).

## 6. Implementation in PetHealth Monitor
* **Go Ingestor:** Uses a local **LRU (Least Recently Used) Cache** with a fixed memory limit to prevent OOM (Out of Memory) errors.
* **Redis:** Leverages optimized in-memory hashes to avoid the overhead of disk-based lookups during 2,000+ RPS ingestion.
* **TimescaleDB:** Uses **B-Tree indexes** for time-based partitioning, ensuring efficient sequential access to telemetry history.
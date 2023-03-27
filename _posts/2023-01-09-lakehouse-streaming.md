---
layout: post
title:  How to build a streaming lakehouse with very high data freshness using debezium, kafka and hudi?
date:   2023-01-09
tags: [kafka, distributed-systems, hudi, debezium]
splash_img_source: /assets/img/lakehouse_cover.png
splash_img_caption: Lakehouse overview.
---

A data lakehouse is a new, big-data storage architecture that combines the best features of both data warehouses and data lakes. A data lakehouse enables a single repository for all your data (structured, semi-structured, and unstructured) while enabling best-in-class machine learning, business intelligence, and streaming capabilities.

Data lakehouses usually start as data lakes containing all data types; the data is then converted to [Delta Lake](https://delta.io/) format (an open-source storage layer that brings reliability to data lakes). Delta lakes enable ACID transactional processes from traditional data warehouses on data lakes.

Using data lakehouse reduces data redundancy, makes the process cost effective, allows support for a wider variety of workload, enhances data governance, security and 

<img src="/assets/img/lakehouse_streaming_2.png" alt="lakehouse">

# Basics

## Why use data lakehouse?

### Traditional data lake

A data lake is a centralized, highly flexible storage repository that stores large amounts of structured and unstructured data in its raw, original, and unformatted form. In contrast to data warehouses, which store already “cleaned” relational data, a data lake stores data using a flat architecture and object storage in its raw form. Data lakes are flexible, durable, and cost-effective and enable organizations to gain advanced insight from unstructured data, unlike data warehouses that struggle with data in this format.

<img src="/assets/img/lakehouse_streaming_3.png" alt="lakehouse">

In data lakes, the schema or data is not defined when data is captured; instead, data is extracted, loaded, and transformed (ELT) for analysis purposes. Data lakes allow for machine learning and predictive analytics using tools for various data types from IoT devices, social media, and streaming data.

Daily snapshots of the tables in RDBMS are taken. The read/write amplifications are usually high and there are dedicated replicas to isolate snapshot queries. Despite that the latency is usually high (more than 24hrs).

<img src="/assets/img/lakehouse_streaming_4.png" alt="lakehouse">

**Requirements:**

- Transaction support
- Scalable Storage and compute
- Openness
- Direct access to files
- End-to-end streaming
- Diverse use cases

## What is change-data-capture (CDC)?

Change data capture (CDC) refers to the process of identifying and capturing changes made to data in a database and then delivering those changes in real-time to a downstream process or system.

<img src="/assets/img/lakehouse_streaming_5.png" alt="lakehouse">

1. Each CRUD operation streamed from DB to Subscriber:
The CRUD (Create, Read, Update, Delete) operations are commonly used in database management systems. In a streaming context, each CRUD operation refers to a change in the data that is being streamed. In this context, "streaming" refers to the continuous flow of data from the database to a subscriber, who is interested in receiving updates to the data.
    
    When a CRUD operation is performed on the database, the change is immediately sent to the subscriber, who can then process the update as needed. This allows subscribers to receive real-time updates to the data they are interested in, without needing to constantly poll the database for changes.
    
    Streaming CRUD operations from the database to subscribers can be accomplished through the use of change data capture (CDC) technologies or real-time event processing systems.
    
2. Merge changes to lake house:
In a data lake architecture, a "lake house" is a term used to describe a hybrid system that combines the benefits of a data lake with a traditional data warehouse. A lake house typically consists of a raw data lake layer, a data refinement layer, and a query and analytics layer.
    
    When changes occur in the data source (such as a database), they are typically captured and stored in the raw data lake layer. These changes may be in the form of new records, updates to existing records, or deletions of existing records.
    
    The process of merging changes to the lake house involves taking these raw changes and applying them to the appropriate data refinement layer. This may involve transforming the raw data into a more structured format, performing data quality checks, and applying business rules and logic.
    
    Once the data has been refined, it can be queried and analyzed by users. The process of merging changes to the lake house is critical for ensuring that the data is up-to-date and accurate.
    
3. Efficient & Fast -> Capture and Apply only deltas:
Capturing and applying only deltas refers to a data management technique that involves tracking changes to data, rather than storing multiple copies of the same data. In this technique, only the changes (or "deltas") are stored, rather than the entire data set.
    
    This approach can be more efficient and faster than traditional data management techniques, because it reduces the amount of data that needs to be stored and processed. Instead of storing multiple copies of the same data, only the changes are stored, which can save significant storage space and processing power.
    
    Capturing and applying only deltas can be used in a variety of contexts, such as database management, data replication, and data synchronization. It is particularly useful in scenarios where large amounts of data are being updated frequently, such as in real-time streaming systems or high-volume transactional systems.
    

## What is Debezium?

Debezium is an open source distributed platform for change data capture. It has distributed Kafka-Connect Service for change data capture. It supports CDC from diverse RDBMS (Postgres, MySQL, MongoDB, etc.) and has pluggable Sinks through Kafka.

<img src="/assets/img/lakehouse_streaming_6.png" alt="lakehouse">

Debezium has numerous advantages over its competitors:

<img src="/assets/img/lakehouse_streaming_7.png" alt="lakehouse">

1. **Postgres Primary Dependency**
ONLY the Postgres Primary publishes WriteAheadLogs (WALs).
**Disk Space:**
    - If a consumer dies, Postgres will keep accumulating WALs to ensure Data Consistency
    - Can eat up all the disk space
    - Need proper monitoring and alerting
    
    **CPU:**
    
    - Each logical replication consumer uses a small amount of CPU
    - Postgres10+ uses pgoutput (built-in) : Lightweight
    Postgres9 uses wal2Json (3rd party) : Heavier
    
    <img src="/assets/img/lakehouse_streaming_8.png" alt="lakehouse">
    
2. **Initial Table Snapshot (Bootstrapping)**
**Need for bootstrapping:**
    - Each table to replicate requires initial snapshot, on top of which ongoing
    logical updates are applied
    
    **Problem with Debezium:**
    
    - Debezium processes data row at row level
    - Large tables are slow
    - Too much pressure on Kafka Infrastructure and Postgres primary
    
    **Solution using Hudi Deltastreamer:**
    
    - Custom bootstrapping framework using partitioned and distributed spark
    reads
    - Can use read-replicas instead of the master
    
    <img src="/assets/img/lakehouse_streaming_9.png" alt="lakehouse">
    
3. **AVRO vs JSON**
<img src="/assets/img/lakehouse_streaming_10.png" alt="lakehouse">
4. **Multiple logical replication streams for horizontal scaling**
    - Multiple large tables can overwhelm a single Debezium connector
    - Split the tables across multiple Debezium connectors
    `Total throughput = throughput_per_connector * num_connectors`
    - Each connector does have small CPU cost
    
    <img src="/assets/img/lakehouse_streaming_11.png" alt="lakehouse">
    
5. **Schema evolution and value of Freezing Schemas**
    - **Failed assumption: Schema changes are infrequent and always backwards compatible.**
        
        Examples:
        
        1. Adding non-nullable columns (Most Common 99/100)
        2. Deleting columns
        3. Changing data types
        4. Can happen anytime during the day #always_on_call
    - **How to handle the non backwards compatible changes?**
    Re-bootstrap the table
    - **Alternatives? Freeze the schema**
        - Debezium allows to specify the list of columns per table.
        - Pros:
            - #not_always_on_call
            - Batch the changes for management window
        - Cons:
            - Schema is temporarily out of sync
    
    <img src="/assets/img/lakehouse_streaming_12.png" alt="lakehouse">
    

## What is Apache Hudi?

- Transactional Lakehouse pioneered by Hudi
- Serverless, transactional layer over lakes.
- Multi-engine, Decoupled storage from engine/compute
- Upserts, Change capture on lakes
- Introduced Copy-On-Write and Merge-on-Read
- Ideas now heavily borrowed outside

### Hudi: Upserts and Incrementals

<img src="/assets/img/lakehouse_streaming_13.png" alt="lakehouse">

### Hudi: Storage Layer

<img src="/assets/img/lakehouse_streaming_14.png" alt="lakehouse">

### Hudi: Copy-on-write table

<img src="/assets/img/lakehouse_streaming_15.png" alt="lakehouse">

### Hudi: merge-on-read table

<img src="/assets/img/lakehouse_streaming_16.png" alt="lakehouse">

# Architecture of our streaming lakehouse

## Basic architecture

<img src="/assets/img/lakehouse_streaming_17.png" alt="lakehouse">

## Data Lake Ingestion: CDC Path

<img src="/assets/img/lakehouse_streaming_18.png" alt="lakehouse">

## Data Lake Ingestion: Bootstrap path

<img src="/assets/img/lakehouse_streaming_19.png" alt="lakehouse">

## Conclusion and outcome:

With tables in the scale of 1000s, **proper automation and self healing, provisioning and isolation based on tiered SLAs, pre-commits and validation for quality checks, monitoring & alerting**, the freshness of data using the above architecture can brought down from **24hrs to 15mins.**

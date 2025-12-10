---
title: "Compute Engine Interoperability"
---
<!--
 - Licensed to the Apache Software Foundation (ASF) under one or more
 - contributor license agreements.  See the NOTICE file distributed with
 - this work for additional information regarding copyright ownership.
 - The ASF licenses this file to You under the Apache License, Version 2.0
 - (the "License"); you may not use this file except in compliance
 - with the License.  You may obtain a copy of the License at
 -
 -   http://www.apache.org/licenses/LICENSE-2.0
 -
 - Unless required by applicable law or agreed to in writing, software
 - distributed under the License is distributed on an "AS IS" BASIS,
 - WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 - See the License for the specific language governing permissions and
 - limitations under the License.
 -->

# Open data lake interoperability guide

This guide summarizes the interfaces and behaviors that a compute engine should expose
to work correctly with open table formats. While examples are framed from Iceberg's
perspective, the checklist is intended to cover capabilities common to Iceberg, Delta
Lake, and Hudi so engines can plan for superset support. Engines can map these
capabilities to their own connectors or catalog implementations to ensure predictable
reads, writes, and metadata management across formats.

## Catalog and namespace resolution

* Support catalog plugins that can resolve table identifiers, namespaces, and views.
* Provide create, drop, alter, list, and rename for namespaces and tables with atomic
  semantics where possible.
* Surface catalog-level authorization and delegation to object storage credentials.
* Expose table property management (owner, retention, encryption, storage location).

## Table metadata, snapshots, and time travel

* Read table metadata without scanning data files, including schema, partition spec,
  sort/order information, and table properties.
* Select a snapshot by ID, timestamp, or branch/tag reference for consistent reads.
* Refresh table metadata lazily and eagerly to avoid stale planning in long-running jobs.
* Support optimistic concurrency control during commits with conflict detection.

## Read path requirements

* Predicate pushdown that uses partition transforms, column statistics, and delete files
  to prune data and minimize I/O.
* Column projection and type promotion to respect column IDs and field-level metadata.
* File-format coverage for Parquet, ORC, and Avro with vectorized and columnar readers.
* Position- and equality-delete handling so row-level deletes are applied during scans.
* Incremental planning that can scan append-only changes or change logs between snapshots.
* Configurable file I/O to work with cloud object stores, HDFS, and encrypted files.

## Write path and transaction model

* Atomic append, dynamic overwrite, and replace-partitions semantics with snapshot isolation.
* MERGE/UPDATE/DELETE that writes appropriate data and delete files instead of in-place edits.
* Idempotent commit protocol with retry-safe staging and manifest rewrite where required.
* Distributed lock coordination or serializable commits to avoid lost updates.
* Validation hooks for schema/partition compatibility and concurrent write detection.

## DDL and evolution

* Schema evolution that preserves column IDs and supports add, drop, rename, reorder,
  type widening, and default values.
* Partition spec evolution without rewriting existing data; engines must track and preserve
  transform definitions such as identity, bucket, truncate, year/month/day/hour.
* Table relocation, property updates, and storage profile changes without data corruption.
* Compatibility checks to ensure evolution choices remain readable for existing data
  files and previously written delete files.

## Streaming and incremental processing

* Source readers that can start from a specific snapshot or timestamp and consume incremental
  changes with exactly-once or at-least-once guarantees.
* Streaming sinks that commit checkpoints to table snapshots, including epoch/offset mapping.
* Handling of late data and watermarking while keeping manifest and metadata refresh in sync.
* CDC-friendly options (e.g., emitting inserts/updates/deletes) derived from snapshot diffs.

## Maintenance and optimization hooks

* Procedures or commands to expire snapshots, remove orphan files, rewrite manifests, and
  compact small files or recluster data.
* Exposure of maintenance metrics so scheduling systems can prioritize large or skewed tables.
* Cache invalidation hooks for metadata and data caches after maintenance operations.

## Governance, security, and compliance

* Integrate with catalog- or engine-level access control (role-based, attribute-based, or
  policy-tag driven).
* Support encryption settings (table, file, and key management), plus secure handling of
  credentials for object storage.
* Row-level and column-level security integration that is evaluated after delete files are applied.
* Auditing and lineage emission for commits, including actor, application ID, and change summary.

## Observability and diagnostics

* Metrics and tracing for planning, file reads, and commits (I/O volume, filtered rows, retries).
* Explain/plan surfaces that show snapshot ID, filters used for pruning, and delete-file application.
* Debug tooling to inspect manifests, data files, and deletes referenced by a given snapshot.

## Compatibility and extension points

* Pluggable file I/O, credential providers, and encryption modules to adapt to cloud or on-prem
  environments.
* Cross-format interoperability via catalog APIs that expose table capabilities so engines can
  degrade gracefully (for example, disabling update/delete when the format lacks row-level deletes).
* Clear feature negotiation so applications know which operations are supported before execution.

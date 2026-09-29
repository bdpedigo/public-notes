# Proposal: A Catalog for Connectomics Data in Cloud Buckets

**Status:** Draft for discussion

## Problem

Most connectomics data lives as files in cloud buckets: precomputed segmentations, skeletons, meshes, Neuroglancer annotations, Parquet/Delta dumps of CAVE tables, and Lance tables of embeddings such as SegCLR. Currently the metadata about these various products is stored in disparate locations.

## Proposal

Build one catalog that holds **metadata about data in cloud buckets**, not the data itself. The core schema is general enough to describe any asset. Type-specific subclasses add richer structure where it makes sense.

## Core concepts

### Asset

An asset is a named, typed pointer to a cloud location plus metadata. Every asset resolves to a URI, usually a prefix such as a precomputed directory or a Delta table root rather than a single file. Every asset has:

- **Identity:** datastack, name, and version (see below)
- **Location:** URI, and whether the location is managed (see below)
- **Kind and format:** for example `table/delta`, `table/lance`, `volume/precomputed`, `annotations/precomputed`, or `skeletons/precomputed`
- **Ownership and lifecycle:** owner, maturity (`experimental` → `stable` → `deprecated`), and an optional expiration
- **Description:** free text and tags, for humans and search
- **Properties:** kind-specific metadata, validated against a schema for that kind

### Managed vs. external locations

- **Managed:** The catalog controls the bucket or prefix. Access follows CAVE permissions: datastack and group membership. Readers get short-lived, downscoped credentials that are scoped to the asset's prefix and read-only. They never receive bucket-wide keys. Note that this does not mean the catalog controls anything about the data that is there.
- **External:** The catalog only points to the location. It doesn't grant access or guarantee that the data persists. This is useful for public buckets.

The catalog stores the same metadata for both. The only difference is whether it can hand out credentials.

### Versions and time

Most assets describe the segmentation at a specific moment, so time is a shared, first-class concept rather than something each asset handles on its own.

- **Series vs. version:** A *series* is a logical asset, for example `synapses` on `minnie65_public`. A *version* is one immutable instance of it at a specific location.
- **Time anchor:** Each version can record the segmentation time it reflects. It stores a timestamp and, if one exists, the CAVE materialization version. The materialization version is a convenient alias; the timestamp is the canonical value. Some assets, such as imagery, have no time anchor.
- **Revision:** This counter separates re-exports of the same series at the same time anchor, for example after a bug fix.
- **Mutability:** Each version is marked `static` (never changes after registration) or `live` (appended or overwritten in place). Only static assets can make strong reproducibility guarantees.

### CAVE semantics

The catalog should understand CAVE concepts inside assets, not just record their file format.

- **Tables:** Columns can be annotated with a semantic kind, such as `node_id` (chunkedgraph), `position` (with voxel resolution).
- **Non-tables:** Assets can declare what they are keyed by. For example, "skeletons keyed by root ID at T" or "segmentation whose labels are root IDs at T."

These annotations make it possible to find every asset that contains root IDs valid at version 1412, and they tell clients when IDs need to be mapped to another time.

### Kind hierarchy

```
Asset
├── Table        (columns, row count, partitioning, CAVE column semantics)
│   ├── Parquet / Delta / Lance / ...
├── Volume       (precomputed image or segmentation: bounds, resolution, scales)
├── Annotations  (Neuroglancer precomputed annotations: type, properties, relationships)
├── Skeletons / Meshes  (keyed-by, format specifics)
└── Other        (generic fallback: URI and properties only)
```

## Easy ingest

Where possible, the catalog extracts kind-specific metadata automatically from the data, for example a Delta log, a precomputed `info` file, or a Lance manifest. Users don't have to type it in.

There is also a UI that makes doing these things (likely just admins) easy, and programatic input via the endpoints/CAVEclient too.

## Browsing and querying

The catalog should support these views, which should be cheap to build:

- **By series:** Show every version of this table over time.
- **By snapshot:** Show everything known for a datastack at a given version or timestamp.
- **By kind or format:** Show all precomputed annotation layers for a datastack.
- **By semantics:** Show tables with a `root_id` column, or assets keyed by nucleus ID.
- **By owner, tag, or maturity:** Show a user's assets, or stable assets only.

Each asset page should also answer the next question: how to read the asset in Python (a CAVEclient snippet) and, for Neuroglancer-compatible kinds, how to open it in Neuroglancer (a generated layer or link).

## Other things worth including

- **Provenance:** "Derived from" links to other assets or to a CAVE source table, plus an optional code or commit reference. This lets people follow where data came from and find downstream assets when the source changes.
- **Registration validation:** When an asset is registered, check that the URI exists, that the format matches, and that the caller has write access to the target datastack.
- **Stable identifiers:** Give each asset an ID that can be cited in papers and notebooks and that survives renames.
- **Client integration:** Add `client.catalog` to CAVEclient for search, returning either metadata or a ready-to-use reader with credentials attached.

## Non-goals

- Storing or copying data. The catalog holds pointers and metadata only.
- Running compute or queries against assets.
- Replacing the Materialization Engine. The catalog records exports and derivatives of it.

## Open questions

1. What is the unit of versioning for assets that aren't anchored to a time? Is `revision` alone enough?
2. Should external assets be allowed to claim `static` mutability, given that the catalog can't enforce it?
3. How far should the automatically extracted metadata go for each kind before it becomes a maintenance burden?
4. Should the catalog span datastacks (one global service) or run as one instance per deployment?
5. Retention: do expired managed assets get deleted, or only hidden?
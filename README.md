
ReGrid Spec
===========

This is the official spec for the [ReGrid](https://github.com/internalfx/regrid) Nodejs library.

ReGrid is meant to be an open specification. You are encouraged to implement a compatible driver in your chosen language.

- Authors [**Bryan Morris**](https://github.com/internalfx), [**Brian Chavez**](https://github.com/bchavez)
- Advisors [**Daniel Mewes**](https://github.com/danielmewes)

ReGrid is a method of storing large files inside a RethinkDB database.

### Features

- **Reliable** - Files are replicated across the cluster, benefiting from RethinkDB's automatic failover.
- **Scalable** - Easily store large files in RethinkDB, distributed across the cluster.
- **Consistent** - Sha256 hashes are calculated when the file is written, and verified when read back out.

### Contents

 - [Document Structure](#document-structure)
   - [Files Record](#files-record)
   - [Chunks Record](#chunks-record)
 - [API](#api)
   - [Contructor(connectionOptions, bucketOptions)](#new-regridconnectionoptions-bucketoptions)
   - [initBucket()](#bucketinitbucket)
   - [createWriteStream(filename, options)](#bucketcreatewritestreamfilename-options)
   - [createReadStreamById(file_id)](#createreadstreambyidfile_id)
   - [createReadStreamByFilename(filename, options)](#createreadstreambyfilenamefilename-options)

### Overview

When a file is written to ReGrid, a **files** record is written to a **files table**. Then the file is broken up into **chunks** which are written as separate records in a **chunks table**. Once all the chunks are written, the **files** record is updated to show that the file is `Complete`. The file is now ready for read operations.

### Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

Document Structure
==================

#### Files record

```json
{
  "id" : "<String>",
  "length" : "<Number>",
  "chunkSizeBytes" : "<Number>",
  "finishedAt" : "<Time>",
  "startedAt" : "<Time>",
  "deletedAt" : "<Time>",
  "sha256" : "<String>",
  "filename" : "<String>",
  "status" : "<String>",
  "metadata" : "<Object>"
}
```

| Key | Description |
|---|---|
| id | a unique ID for this document. |
| length | the length of this stored file, in bytes. |
| chunkSizeBytes | the size, in bytes, of each data chunk of this file. This value is configurable by file. The default is 255KB (1024 * 255). |
| finishedAt | the date and time this file finished writing to ReGrid. The value of this field MUST be the datetime when the upload completed, not the datetime when it was begun. |
| startedAt | the date and time this file started writing to ReGrid. The value of this field MUST be the datetime when the upload started, not the datetime when it was finished. |
| deletedAt | the date and time this files status was set to `Deleted`. The value of this field MUST be the datetime when file was marked `Deleted`. |
| sha256 | SHA256 checksum for this user file, computed from the file’s data, stored as a hex string (lowercase). |
| filename | the name of this stored file; this does not need to be unique. |
| status | Status may be "Complete" or "Incomplete" or "Deleted". |
| metadata | any additional application data the user wishes to store. |

#### Chunks record

```json
{
  "id": "<String>",
  "file_id": "<String>",
  "num": "<Number>",
  "data": "<Binary>"
}
```

| Key | Description |
|---|---|
| id | a unique ID for this document. |
| file_id | the id for this file (the id from the files table document). |
| num | the index number of this chunk, zero-based |
| data | a chunk of data from the user file |

API
===

**Note:** Code examples are offered to give a sense of the API design of ReGrid. Adapt to your chosen language as necessary.

### `new ReGrid(connectionOptions, bucketOptions)`

ReGrid drivers MUST provide a constructor to return a new `Bucket` instance, which exposes all the public API methods.

`connectionOptions` MAY be an existing connection, if that is more suitable to your chosen language.

###### Code Example

```javascript

var connectionOptions = {
  // required connection options. Adapt to your chosen language.
}

var bucketOptions = {
  bucketName: 'fs',
  chunkSizeBytes: 1024 * 255, // 255KB SHOULD be the default chunk size.
  concurrency: 10 // OPTIONAL - useful if you are writing files asynchronously
}

var bucket = new ReGrid(connectionOptions, bucketOptions)

bucket // a new bucket instance
```

---

### `bucket.initBucket()`

ReGrid drivers MUST provied a method to create required tables and indexes.

##### Table Names

Two tables MUST be created for ReGrid to function, the 'files' table and the 'chunks' table. Tables MUST be a combination of the `bucketName` followed by an underscore and the table type. Given the default `bucketName` of 'fs' the files table MUST be named `fs_files` and the chunks table MUST be named `fs_chunks`

The driver MUST check whether the tables already exist before creating them. If creating the tables fails the driver MUST return an error.

##### Indexes

For efficient retrieval of files and chunks, a few indexes are required by ReGrid. Indexes MUST be named as shown below.

```javascript
r.table('<FilesTable>').indexCreate('file_ix', [r.row('status'), r.row('filename'), r.row('finishedAt')])

r.table('<ChunksTable>').indexCreate('chunk_ix', [r.row('file_id'), r.row('num')])
```

The driver MUST check whether the indexes already exist before creating them. If creating the indexes fails the driver MUST return an error.

###### Code Example

```javascript
var bucket = new ReGrid(connectionOptions, bucketOptions)

// Takes no arguments, and is asynchronous. Node.js ReGrid library returns a promise, adapt to your chosen language.
bucket.initBucket().then(function () {
  // Tables and indexes MUST now be ready for use.
  // use tableWait() and indexWait()
})
```

---

### `bucket.createWriteStream(filename, options)`

Drivers SHOULD use their languages built-in stream abstraction. Otherwise, they MUST provide their own.

###### Code Example

```javascript
// An options object MAY be passed in. All fields are optional.
var options = {
  chunkSizeBytes: 1024 * 255,
  metadata: null
}

bucket.createWriteStream(filename, options) // returns a stream
```

---

### `createReadStreamById(file_id)`

Get a `readStream` by `id`

###### Code Example

```javascript

bucket.createReadStreamById(file_id) // returns a stream
```

---

### `createReadStreamByFilename(filename, options)`

Get a `readStream` by `filename`. Since filenames are not unique, there can be multiple "revisions" of a file. A user may optionally specify a `revision` in the options object.

###### Code Example

```javascript

var options = {
  // Revision numbers are defined as follows:
  // 0 = the original stored file
  // 1 = the first revision
  // 2 = the second revision
  // etc…
  // -2 = the second most recent revision
  // -1 = the most recent revision
  // Defaults to -1 (the most recent revision).

  revision: -1
}

bucket.createReadStreamByFilename(filename, options) // returns a stream
```

### Finding Files

TODO

### Deleting Files

TODO

### Renaming Files

TODO

### Maintenance Operations

TODO

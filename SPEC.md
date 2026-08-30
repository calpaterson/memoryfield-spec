# The memoryfield format

## Abstract

This document specifies model-independent format for storing and sharing
memories for use by AI agents and humans.

The format prioritises being

- simple
- human readable
- portable
- using widely adopted, standard technology
- and "progressive" (ie: optional) efficiency enhancements

## Status

This is a draft specification, version 0.1 (2026-08), and feedback is most
welcome.

## Requirement key words

This document uses the capitalised key words "MUST", "MUST NOT", "REQUIRED",
"SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT
RECOMMENDED", "MAY", and "OPTIONAL", as described in RFC 2119.  Lowercase
uses of those words in prose are descriptive and impose no requirement, as
described in RFC 8174.

## Overview

A memoryfield is a collection of files (for example in a zipfile, in a git
repo, served over HTTP, in an Amazon S3 bucket, etc) composed mainly of
Markdown text files with YAML frontmatter and, optionally, one or more vector
indexes.  The canonical data is the Markdown files - the indexes are derived
data and can be regenerated at any time.

## Definitions

- memoryfield - a named collection of files conforming to this spec,
  distributed as a local directory, zip file, over git, or some other transport
- page - a UTF-8 encoded markdown file ending in `.md`
- `index.md` - an optional introduction file.  It is free format but does not
  contain a catalogue of the memoryfield's pages or files
- vector index - an optional, pre-computed file of embeddings derived from pages
- model code - a canonical, version-pinned identifier of an embedding model
  as published by that model's provider, such as `nomic-embed-text-v1.5`.
  Where a provider's shorthand omits the version (eg: ollama's
  `nomic-embed-text` tag), the model code still includes it, so that indexes
  produced by different versions are never conflated
- transport - mechanism for distribution of a memoryfield - could be zip, git,
  Amazon S3, sftp, SSH or a sync-based service such as Dropbox

## Format

### Container

- memoryfields MUST be flat directories
    - they MUST contain one or more "page" files ending in `.md`
    - they MUST NOT include sub-directories with pages inside
      - they MAY include sub-directories for other reasons
        - Implementations MUST NOT index such sub-directories
    - they MAY include non-markdown files (such as images, or videos that add extra context)
        - non-`.md` files are never pages and MUST NOT be indexed
- Implementations MUST ignore debris files created by sync tools, editors or
  operating systems: names containing `.sync-conflict-`, names ending in `~`,
  and `.DS_Store`, `desktop.ini` and `Thumbs.db`
- Producers SHOULD NOT include helper or log files in a memoryfield; any
  other `.md` file at the root is a page
  - logs MAY be placed in a subdirectory to avoid being indexed
- if served as a single archive, they MUST be a valid ZIP file
    - the archive SHOULD use the `.memoryfield.zip` extension
    - and MAY be served over HTTP with `Content-Type: application/zip`

### Pages

Pages contain prose describing a topic.

- Pages MUST be UTF-8 encoded Markdown with the `.md` file extension
- Page filenames MUST consist only of ASCII lowercase letters, digits, and
  hyphens, and MUST begin and end with a letter or digit, such as
  `carbon-fibre.md`
    - Pages SHOULD use `-` in preference to ` ` or `_` to separate words in the filename
- Pages SHOULD include a YAML (1.1) frontmatter block at the start of the file
- Pages SHOULD include comprehensive sources and citations
    - This is to allow for later confirmation of facts, reflowing, splitting and
      other editing of pages without information loss

#### Frontmatter

The following frontmatter fields are defined.

Implementations MUST NOT rely on the presence of these fields nor even the
presence of frontmatter at all.  Pages without frontmatter are valid pages.

Implementations MUST NOT raise errors on the presence of frontmatter fields
other than those described below.

| Field     | Requirement | Description                                        |
|-----------|-------------|----------------------------------------------------|
| `title`   | SHOULD      | Human-readable title for the file                  |
| `uuid`    | SHOULD      | An unchanging universally unique UUID for the file |
| `summary` | MAY         | One-sentence summary for display                   |
| `created` | SHOULD      | ISO 8601 datetime                                  |
| `updated` | SHOULD      | ISO 8601 datetime                                  |

Example:

```markdown
---
title: Carbon Fibre Woks
created: '2026-03-01T09:00:00Z'
updated: '2026-08-22T14:30:00Z'
uuid: 6aa615f0-486f-48a7-a210-ba4f5ff18c8b
summary: Notes on the surprising thermal properties of carbon fibre cookware.
---

Carbon fibre woks conduct heat unevenly but...
```

Datetimes MUST be quoted strings, otherwise YAML 1.1 parsers (eg: PyYAML) will
coerce them to datetime objects.

#### Page length

- Pages SHOULD NOT exceed 8192 bytes
- Pages MAY exceed this limit, but index implementations are not required to
  handle more than the first 8,192 bytes
- Producers SHOULD split pages that would exceed the limit; each resulting
  page SHOULD receive a fresh `uuid` and SHOULD preserve the original's
  sources and citations

### Index (non-vector) file (`index.md`)

- The memoryfield MAY contain an `index.md` file
- The `index.md` MAY include a broad introduction to the theme and content of the memoryfield
- The `index.md` file MUST NOT contain or comprise a catalogue of pages present in the memoryfield
    - Indexes are commonly read by agents and should not bulk insert the titles
      of all pages into the current context window
    - The memoryfield MAY include a `listing.md` to provide such a catalogue of
      pages and their titles/subjects/etc.
          - This is to allow for page enumeration over transports that do not
            otherwise support this (eg: some HTTP servers)

### Vector index files

- memoryfields SHOULD include at least one pre-computed vector index
  - the index is a convenience, not part of the canonical data; very small
    fields (eg: under ~100 pages) MAY omit it
  - `nomic-embed-text-v1.5` SHOULD be one of them
  - vector index filenames MUST begin with the full code of the embedding model
    - eg: `nomic-embed-text-v1.5.sqlite3`
- memoryfields MAY support other models
- When embedding a page for the vector index, the embedding input MUST be the
  complete UTF-8 contents of the .md file, including frontmatter
  - Implementations MAY prepend or append model-mandated task markers, such as
    `search_document: `
  - Implementations embedding pages MAY truncate the file for embedding
    purposes if it exceeds 8,192 bytes
- Vector index files MAY be in any format
  - sqlite3 is suggested
  - Vector indexes SHOULD NOT be provided in textual formats - such as csv - as
    floats do not round-trip cleanly through such formats

A valid sqlite schema for `nomic-embed-text-v1.5` (requires the sqlite-vec
extension; the embedding is the vector serialized as a float32 BLOB,
e.g. `sqlite_vec.serialize_float32()`):

```sql
CREATE TABLE pages (
    filename      TEXT PRIMARY KEY,
    frontmatter   JSON NOT NULL,          -- frontmatter encoded as JSON
    last_modified DATETIME NOT NULL,      -- MAY differ from `updated`
    sha256_hash   BLOB NOT NULL,          -- sha256 of file contents
    embedding     BLOB NOT NULL           -- vector serialized as float32 (768 weights for nomic-embed-text-v1.5)
);
```

Inserting a page:

```sql
INSERT INTO pages (filename, frontmatter, last_modified, sha256_hash, embedding)
VALUES (:filename, :frontmatter, :last_modified, :sha256_blob, :embedding_blob);
```

Searching (k-nearest neighbours; a full scan is fine for field-scale index
sizes):

```sql
SELECT filename, vec_distance_cosine(embedding, :query_blob) AS distance
FROM pages
ORDER BY distance
LIMIT 20;
```

The schema is an example only - other layouts are permitted (see "Vector index
files": indexes MAY be in any format).  Fields large enough to need an
approximate nearest-neighbour index may instead use a `vec0` virtual table,
which holds only the vectors and requires a separate mapping table from
filename to integer rowid.

## Transport specific notes

A memoryfield MAY be provided over any transport, but specifically supported transports are:

- Local files
- HTTP(S)
- Git
- Amazon S3-compatible object stores
- Syncthing and Dropbox

### HTTP

- A memoryfield MAY be served over HTTP instead of distributed as a solid zip file
- The HTTP server MUST serve `index.md` as `/` if it does not natively offer directory listing
- The HTTP server SHOULD support `GET /memoryfield.zip` returning the entire dataset
- The HTTP server MUST support `GET /{page_filename}.md` returning individual memory files
- The HTTP server SHOULD support `GET /search?q={search_terms}` returning
  ranked results as JSON
    - The response MUST be a JSON object containing the key `results`,
      which is an array ordered by descending relevance, or an empty array `[]`
      when there are no matches
    - The object MAY contain additional keys (eg: `count`, pagination, or
      server metadata); consumers MUST NOT rely on them
    - Each result MUST contain:
        - `filename` - the page filename, e.g. `carbon-fibre-woks.md`
        - `frontmatter` - the page's frontmatter as a JSON object, or `null`
          if the page has no frontmatter
    - Results MAY include a `score` so that consumers can threshold
    - Servers without a vector index MUST fall back to a case-insensitive
      substring match over the filename, title and summary
- If authentication is used, the HTTP server SHOULD support authentication via
  HTTP Basic Auth

#### Writable pages (`PUT`)

A memoryfield server MAY support `PUT /{page_filename}.md` to create or update
a single page.  If it does:

- The request body MUST be the complete UTF-8 encoded page, including any
  frontmatter.  There is no partial update or merge - the file is replaced in
  its entirety.  Whole-page granularity keeps conflicts rare and
  last-write-wins well-defined, and maps directly onto git
- `PUT` on a filename that does not exist creates the page (201 Created);
  `PUT` on an existing filename replaces it (204 No Content)
- Page identity is the filename.  When the stored page and the request body
  both carry a `uuid` frontmatter field and the values differ, the server
  SHOULD reject the request with 409 Conflict rather than overwrite; a client
  intending to replace the page keeps the existing `uuid`.  When the body
  omits `uuid`, the server SHOULD preserve the stored value on update, and
  MAY generate one on create
- After a successful `PUT` the server MUST regenerate the vector index entries
  for the affected page from the new contents.  This MAY be asynchronous, but
  the new entries MUST be in place before the page is next returned by any
  index-backed query.  Derived artefacts such as `listing.md` MUST also be
  brought up to date
- Servers MUST reject filenames that do not conform to the page filename
  rules (400 Bad Request), bodies that are not valid UTF-8 (415 Unsupported
  Media Type), and empty bodies (400 Bad Request).  Pages without frontmatter
  are valid and MUST NOT be rejected
- If the server uses authentication, writes MUST require it; read access MAY
  remain unauthenticated.  A server without any authentication configured
  SHOULD NOT expose `PUT` except on loopback
- Concurrency is last-write-wins per page.  Servers MAY support conditional
  writes (eg: `If-Match` on an ETag) to surface conflicts, but MUST NOT
  require them

Servers MAY also support `DELETE /{page_filename}.md`; if so, they MUST
remove the page and all its index entries.  A rename is `PUT` to the new
filename plus `DELETE` of the old.

### S3-compatible object stores

- Index timestamps MUST refer to the `Last-Modified` date provided by the
  `ListObjectsV2` operation.
- Implementations MAY keep the vector index outside the object store
  - In this case, implementations SHOULD describe the location of it within the
    `index.md`

### Git

- Index timestamps MUST refer to the last 'committer date' touching the file in
  question
  - Index timestamps MUST NOT refer to `ctime`, `atime` or `mtime`.

### Local files and sync-based transports

Index timestamps MUST refer to the file's modification time (mtime).
Sync-based transports such as Syncthing and Dropbox preserve mtime, so the
same rule applies.

## Versioning and compat

Memoryfields intentionally don't carry a format version.  The format is
designed to be additive: future revisions of this specification will only add
or relax requirements.

## Security considerations

Memoryfields may contain private information.  When a memoryfield is served
or distributed over a network:

- The server MUST resolve page paths against the memoryfield root and reject
  any request that escapes it (eg: `..`, absolute paths, symlinks)
- If the server supports writes, it MUST require authentication for `PUT` and
  `DELETE`, and SHOULD use TLS for all authenticated traffic
- A vector index reveals page existence and content hashes even where the
  `.md` files themselves are access-controlled; do not distribute it for
  fields whose contents must not leak
- A downloadable `.memoryfield.zip` is a complete snapshot of the field; do
  not offer it from servers that gate individual pages

## Appendix A: Minimal example memory field

```
my-memories.memoryfield.zip
├── index.md
├── carbon-fibre-woks.md
├── finnish-bureaucracy-tips.md
├── wec-2026-season-notes.md
└── nomic-embed-text-v1.5.sqlite3
```

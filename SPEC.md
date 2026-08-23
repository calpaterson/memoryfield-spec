# memoryfield format

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

This is a draft specification and feedback is most welcome.

## Overview

A memoryfield is a collection of files (for example in a zipfile, in a git
repo, served over HTTP, in an Amazon S3 bucket, etc) composed mainly of
Markdown text files with YAML frontmatter and, optionally, one or more vector
indexes.  The canonical data is the Markdown files - the indexes are derived
data and can be regenerated at any time.

## Format

### Container

- memoryfields MUST be flat directories
    - they MUST contain one or more "page" files ending in `.md`
    - they MUST NOT include sub directories with pages inside
      - they MAY include sub-directories for other reasons
        - Implementations MUST NOT index such sub-directories
    - they MAY include non-markdown files (such as images, or videos that add extra context)
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
- Pages SHOULD include a YAML frontmatter block at the start of the file
- Pages SHOULD include comprehensive sources and citations
    - This is to allow for later confirmation of facts, reflowing, splitting and
      other editing of pages without information loss

#### Frontmatter

The following frontmatter fields are defined.

Implementions MUST NOT rely on the presence of these fields nor even the
prescence of frontmatter at all.  Pages without frontmatter are valid pages.

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
created: 2026-03-01T09:00:00Z
updated: 2026-08-22T14:30:00Z
uuid: 6aa615f0-486f-48a7-a210-ba4f5ff18c8b
summary: Notes on the surprising thermal properties of carbon fibre cookware.
---

Carbon fibre woks conduct heat unevenly but...
```

#### Page length

- Pages SHOULD NOT exceed 8192 bytes
- Pages MAY exceed this limit, but index implementations are not required to
  handle more than the first 8,192 bytes

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
  - `nomic-embed-text-v1.5` SHOULD be one of them
  - vector index filenames MUST begin with the full code of the embedding model
    - eg: `nomic-embed-text-v1.5.sqlite3`
- memoryfields MAY support other models
- When embedding a page for the vector index, the embedding input MUST be the
  complete UTF-8 contents of the .md file, including frontmatter, without
  modification
  - Implementations embedding pages MAY truncate the file for embedding
    purposes if it exceeds 8,192 bytes
- Vector index files MAY be in any format
  - sqlite3 is suggested
  - Vector indexes SHOULD NOT be provided in textual formats - such as csv - as
    floats do not round-trip cleanly through such formats

A sample sqlite3 index schema for `nomic-embed-text-v1.5`:

```sql
CREATE TABLE pages (
    filename TEXT PRIMARY KEY,
    frontmatter JSON, -- frontmatter encoded as JSON
    last_modified DATETIME NOT NULL, -- last modified time of the file, MAY differ from `updated`
    sha256_hash BLOB NOT NULL, -- sha256 hash of file contents
    embedding FLOAT[768] NOT NULL -- nomic-embed-text-v1.5 contains 768 weights
)
```

## Transport specific notes

Memoryfiles MAY be provided over any transport, but specifically supported transports are:

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
- The HTTP server SHOULD suport `GET /search?p={search_terms}` returning ranked results as JSON
- The HTTP server MAY support `PUT /{page_filename}.md` for addition or update of a page
    - If so, the server MUST regenerate all index entries for the affect file
- If authentication is used, the HTTP server SHOULD support authentication via
  HTTP Basic Auth

### S3-compatible object stores

- Index timestamps MUST refer to the `Last-Modified` date provided by the
  `ListObjectsV2` operation.
- Implemenations MAY keep the vector index outside the object store
  - In this case, implementations SHOULD describe the location of it within the
    `index.md`

### Git

- Index timestamps MUST refer to the last 'commiter date' touching the file in
  question
  - Index timestamps MUST NOT refer to `ctime`, `atime` or `mtime`.

## Appendix A: Minimal example memory field

```
my-memories.memoryfield.zip
├── index.md
├── carbon-fibre-woks.md
├── finnish-bureaucracy-tips.md
├── wec-2026-season-notes.md
└── nomic-embed-text-v1.5.sqlite3
```

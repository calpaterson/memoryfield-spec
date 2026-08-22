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
  - they MUST contain one or more "pages" ended in `.md`
  - they MUST NOT include sub directories
  - they MAY include non-markdown files (such as images, or videos that add extra context)
- if served as a single archive, they MUST be a valid ZIP file
  - the archive SHOULD use the `.memoryfield.zip` extension
  - and MAY be served over HTTP with `Content-Type: application/zip`

### Pages

- Pages MUST be UTF-8 encoded Markdown with the `.md` file extension
- Pages MUST have a filename that is a URL-safe sluf, such as `carbon-fibre.md`
  - Pages SHOULD use `-` in preference to ` ` or `_` to separate words in the filename
- PAges SHOULD include a YAML frontmatter block at the start of the file

#### Frontmatter

The following frontmatter fields are defined.  Implementions MUST NOT rely on
the presence of these fields nor even the prescence of frontmatter at all.
Markdown files without frontmatter are valid pages.

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
- Pages MAY exceed this limit, but index implementations are not required to handle more than the first 8,192 bytes

### Index (non-vector) file (`index.md`)

- The memoryfield MAY contain an `index.md` file
- If present, `index.md` SHOULD contain a listing of files within the memoryfield
  - This is to aid file enumeration over transports (like HTTP) that don't offer that feature
- The `index.md` MAY include a broader introduction to the theme and content of the memoryfield

### Vector index files

- memoryfields SHOULD include at least one pre-computed vector index
  - `nomic-embed-text-v1.5` SHOULD be one of them
  - vector index filenames MUST begin with the full code of the embedding model
    - eg: `nomic-embed-text-v1.5.sqlite3`
- memoryfields MAY support other models
- When embedding a page for the vector index, the whole of the page MUST be used as embedding input
  - Implementations embedding pages MAY truncate the file for embedding purposes if it exceeds 8,192 bytes
- Vector index files MAY be in any format
  - sqlite3 is suggested

A sample sqlite3 index schema for `nomic-embed-text-v1.5`:

```sql
CREATE TABLE pages (
    filename TEXT PRIMARY KEY,
    frontmatter JSONB, -- frontmatter encoded as JSON
    last_modified DATETIME NOT NULL, -- last modified time of the file, MAY differ from `updated`
    sha256_hash BLOB NOT NULL, -- sha256 hash of file contents
    embedding FLOAT[768] NOT NULL -- nomic-embed-text-v1.5 contains 768 weights
)
```

## Transport specific notes

### HTTP

- A memoryfield MAY be served over HTTP instead of distributed as a solid zip file
- The HTTP server MUST serve `index.md` as `/` if it does not natively offer directory listing
- The HTTP server SHOULD support `GET /memoryfield.zip` returning the entire dataset
- The HTTP server MUST support `GET /{page_title}.md` returning individual memory files
- The HTTP server SHOULD suport `GET /search?p={search_terms}` returning ranked results as JSON
- The HTTP server MAY support `PUT /{page_title}.md` for addition or update of a page
  - If so, the server MUST regenerate all index entries for the affect file
- If authentication is used, the HTTP server SHOULD support authentication via HTTP Basic Auth
  - The HTTP server SHOULD NOT invent any alternative form of authentication eg
    custom headers, cookies, etc

### S3-compatible object stores

...

### Git

...

## Appendix A: Minimal example memory field

```
my-memories.memoryfield.zip
├── index.md
├── carbon-fibre-woks.md
├── finnish-bureaucracy-tips.md
├── wec-2026-season-notes.md
└── nomic-embed-text-v1.5.sqlite3
```

# Tree-sitter Queries

## Why?

Traversal lets us walk the entire AST, but sometimes we already know the exact structure we want.

A **Tree-sitter Query** lets us search the AST for specific structural patterns.

```text
AST
 ↓
Query Pattern
 ↓
Matching Nodes
 ↓
Captures
 ↓
Extract Information
```

## Key Concepts

**Query**

A pattern describing the AST structure we want to find.

**Match**

One occurrence of that pattern in the AST.

**Capture**

A node explicitly named with `@name` in the query so we can access it.

Example:

```text
string_fragment @import_path
```

`@import_path` gives that node a name we can reference.

## Example

For:

```ts
import { User } from './models/user';
```

Query:

```text
import_statement
    ↓
source
    ↓
string
    ↓
string_fragment @import_path
```

The captured node's text gives:

```text
./models/user
```

## Traversal vs Query

**Traversal:**

```text
Walk nodes → inspect each node → decide what to extract
```

**Query:**

```text
Describe structure → find matching nodes → capture what matters
```

Both can be used for AST analysis.

## Repo AI Relevance

Queries allow us to efficiently extract useful code information such as:

* Imports
* Exported functions
* Classes
* Other specific code structures

```text
Source Code
 ↓
AST
 ↓
Query
 ↓
Matching Nodes
 ↓
Structured Repository Data
```

## What I Learned

* Queries search the AST using structural patterns.
* `matches()` runs a query against a tree.
* A match represents an occurrence of the pattern.
* `@name` creates a capture.
* `capture.node.text` gives the source text of the captured node.
* Queries are useful when we know exactly what AST structure we're looking for.

## Status

✅ Query concept understood
✅ Import query understood
✅ Match/capture concept understood

**Next:** Extracting structured symbols from source files

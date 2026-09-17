# Scanner, Resolver & Slicer

## Overview

Repo AI needs to understand an entire repository rather than individual source files.

The pipeline is:

```text
Repository
    ↓
Scanner
    ↓
Repo Metadata
    ↓
Resolver
    ↓
Dependency Graph
    ↓
Slicer
    ↓
Feature Slice
```

Each stage has a different responsibility:

* **Parser** → Understand one source file.
* **Scanner** → Discover and parse all relevant source files.
* **Resolver** → Convert import strings into concrete file-to-file dependencies.
* **Slicer** → Traverse those dependencies to extract a relevant group of files.

The important design principle is **deterministic analysis first**. We do not need an LLM to understand basic filesystem structure or import relationships.

---

# 1. Filesystem Scanner

## Problem

The parser can process a string of source code:

```ts
parser.parseSource(sourceCode)
```

But a real repository contains:

```text
repo/
├── src/
├── models/
├── controllers/
├── services/
├── node_modules/
├── dist/
├── .git/
└── ...
```

We need to discover relevant source files and feed each one to `CodeParser`.

## Responsibility

The scanner:

1. Receives a repository path.
2. Recursively discovers source files.
3. Ignores irrelevant directories/files.
4. Reads each source file.
5. Passes the source code to `CodeParser`.
6. Stores the resulting `FileSymbols`.

Supported source types:

```text
.ts
.js
.tsx
.jsx
```

Ignored examples:

```text
node_modules/
dist/
build/
.git/
coverage/
*.d.ts
```

## Technology

We use:

```text
fast-glob
```

because it provides recursive filesystem matching and convenient ignore patterns.

---

## Scanner Data Model

```ts
export interface RepoMetadata {
  rootPath: string;
  totalFiles: number;
  files: Record<string, FileSymbols>;
}
```

Conceptually:

```json
{
  "rootPath": "C:/projects/my-app",
  "totalFiles": 15,
  "files": {
    "src/app.ts": {
      "imports": [],
      "exports": []
    },
    "src/services/auth.ts": {
      "imports": [],
      "exports": []
    }
  }
}
```

The paths inside `files` are **relative to the repository root**.

This is important because the repository may be located anywhere on a developer's machine.

---

## Scanner Flow

```text
Repository path
      ↓
path.resolve()
      ↓
fast-glob
      ↓
source file paths
      ↓
read file
      ↓
CodeParser.parseSource()
      ↓
FileSymbols
      ↓
RepoMetadata
```

The scanner does **not** determine dependencies.

For example, if it sees:

```ts
import { User } from '../models/user';
```

it simply stores the import information produced by the parser.

It does not yet know which physical file that import refers to.

---

# 2. Dependency Resolver

## Problem

Tree-sitter can tell us:

```ts
import { User } from '../models/user';
```

But:

```text
"../models/user"
```

is only a **module specifier**.

Repo AI needs a concrete relationship:

```text
src/controllers/auth.ts
        ↓
src/models/user.ts
```

The resolver performs this conversion.

---

## Responsibility

The resolver:

1. Reads all files discovered by the scanner.
2. Examines every import.
3. Determines whether the import is external or internal.
4. Resolves relative imports to actual repository files.
5. Creates dependency edges.
6. Records unresolved imports.

---

# 3. Internal vs External Dependencies

An import such as:

```ts
import { User } from '../models/user';
```

starts with:

```text
.
```

so it is treated as a local repository import.

An import such as:

```ts
import express from 'express';
```

does not start with `.` or `/`.

Therefore it is treated as an external package.

The resolver produces two different types of relationships.

### Internal

```text
auth.ts → user.ts
```

### External

```text
auth.ts → express
```

---

# 4. Import Resolution

Suppose the importing file is:

```text
src/controllers/auth.ts
```

and it contains:

```ts
import { User } from '../models/user';
```

The resolver starts from the importing file's directory:

```text
src/controllers/
```

Then applies:

```text
../models/user
```

Result:

```text
src/models/user
```

The resolver then checks possible extensions:

```text
src/models/user.ts
src/models/user.tsx
src/models/user.js
src/models/user.jsx
```

It also checks index files:

```text
src/models/user/index.ts
src/models/user/index.tsx
src/models/user/index.js
```

If a known file matches, the dependency becomes concrete.

```text
src/controllers/auth.ts
              │
              ▼
       src/models/user.ts
```

---

# 5. Dependency Graph

The resolver produces:

```ts
export interface DependencyGraph {
  nodes: string[];
  internalEdges: ResolvedEdge[];
  externalDependencies: ExternalDependency[];
  unresolvedImports: Array<{
    from: string;
    source: string;
  }>;
}
```

## Nodes

Every discovered source file is a node.

```text
src/app.ts
src/controllers/auth.ts
src/services/auth.ts
src/models/user.ts
```

## Internal Edges

```ts
export interface ResolvedEdge {
  from: string;
  to: string;
  importedSymbols: string[];
}
```

Example:

```json
{
  "from": "src/controllers/auth.ts",
  "to": "src/models/user.ts",
  "importedSymbols": ["User"]
}
```

This tells us both:

* which files are connected
* what was imported

---

## External Dependencies

Example:

```json
{
  "from": "src/app.ts",
  "pkgName": "express"
}
```

This means:

```text
src/app.ts → express
```

External packages are not repository nodes because their source code is not part of the scanned repository.

---

## Unresolved Imports

Sometimes resolution fails.

For example:

```ts
import something from '@/utils/helper';
```

If the resolver does not understand the project's `@` alias configuration, it cannot safely determine the target file.

Instead of guessing, it records:

```json
{
  "from": "src/app.ts",
  "source": "@/utils/helper"
}
```

This is important.

**Unknown should remain unknown rather than becoming a false dependency.**

---

# 6. Graph Slicer

Once we have the dependency graph, we can ask a more useful question:

> "What files are required by this particular entry point?"

This is the responsibility of the slicer.

Suppose:

```text
auth.controller.ts
        ↓
auth.service.ts
        ↓
User.ts
```

If we extract a slice starting from:

```text
auth.controller.ts
```

we want:

```text
auth.controller.ts
auth.service.ts
User.ts
```

This is called a **transitive dependency slice**.

---

# 7. BFS Traversal

The slicer uses **Breadth-First Search (BFS)**.

Starting from:

```text
auth.controller.ts
```

the algorithm:

1. Add the entry file to the queue.
2. Visit the current file.
3. Add its dependencies.
4. Continue until the queue is empty.
5. Use a `visited` set to prevent duplicate traversal and infinite loops.

Example:

```text
auth.controller.ts
       │
       ├── auth.service.ts
       │       │
       │       └── User.ts
       │
       └── validator.ts
```

Traversal:

```text
Queue:
[auth.controller.ts]

Visit:
auth.controller.ts

Queue:
[auth.service.ts, validator.ts]

Visit:
auth.service.ts

Queue:
[validator.ts, User.ts]

...
```

Eventually:

```text
Visited:
auth.controller.ts
auth.service.ts
validator.ts
User.ts
```

---

# 8. Why `visited` Matters

Real repositories can contain circular dependencies:

```text
A → B
↑   ↓
└── C
```

Without a `visited` set, traversal could continue forever.

The slicer therefore maintains:

```ts
const visited = new Set<string>();
```

Before adding a dependency:

```ts
if (!visited.has(neighbor)) {
  visited.add(neighbor);
  queue.push(neighbor);
}
```

This guarantees that a file is processed at most once during the slice operation.

---

# 9. External Dependencies in a Slice

The slicer also collects external packages used by files inside the slice.

For example:

```text
auth.controller.ts
   ↓
auth.service.ts
   ↓
bcrypt
jsonwebtoken
```

The resulting slice can contain:

```json
{
  "entryPoint": "src/controllers/auth.ts",
  "internalFiles": [
    "src/controllers/auth.ts",
    "src/services/auth.ts",
    "src/models/User.ts"
  ],
  "externalDependencies": [
    "bcrypt",
    "jsonwebtoken"
  ],
  "fileCount": 3
}
```

This gives Repo AI both:

* the repository files required
* the external technologies required

---

# 10. Complete Data Flow

The three components work together:

```text
                REPOSITORY
                    │
                    ▼
             ┌─────────────┐
             │   SCANNER   │
             └──────┬──────┘
                    │
                    ▼
             RepoMetadata
                    │
                    ▼
             ┌─────────────┐
             │  RESOLVER   │
             └──────┬──────┘
                    │
                    ▼
            DependencyGraph
                    │
                    ▼
             ┌─────────────┐
             │   SLICER    │
             └──────┬──────┘
                    │
                    ▼
              FeatureSlice
```

---

# 11. Why This Matters for Repo AI

At this point Repo AI can perform **deterministic repository understanding**.

It can answer:

### Scanner

> What source files exist?

### Parser

> What does each file import and export?

### Resolver

> Which actual files depend on which other files?

### Slicer

> What files are involved when starting from this entry point?

This creates the structural foundation for later AI functionality.

---

# 12. What We Have vs What We Don't Have

### We have

```text
✅ AST parsing
✅ Import extraction
✅ Export extraction
✅ Repository scanning
✅ File metadata
✅ Internal dependency resolution
✅ External dependency detection
✅ Unresolved import tracking
✅ Dependency graph
✅ Transitive dependency slicing
```

### We don't have yet

```text
❌ Understanding what a feature actually does
❌ Feature manifests
❌ Semantic embeddings
❌ Vector search
❌ Requirement → feature retrieval
❌ Blueprint generation
❌ Autonomous code modification
```

Those belong to the later **semantic/AI layer**.

---

# 13. Important Architectural Principle

The current system deliberately separates:

```text
DETERMINISTIC LAYER
────────────────────
Filesystem
AST
Imports
Exports
Paths
Dependencies
Graph
Slicing
```

from:

```text
AI LAYER
────────
Feature understanding
Semantic descriptions
Embeddings
Retrieval
Blueprint generation
Code generation
```

This separation is important because deterministic information should not depend on an LLM.

For example, we should not ask an LLM:

> "Which file does `../models/user` refer to?"

The resolver can determine that directly.

We should eventually ask the LLM something more appropriate:

> "Given this dependency slice, what capability does this code implement?"

---

# 14. Current Repo AI Pipeline

The architecture now looks like:

```text
SOURCE REPOSITORY
       │
       ▼
   FILESYSTEM
    SCANNER
       │
       ▼
   CODE PARSER
    Tree-sitter
       │
       ▼
   FILE SYMBOLS
       │
       ▼
    RESOLVER
       │
       ▼
 DEPENDENCY GRAPH
       │
       ▼
     SLICER
       │
       ▼
  FEATURE SLICE
       │
       ▼
   [NEXT STAGE]
 FEATURE MANIFEST
    EXTRACTION
```

## Current milestone

**Repository → Dependency Graph → Feature Slice**

This is the deterministic structural foundation of Repo AI.

Next we move from:

> **"What files are connected?"**

to:

> **"What capability does this connected group of files implement?"**

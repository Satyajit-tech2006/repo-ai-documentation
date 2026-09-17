# Embeddings & Feature Catalog

## Overview

The previous stage converts repository code into **Feature Manifests**.

Example:

```text
Feature Manifest
├── featureName
├── category
├── summary
├── capabilities
├── entryPoints
├── files
└── externalDependencies
```

The problem is now:

> How can Repo AI find features that are semantically relevant to a new requirement?

A simple keyword search is not enough.

For example, a user may search:

```text
"I need user login with secure token-based authentication."
```

while an existing manifest may say:

```text
"JWT Authentication"
```

There may be no exact keyword overlap, but the concepts are related.

This is where **embeddings** and **semantic search** are introduced.

---

# 1. Current Pipeline

The Repo AI pipeline is now:

```text
Repository
    ↓
Scanner
    ↓
Parser / Tree-sitter
    ↓
Resolver
    ↓
Dependency Graph
    ↓
Slicer
    ↓
Feature Slice
    ↓
Manifest Generator
    ↓
Feature Manifest
    ↓
Embedding
    ↓
Feature Catalog
    ↓
Semantic Search
```

The important transition is:

```text
Code Structure
      ↓
Semantic Representation
      ↓
Vector Representation
```

---

# 2. What Is an Embedding?

An embedding is a numerical representation of information.

Text such as:

```text
"JWT authentication with password verification"
```

is converted into a vector:

```text
[
  0.012,
  -0.284,
  0.731,
  ...
]
```

The vector contains many numerical dimensions.

The important property is that **semantically related text tends to have similar vector representations**.

For example:

```text
"JWT authentication"
```

and:

```text
"secure login using access tokens"
```

may produce vectors that are closer together than unrelated text such as:

```text
"image compression"
```

This allows Repo AI to search by **meaning rather than exact words**.

---

# 3. Why Repo AI Needs Embeddings

Suppose the feature catalog contains:

```text
Feature A:
JWT Authentication

Feature B:
Image Upload

Feature C:
Payment Processing

Feature D:
Real-time Chat
```

A new client requirement is:

```text
"Build a login system using secure access tokens."
```

Keyword matching may struggle because the requirement does not explicitly say:

```text
JWT Authentication
```

Semantic search instead compares the meaning of the requirement against the meaning of each stored feature.

Conceptually:

```text
New Requirement
      ↓
Embedding
      ↓
Query Vector
      ↓
Compare against stored vectors
      ↓
Similarity scores
      ↓
Top-K Features
```

---

# 4. `VectorEmbedder`

The `VectorEmbedder` class is responsible for communicating with the embedding model.

```ts
export class VectorEmbedder {
  private apiKey: string;
  private endpoint: string;

  ...
}
```

Its main operation is:

```ts
public async embed(text: string): Promise<number[]>
```

Input:

```text
text
```

Output:

```text
number[]
```

The returned array is the embedding vector.

---

# 5. Embedding API

The current implementation sends the text to the Gemini embedding endpoint.

Conceptually:

```text
Text
 ↓
Gemini Embedding Model
 ↓
Vector
```

The important abstraction is that the rest of Repo AI does not need to know the details of the HTTP request.

It can simply use:

```ts
const vector = await embedder.embed(text);
```

This creates a useful separation:

```text
Feature Catalog
      ↓
VectorEmbedder
      ↓
Embedding Provider
```

Later, the embedding provider can potentially be replaced without redesigning the entire catalog.

---

# 6. What Should Be Embedded?

We do not embed only the feature name.

For example:

```text
JWT Authentication
```

contains very little information.

Instead, the catalog constructs a richer semantic representation:

```text
Name: JWT Authentication
Category: auth
Summary: Handles authentication and JWT token generation.
Capabilities: credential verification, password hashing, JWT generation
```

This is called the **search text**.

The richer representation gives the embedding model more context.

---

# 7. Feature Catalog

The `FeatureCatalog` manages indexed features.

Its responsibilities are:

```text
1. Load existing catalog
2. Embed feature manifests
3. Store manifest + embedding
4. Search indexed features
5. Rank results
6. Return top-K matches
```

The current prototype stores everything in:

```text
feature-catalog.json
```

---

# 8. Indexed Feature

Each indexed entry has:

```ts
export interface IndexedFeature {
  manifest: FeatureManifest;
  embedding: number[];
}
```

Conceptually:

```text
IndexedFeature
├── Feature Manifest
└── Embedding Vector
```

Example:

```json
{
  "manifest": {
    "featureName": "JWT Authentication",
    "category": "auth",
    "summary": "...",
    "capabilities": [
      "credential verification",
      "JWT generation"
    ],
    "entryPoints": [
      "src/controllers/auth.ts"
    ],
    "files": [
      "src/controllers/auth.ts",
      "src/services/auth.ts"
    ],
    "externalDependencies": [
      "jsonwebtoken"
    ]
  },
  "embedding": [
    0.012,
    -0.284,
    0.731
  ]
}
```

The real embedding contains many more dimensions.

---

# 9. Indexing a Feature

The indexing process is:

```text
Feature Manifest
      ↓
Build search text
      ↓
Generate embedding
      ↓
Store manifest + vector
```

The important method is:

```ts
indexFeature(manifest)
```

The catalog constructs:

```ts
const searchText = `
  Name: ${manifest.featureName}
  Category: ${manifest.category}
  Summary: ${manifest.summary}
  Capabilities: ${manifest.capabilities.join(', ')}
`.trim();
```

Then:

```ts
const embedding = await this.embedder.embed(searchText);
```

The resulting pair is stored:

```ts
{
  manifest,
  embedding
}
```

---

# 10. Updating Existing Features

The current implementation checks whether a feature with the same `featureName` already exists.

If it does:

```text
Existing feature
      ↓
Replace manifest + embedding
```

Otherwise:

```text
New feature
      ↓
Append to catalog
```

This prevents multiple entries from being created for the same feature name during repeated indexing.

---

# 11. Query Embeddings

Searching works similarly.

Suppose the user enters:

```text
"I need code for secure login using JWT."
```

The query itself is converted into an embedding:

```text
User Requirement
      ↓
Embedding Model
      ↓
Query Vector
```

Then the query vector is compared with every indexed feature vector.

---

# 12. Cosine Similarity

The current prototype uses **cosine similarity**.

Conceptually:

```text
Vector A
   ↘
    Similarity
   ↗
Vector B
```

Cosine similarity measures the angle between two vectors.

A simplified interpretation:

```text
Closer direction → more semantically similar
Different direction → less semantically similar
```

The implementation calculates:

```ts
dotProduct / (normA * normB)
```

where:

```text
dotProduct = A · B
normA      = |A|
normB      = |B|
```

The exact mathematical definition is:

```text
cos(θ) = (A · B) / (||A|| ||B||)
```

---

# 13. Important: Similarity Is Not a Percentage

The system currently has code such as:

```ts
(results[0].score * 100).toFixed(2)
```

This should be understood only as a convenient display.

A cosine similarity score is **not automatically a probability or percentage match**.

For example:

```text
0.87
```

does not necessarily mean:

```text
87% match
```

It is better to think of it as:

```text
Cosine similarity = 0.87
```

and compare scores relative to other candidates.

---

# 14. Ranking Results

For every indexed feature:

```text
Query Vector
     ↓
Compare with Feature Vector
     ↓
Similarity Score
```

The catalog creates:

```ts
{
  manifest,
  score
}
```

for each feature.

Then results are sorted:

```ts
scored.sort((a, b) => b.score - a.score);
```

This puts the highest similarity scores first.

Finally:

```ts
scored.slice(0, topK);
```

returns only the requested number of results.

If:

```ts
topK = 3
```

the catalog returns the three highest-scoring candidates.

---

# 15. Example

Suppose the catalog contains:

```text
JWT Authentication
Image Upload
Payment Processing
Realtime Chat
```

Query:

```text
"I need secure user login with access tokens."
```

The system may produce something conceptually like:

```text
JWT Authentication     0.86
Realtime Chat           0.41
Payment Processing      0.32
Image Upload            0.18
```

The important result is:

```text
JWT Authentication
```

because its semantic representation is closest to the requirement.

The exact scores depend on the embedding model and should not be treated as universal thresholds.

---

# 16. Why This Is Better Than Keyword Search

Keyword search asks:

> Do these texts contain similar words?

Semantic search asks:

> Do these texts represent similar concepts?

Example:

```text
Query:
"secure login using access tokens"
```

Existing feature:

```text
"JWT-based user authentication"
```

The vocabulary differs, but the underlying capability is similar.

Embeddings allow Repo AI to detect this relationship.

---

# 17. Current Storage: JSON

The current implementation uses:

```text
feature-catalog.json
```

This is intentionally simple.

Advantages for the prototype:

* no database setup
* easy to inspect
* easy to debug
* easy to back up
* zero infrastructure
* suitable for small experiments

The JSON catalog is therefore a **prototype storage layer**, not the final architecture.

---

# 18. Future Storage: PostgreSQL + pgvector

As the number of projects and features grows, a JSON file becomes unsuitable.

The planned architecture is:

```text
PostgreSQL
      +
pgvector
```

Conceptually:

```text
Feature
├── metadata
├── manifest
└── embedding vector
```

Then vector search can be performed directly inside the database.

The eventual pipeline becomes:

```text
Feature Manifest
      ↓
Embedding
      ↓
PostgreSQL + pgvector
      ↓
Vector Search
      ↓
Top-K Features
```

The current JSON implementation allows us to validate the **retrieval concept before introducing database infrastructure**.

---

# 19. Why We Validate the Idea Before pgvector

There is no reason to build a production vector database before proving that:

```text
Requirement
     ↓
Embedding
     ↓
Relevant previous feature
```

actually produces useful results.

The correct development strategy is:

```text
Simple prototype
      ↓
Validate retrieval quality
      ↓
Improve representation/ranking
      ↓
Scale storage
```

This prevents infrastructure work from hiding a weak retrieval strategy.

---

# 20. What We Have Achieved

The system can now perform:

```text
Repository
    ↓
Feature Extraction
    ↓
Feature Manifest
    ↓
Embedding
    ↓
Catalog
    ↓
Semantic Query
    ↓
Ranked Features
```

This is a major milestone.

Repo AI can now begin answering:

> **"Which previous features are semantically similar to this new requirement?"**

---

# 21. What We Have Not Solved Yet

Semantic similarity alone is not enough to decide what should actually be reused.

For example, a retrieved feature might be semantically relevant but technically incompatible.

Suppose:

```text
Requirement:
React + PostgreSQL authentication
```

Retrieved feature:

```text
Express + MongoDB authentication
```

The feature is semantically relevant.

But blindly reusing it may be inappropriate.

Therefore, the next stage must consider:

```text
Semantic similarity
        +
Technical compatibility
        +
Architecture
        +
Dependencies
        +
Required adaptation
```

This leads to the next major Repo AI component:

```text
Retrieval
    ↓
Suitability / Compatibility Ranking
    ↓
Best Foundation
    ↓
Reusable Components
```

---

# 22. Current Architecture

```text
                 REPOSITORY
                      │
                      ▼
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
             MANIFEST GENERATOR
                      │
                      ▼
                  GEMINI
                      │
                      ▼
              FEATURE MANIFEST
                      │
                      ▼
                EMBEDDER
                      │
                      ▼
                 VECTOR
                      │
                      ▼
              FEATURE CATALOG
                      │
                      ▼
              SEMANTIC SEARCH
                      │
                      ▼
               TOP-K FEATURES
```

---

# 23. Key Engineering Principles

## Principle 1 — Structure Before Semantics

First determine:

```text
What files exist?
What do they import?
What depends on what?
What files belong to the feature?
```

Then use AI to determine:

```text
What does the feature actually do?
```

---

## Principle 2 — Facts Should Come From Deterministic Systems

The system already knows:

```text
files
entry points
dependencies
```

Do not ask the LLM to regenerate these facts.

---

## Principle 3 — Embed Rich Representations

Instead of embedding only:

```text
"Authentication"
```

embed meaningful context:

```text
Name
Category
Summary
Capabilities
```

This gives semantic retrieval more information.

---

## Principle 4 — Retrieval Is Not Reuse

A high semantic similarity score means:

> "This feature appears relevant."

It does **not** mean:

> "Copy this code."

Actual reuse requires additional analysis.

---

# 24. Current Milestone

The current milestone is:

> **Feature Manifests → Embeddings → Semantic Retrieval**

The system can now transform previous project knowledge into searchable semantic representations.

The next problem is:

> **Given several semantically relevant features, which one is actually suitable for the new project, and what can be safely reused?**

That is the beginning of the **retrieval + suitability ranking layer**.

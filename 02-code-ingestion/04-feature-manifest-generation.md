# Feature Manifest Generation

## Overview

At this stage, Repo AI moves from **deterministic code analysis** to **semantic understanding**.

The previous stages tell us the structure of the repository:

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
```

The problem is that structural information alone does not tell us what a group of files actually **does**.

For example, the system may know:

```text
src/controllers/auth.ts
src/services/auth.ts
src/models/User.ts

External:
bcrypt
jsonwebtoken
```

But we want Repo AI to understand:

```text
Feature:
JWT Authentication

Capabilities:
- credential verification
- password hashing
- JWT generation
```

The **Manifest Generator** uses an LLM to perform this semantic interpretation.

---

# 1. Why Feature Manifests?

A repository contains implementation details, but Repo AI eventually needs to reason about **capabilities**.

Consider two projects:

```text
Project A
├── auth.ts
├── User.ts
├── jwt.ts
└── bcrypt
```

and:

```text
Project B
├── loginController.ts
├── authService.ts
├── TokenService.ts
└── bcrypt
```

The file names and architecture may be different, but both may implement:

```text
User authentication
Password verification
JWT token generation
```

A feature manifest creates a normalized semantic description that can later be compared and retrieved.

Therefore:

```text
Code
 ↓
Structural analysis
 ↓
Feature slice
 ↓
Semantic interpretation
 ↓
Feature manifest
```

---

# 2. The Input: Feature Slice

The Manifest Generator does not analyze the entire repository.

It receives a `FeatureSlice`.

Example:

```ts
{
  entryPoint: "src/controllers/auth.ts",

  internalFiles: [
    "src/controllers/auth.ts",
    "src/services/auth.ts",
    "src/models/User.ts"
  ],

  externalDependencies: [
    "bcrypt",
    "jsonwebtoken"
  ],

  fileCount: 3
}
```

The slice was produced deterministically by the Slicer.

This is important because the LLM receives a **bounded and relevant portion of the repository** rather than thousands of unrelated files.

---

# 3. Manifest Generator Responsibility

The generator performs four major operations:

```text
FeatureSlice
     ↓
Load source files
     ↓
Build LLM context
     ↓
Gemini semantic analysis
     ↓
Validate response
     ↓
FeatureManifest
```

It should not:

* discover repository files
* resolve imports
* calculate dependency relationships
* invent file paths
* invent external dependencies

Those are responsibilities of previous deterministic stages.

---

# 4. Feature Manifest Schema

The manifest is represented using Zod.

```ts
export const FeatureManifestSchema = z.object({
  featureName: z.string(),
  category: z.string(),
  summary: z.string(),
  capabilities: z.array(z.string()),
  entryPoints: z.array(z.string()),
  files: z.array(z.string()),
  externalDependencies: z.array(z.string()),
});
```

The resulting type is:

```ts
export type FeatureManifest =
  z.infer<typeof FeatureManifestSchema>;
```

---

# 5. Meaning of Each Field

## `featureName`

A concise name describing the capability.

Example:

```text
JWT Authentication
```

Not:

```text
auth.ts code
```

The goal is semantic identification rather than a filename.

---

## `category`

A broad classification.

Examples:

```text
auth
payments
developer_tools
code_analysis
messaging
file_upload
search
```

This allows later retrieval to use both semantic similarity and categorical information.

---

## `summary`

A short explanation of what the feature does.

Example:

```text
Authenticates users using credentials and generates JWT
tokens for authenticated sessions.
```

The summary should describe **behavior**, not implementation details alone.

---

## `capabilities`

A list of concrete capabilities provided by the feature.

Example:

```json
[
  "credential verification",
  "password hashing",
  "JWT generation",
  "role-based authentication"
]
```

These are particularly useful for later semantic retrieval.

---

## `entryPoints`

The starting files of the slice.

Example:

```json
[
  "src/controllers/auth.ts"
]
```

This is deterministic information from the slicer.

The LLM does not need to generate it.

---

## `files`

All internal files belonging to the feature slice.

Example:

```json
[
  "src/controllers/auth.ts",
  "src/services/auth.ts",
  "src/models/User.ts"
]
```

Again, this comes directly from the slicer.

---

## `externalDependencies`

External packages used by the feature.

Example:

```json
[
  "bcrypt",
  "jsonwebtoken"
]
```

This also comes from deterministic analysis.

---

# 6. Deterministic vs Semantic Information

This is one of the most important architectural principles in Repo AI.

The final manifest combines two sources of information.

### Deterministic

Produced by our own code:

```text
entryPoints
files
externalDependencies
```

### Semantic

Produced by the LLM:

```text
featureName
category
summary
capabilities
```

Therefore:

```text
                 Feature Slice
                      │
          ┌───────────┴───────────┐
          ↓                       ↓
   Deterministic Data         Source Code
          │                       │
          │                       ↓
          │                    Gemini
          │                       │
          └───────────┬───────────┘
                      ↓
              Feature Manifest
```

---

# 7. Why the LLM Should Not Generate Everything

Suppose the LLM is asked:

> List all files involved in this feature.

It could hallucinate:

```text
src/utils/authHelper.ts
```

even though the file does not exist.

Our system already knows the exact file list.

Therefore we construct:

```ts
const fullManifest: FeatureManifest = {
  featureName: parsed.featureName,
  category: parsed.category,
  summary: parsed.summary,
  capabilities: parsed.capabilities,

  entryPoints: [slice.entryPoint],
  files: slice.internalFiles,
  externalDependencies: slice.externalDependencies,
};
```

This gives us a useful rule:

> **Use deterministic systems for facts and LLMs for interpretation.**

---

# 8. Building the LLM Context

The generator reads the source files belonging to the slice.

Conceptually:

```text
Feature Slice
     ↓
auth.ts
authService.ts
User.ts
     ↓
combine source
     ↓
LLM context
```

The context contains file boundaries:

```text
--- FILE: src/controllers/auth.ts ---

<source code>

--- FILE: src/services/auth.ts ---

<source code>

--- FILE: src/models/User.ts ---

<source code>
```

The file markers are important because they prevent the model from seeing the code as one giant undifferentiated block.

---

# 9. Repository Root Consideration

The paths stored by the scanner and slicer are repository-relative.

For example:

```text
src/controllers/auth.ts
```

This does not necessarily exist relative to the current process directory.

The actual path may be:

```text
C:/Users/Satyajit/projects/client-app/src/controllers/auth.ts
```

Therefore, production code should read files using:

```text
repository root + relative file path
```

Conceptually:

```ts
const fullPath = path.join(
  repositoryRoot,
  relativeFilePath
);
```

rather than:

```ts
fs.readFileSync(relativeFilePath);
```

This matters when Repo AI analyzes repositories located somewhere other than its own working directory.

---

# 10. Prompt Design

The LLM should be given a clearly defined task.

The prompt should explain:

```text
You are analyzing a feature slice from a software repository.

Identify:
- what the feature does
- its broad category
- its major capabilities

Return structured JSON.
```

The prompt should also provide:

```text
Entry Point
External Dependencies
Source Files
```

The model should focus on **observable functionality in the provided code**.

It should not speculate about functionality that is not supported by the code.

---

# 11. Structured JSON Output

The generator requests JSON output from Gemini.

Conceptually:

```ts
generationConfig: {
  responseMimeType: 'application/json'
}
```

The expected response is:

```json
{
  "featureName": "JWT Authentication",
  "category": "auth",
  "summary": "Handles user authentication and JWT token generation.",
  "capabilities": [
    "credential verification",
    "password hashing",
    "JWT generation"
  ]
}
```

This is preferable to free-form text because the output will be consumed programmatically.

---

# 12. Parsing the LLM Response

The API response contains generated text.

The generator extracts it:

```ts
const candidateText =
  data.candidates?.[0]?.content?.parts?.[0]?.text;
```

If no content is returned, the generator throws an error rather than silently producing an invalid manifest.

Then:

```ts
const parsed = JSON.parse(candidateText);
```

converts the JSON string into a JavaScript object.

---

# 13. Runtime Validation with Zod

TypeScript types disappear at runtime.

For example:

```ts
interface FeatureManifest {
  featureName: string;
}
```

does not protect us from an API response such as:

```json
{
  "featureName": 123
}
```

Therefore we use Zod:

```ts
FeatureManifestSchema.parse(fullManifest);
```

If the object violates the schema, Zod throws an error.

The flow becomes:

```text
Gemini
  ↓
JSON
  ↓
JSON.parse()
  ↓
Zod validation
  ↓
FeatureManifest
```

This gives the external AI response a runtime safety boundary.

---

# 14. Why Validation Matters

An LLM is probabilistic.

Even with structured output, responses can sometimes be:

* incomplete
* malformed
* missing fields
* incorrectly typed
* unexpected

Zod provides a deterministic contract:

```text
LLM output
     ↓
Does it satisfy our schema?
     │
   ┌─┴─┐
   │   │
  YES  NO
   │   │
   ↓   ↓
Manifest Error
```

This is a general engineering principle:

> **Never allow an external AI response to enter the application's trusted data layer without validation.**

---

# 15. Verification Harness

The generator reads:

```text
feature-slice.json
```

and sends the slice to Gemini.

The output is written to:

```text
feature-manifest.json
```

The current test flow is:

```text
feature-slice.json
       ↓
ManifestGenerator
       ↓
Gemini API
       ↓
FeatureManifest
       ↓
feature-manifest.json
```

This allows the entire pipeline to be tested stage by stage.

---

# 16. Current Repo AI Pipeline

We now have:

```text
                  REPOSITORY
                      │
                      ▼
                  SCANNER
                      │
                      ▼
                 FILES
                      │
                      ▼
             TREE-SITTER PARSER
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
```

The first part is deterministic.

The final semantic interpretation uses AI.

---

# 17. Example End-to-End Result

Suppose a repository contains:

```text
src/controllers/auth.ts
src/services/auth.ts
src/models/User.ts
```

and the dependency graph identifies:

```text
auth.ts
   ↓
authService.ts
   ↓
User.ts
```

The slicer produces:

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

Gemini interprets the source and returns:

```json
{
  "featureName": "JWT Authentication",
  "category": "auth",
  "summary": "Provides user authentication using credential verification and JWT-based sessions.",
  "capabilities": [
    "credential verification",
    "password hashing",
    "JWT generation"
  ]
}
```

Our system combines them:

```json
{
  "featureName": "JWT Authentication",
  "category": "auth",
  "summary": "Provides user authentication using credential verification and JWT-based sessions.",
  "capabilities": [
    "credential verification",
    "password hashing",
    "JWT generation"
  ],
  "entryPoints": [
    "src/controllers/auth.ts"
  ],
  "files": [
    "src/controllers/auth.ts",
    "src/services/auth.ts",
    "src/models/User.ts"
  ],
  "externalDependencies": [
    "bcrypt",
    "jsonwebtoken"
  ]
}
```

This is the first representation of a repository feature that is understandable to both humans and machines.

---

# 18. Why Feature Manifests Matter for the Final Product

The eventual Repo AI workflow is:

```text
Old Projects
     ↓
Scanner
     ↓
Dependency Graph
     ↓
Feature Slices
     ↓
Feature Manifests
     ↓
Embeddings / Index
     ↓
Semantic Retrieval
     ↓
New Client Requirement
     ↓
Find Relevant Previous Work
     ↓
Best Foundation + Reusable Components
     ↓
Blueprint
```

Without manifests, the system mainly understands **code structure**.

With manifests, it starts understanding **capabilities**.

That distinction is critical.

---

# 19. What We Have Completed

```text
Repository ingestion          ✅
Tree-sitter parsing           ✅
Import extraction             ✅
Export extraction             ✅
Filesystem scanning           ✅
Dependency resolution         ✅
Dependency graph              ✅
Graph slicing                 ✅
Feature slice generation      ✅
LLM semantic analysis         ✅
Feature manifest schema       ✅
Runtime validation             ✅
```

---

# 20. What Comes Next

The next major stage is **semantic indexing and retrieval**.

We currently have:

```text
Feature A → Manifest A
Feature B → Manifest B
Feature C → Manifest C
...
```

We need to make these searchable by meaning.

For example, a new requirement:

```text
"Build a student authentication system with
JWT login and role-based access."
```

should be able to retrieve:

```text
Previous Project:
JWT Authentication

Capabilities:
- credential verification
- JWT generation
- role-based authentication
```

This leads to:

```text
Feature Manifests
       ↓
Embeddings
       ↓
Vector Database
       ↓
Semantic Search
```

The planned storage layer is PostgreSQL + `pgvector`.

---

## Current Milestone

**Repository → Dependency Graph → Feature Slice → Feature Manifest**

The system has now crossed from simply understanding **where code is** to beginning to understand **what the code does**.

The next architectural problem is:

> **How do we efficiently retrieve the most relevant previous features when a new client requirement arrives?**

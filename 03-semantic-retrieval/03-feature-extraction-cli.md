Feature Extraction CLI

1. Purpose

The previous stages allow Repo AI to discover, describe, embed, and search reusable features.

This stage makes that capability usable from the terminal.

The CLI provides two commands:

search
extract

The flow is:

User Query
    ↓
CLI
    ↓
Feature Catalog
    ↓
Semantic Search
    ↓
Feature Manifest
    ↓
Feature Extractor
    ↓
Standalone Feature

The two files responsible for this are:

src/cli/index.ts
src/cli/extractor.ts

2. src/cli/index.ts

index.ts is the CLI entry point.

Its job is not to perform feature analysis itself.

Instead, it:

Read command
    ↓
Parse arguments
    ↓
Call the appropriate engine
    ↓
Display results

It acts as the interface between the developer and the Repo AI pipeline.

3. Imports

The CLI imports:

import { FeatureCatalog } from '../semantic';
import { FeatureExtractor } from './extractor';

FeatureCatalog

Provides semantic feature search.

It is responsible for:

Query
 ↓
Embedding
 ↓
Similarity search
 ↓
Ranked feature results

FeatureExtractor

Handles the physical extraction of a selected feature from the source repository.

The CLI therefore connects:

FeatureCatalog
      ↓
FeatureExtractor

4. Main Function

The CLI starts inside:

async function main() {

Because catalog search is asynchronous, the function is asynchronous.

At the end:

main().catch(console.error);

Any unhandled error is printed instead of silently failing.

5. Reading CLI Arguments

The command-line arguments come from:

process.argv

The implementation uses:

const args = process.argv.slice(2);

The first two values in process.argv are normally the Node executable and script path.

Therefore:

process.argv
    ↓
remove first two entries
    ↓
actual user arguments

Example:

npx tsx src/cli/index.ts search "JWT authentication"

becomes conceptually:

args[0] = "search"
args[1] = "JWT authentication"

6. Detecting the Command

The CLI reads:

const command = args[0];

It then chooses between:

search
extract
other

Conceptually:

command
   │
   ├── search  → semantic search
   │
   ├── extract → search + physical extraction
   │
   └── other   → show help

7. Creating the Catalog

The CLI initializes:

const catalog = new FeatureCatalog();

This gives both supported commands access to the indexed feature catalog.

The catalog loads the existing feature index and provides the search() operation.

8. The search Command

Example:

npx tsx src/cli/index.ts search "code parsing and graph dependency slicing"

The CLI first collects the query:

const query = args.slice(1).join(' ');

This allows multi-word queries to be passed naturally from the terminal.

For example:

args:
[
  "search",
  "code",
  "parsing",
  "and",
  "graph",
  "dependency",
  "slicing"
]

becomes:

"code parsing and graph dependency slicing"

9. Search Validation

If no query was supplied:

if (!query) {

the CLI prints:

Usage:
npx tsx src/cli/index.ts search "<prompt>"

and returns.

This prevents an empty semantic search request.

10. Performing Semantic Search

The CLI calls:

const results = await catalog.search(query, 3);

The 3 means:

Return the top 3 matching features.

The actual search is performed by FeatureCatalog.

Conceptually:

User Query
    ↓
Query Embedding
    ↓
Compare with catalog vectors
    ↓
Cosine similarity
    ↓
Sort descending
    ↓
Top 3

The CLI does not implement the vector search itself.

It simply consumes the catalog's results.

11. Displaying Search Results

The CLI loops through the results:

results.forEach((res, i) => {

For each result it displays:

Feature name
Similarity score
Category
Summary
Files

Example:

[1] Static Analysis Suite (Score: 74.8%)
    Category: code_analysis
    Summary: ...
    Files: parser.ts, resolver.ts, slicer.ts

This gives the developer enough information to decide what feature was found.

12. Similarity Score Display

The code uses:

(res.score * 100).toFixed(1)

This converts the numerical similarity into a convenient human-readable display.

For example:

0.748

is displayed as:

74.8%

Important:

This display should not be interpreted as a calibrated probability or literal percentage of reusable code.

It is a similarity score represented as a percentage for readability.

13. The extract Command

Example:

npx tsx src/cli/index.ts extract "dependency slicing" --out ./extracted/slicer

Unlike search, extract performs two operations:

Search
  ↓
Extract selected feature

The CLI first searches for the best matching feature.

14. Reading the Extract Query

The extraction query is taken from:

const query = args[1];

For:

extract "dependency slicing"

the query becomes:

dependency slicing

If the query is missing, the CLI displays usage information and stops.

15. Output Directory

The CLI searches for:

--out

using:

const outIdx = args.indexOf('--out');

If a valid output path follows it, that path is used.

Otherwise:

./extracted-feature

is used as the default.

Example:

--out ./extracted/slicer

results in:

./extracted/slicer

being passed to the extractor.

16. Finding the Best Feature

The CLI performs:

const results = await catalog.search(query, 1);

Unlike the normal search command, extraction only needs the best match.

Therefore:

topK = 1

The first result is treated as the selected feature.

17. Threshold Guard

Before extraction, the CLI checks:

if (results.length === 0 || results[0].score < 0.4)

There are two failure conditions:

No results
      OR
Similarity < 0.4

If either happens, extraction is refused.

This prevents Repo AI from blindly extracting an unrelated feature.

Conceptually:

Query
 ↓
Best candidate
 ↓
Score >= threshold?
 ├── No → stop
 └── Yes → extract

The 0.4 value is a prototype threshold, not a universal semantic boundary.

It should eventually be validated against real data.

18. Selecting the Manifest

Once the candidate passes the threshold:

const matched = results[0].manifest;

The CLI now has the complete FeatureManifest.

The manifest contains deterministic and semantic information such as:

featureName
category
summary
capabilities
entryPoints
files
externalDependencies

This manifest becomes the extraction contract.

19. Calling FeatureExtractor

The CLI finally calls:

FeatureExtractor.extract(
  matched,
  '.',
  outDir
);

The arguments mean:

matched → selected FeatureManifest
'.'     → source repository root
outDir  → destination directory

The CLI therefore hands the physical work to FeatureExtractor.

20. Complete index.ts Flow

CLI command
    ↓
process.argv
    ↓
Detect search / extract
    ↓
FeatureCatalog
    ↓
Semantic Search
    ↓
Candidate Feature
    │
    ├── search → display result
    │
    └── extract
          ↓
       threshold check
          ↓
       Feature Manifest
          ↓
       FeatureExtractor

21. src/cli/extractor.ts

extractor.ts is responsible for the physical construction of the extracted feature.

Its job is:

Feature Manifest
      ↓
Locate source files
      ↓
Copy files
      ↓
Collect dependencies
      ↓
Create package.json
      ↓
Create README
      ↓
Standalone feature directory

This is deliberately deterministic.

The extractor does not use an LLM to decide which files to copy.

The feature boundary has already been determined by the manifest.

22. Imports

The extractor uses:

import fs from 'fs';
import path from 'path';
import { FeatureManifest } from '../semantic';

fs

Used for:

checking files

creating directories

reading package.json

copying files

writing generated files

path

Used for safe filesystem path resolution.

FeatureManifest

Defines the extraction contract.

23. FeatureExtractor.extract()

The main method is:

public static extract(
  manifest: FeatureManifest,
  sourceRoot: string,
  outputDirectory: string
): void

It receives three things:

manifest
    ↓
What should be extracted?

sourceRoot
    ↓
Where does the original repository live?

outputDirectory
    ↓
Where should the extracted feature be created?

24. Resolving the Destination

The extractor creates:

const targetDir = path.resolve(outputDirectory);

This converts the output path into an absolute path.

If the directory does not exist:

fs.mkdirSync(targetDir, { recursive: true });

creates it.

recursive: true allows nested directories to be created automatically.

25. Step 1 — Copying Feature Files

The extractor loops through:

manifest.files

Each file is a path relative to the source repository.

For every file:

const srcPath = path.resolve(sourceRoot, relFile);
const destPath = path.resolve(targetDir, relFile);

The result is:

Source repository
      ↓
relative file path
      ↓
source absolute path

Target directory
      ↓
same relative file path
      ↓
destination absolute path

26. Preserving Directory Structure

Suppose the manifest contains:

src/analyzer/parser.ts
src/analyzer/resolver.ts
src/analyzer/slicer.ts

The extractor creates:

extracted/
└── src/
    └── analyzer/
        ├── parser.ts
        ├── resolver.ts
        └── slicer.ts

Before copying:

fs.mkdirSync(path.dirname(destPath), {
  recursive: true
});

creates the required directory hierarchy.

This is important because relative imports may depend on the original structure.

27. Missing Source Files

Before copying, the extractor checks:

if (!fs.existsSync(srcPath))

If a file cannot be found, it prints a warning:

[WARN] File not found at source: ...

and continues.

This prevents one missing file from immediately crashing the entire extraction.

However, a production implementation should eventually distinguish between optional and critical missing files.

28. Step 2 — Reading package.json

After copying files, the extractor looks for:

sourceRoot/package.json

It reads:

dependencies
devDependencies

and combines them into:

rootDeps

The purpose is to discover the exact dependency versions already used by the source repository.

29. Why Dependency Versions Matter

Suppose the feature requires:

tree-sitter
fast-glob

and the source project uses:

tree-sitter: 0.x.x
fast-glob: 3.x.x

The extractor can carry those versions into the generated package.

This is better than blindly installing arbitrary versions because the feature was originally built against the source project's dependency versions.

30. Step 3 — Filtering Node Built-ins

Some dependencies are provided by Node itself.

Examples:

fs
path
os
crypto
http
https
stream
util
events

They should not be added to package.json.

The extractor creates:

const nodeBuiltins = new Set([...]);

and skips them.

Conceptually:

Manifest dependencies
        ↓
Node built-in?
   ├── Yes → ignore
   └── No  → package dependency

31. Synthesizing package.json

The extractor creates a new package definition:

const standalonePkg = {
  name: `@repo-ai/${featureSlug}`,
  version: '1.0.0',
  description: manifest.summary,
  main: manifest.entryPoints[0] || 'index.ts',
  dependencies: standaloneDeps,
};

This package belongs to the extracted feature rather than the original application.

The generated package therefore contains only the dependencies required by the feature according to the manifest.

32. Feature Slug

The feature name is converted into a package-friendly identifier:

manifest.featureName
  .toLowerCase()
  .replace(/[^a-z0-9]/g, '-');

For example:

JWT Authentication

becomes approximately:

jwt-authentication

The package name becomes:

@repo-ai/jwt-authentication

33. Missing Dependency Version

The current implementation uses:

rootDeps[pkg] || 'latest'

This means:

Dependency found in source package.json
    ↓
use its version

Dependency not found
    ↓
use "latest"

latest is a convenient fallback for the prototype, but it is not ideal for reproducible builds.

A future version should probably fail, warn, or resolve the dependency more deliberately instead of silently using an unpinned version.

34. Step 4 — Generating the README

The extractor creates a README containing:

Feature name
Category
Summary
Capabilities
Files extracted

Conceptually:

# Feature Name

Category: ...

Summary: ...

## Capabilities

- capability 1
- capability 2

## Files Extracted

- file 1
- file 2

This gives the extracted feature basic human-readable documentation.

35. Final Extraction Result

After extraction, the destination can look like:

extracted/
└── slicer/
    ├── src/
    │   └── analyzer/
    │       ├── parser.ts
    │       ├── resolver.ts
    │       └── slicer.ts
    │
    ├── package.json
    └── README.md

The exact structure depends on the manifest.

36. Why Extraction Is Deterministic

The extractor does not ask an LLM:

"Which files should I copy?"

Instead:

Feature Manifest
      ↓
manifest.files
      ↓
Copy exactly those files

This is important because the earlier analysis stages already established the feature boundary.

The extractor should therefore be predictable and repeatable.

37. Search vs Extract

The two commands serve different purposes.

Search

search "<query>"

Answers:

Which existing features are relevant?

It only reads the catalog and displays results.

Extract

extract "<query>" --out <directory>

Answers:

Find the best matching feature and physically isolate it.

It performs:

Search
  ↓
Best candidate
  ↓
Threshold validation
  ↓
Physical extraction

38. End-to-End Example

Command:

npx tsx src/cli/index.ts search "code parsing and graph dependency slicing"

Flow:

Query
  ↓
FeatureCatalog.search()
  ↓
Embedding
  ↓
Cosine similarity
  ↓
Rank catalog
  ↓
Display top 3

Example result:

[1] Static Analysis Suite
    Score: 74.8%
    Category: code_analysis
    Files: parser.ts, resolver.ts, slicer.ts

Then:

npx tsx src/cli/index.ts extract "dependency slicing" --out ./extracted/slicer

Flow:

"dependency slicing"
        ↓
FeatureCatalog.search(query, 1)
        ↓
Best matching manifest
        ↓
Score >= 0.4?
        ↓
FeatureExtractor.extract()
        ↓
Copy feature files
        ↓
Create package.json
        ↓
Create README
        ↓
./extracted/slicer/

39. Architectural Significance

This CLI connects the previous stages into an actual developer workflow.

Before:

Repository Analysis

After:

Repository
    ↓
Feature Library
    ↓
Natural Language Search
    ↓
Reusable Feature
    ↓
Physical Extraction

Repo AI has therefore moved from simply understanding code to retrieving and transporting reusable code capabilities.

40. Current Milestone

The completed flow is:

✓ Repository scanning
✓ AST analysis
✓ Dependency graph
✓ Feature slicing
✓ Feature manifests
✓ Embeddings
✓ Feature catalog
✓ Multi-feature indexing
✓ Semantic search
✓ Feature extraction CLI

The next major problem is:

Retrieved Feature
        ↓
Is it actually compatible?
        ↓
How much adaptation is required?
        ↓
What should be reused?
        ↓
How should it be integrated?

That is the responsibility of the Compatibility & Suitability Engine.

41. Key Engineering Principle

The CLI follows a clean separation of responsibility:

FeatureCatalog
    → Find relevant features

FeatureManifest
    → Define the feature boundary

FeatureExtractor
    → Physically extract the boundary

CLI
    → Orchestrate these operations for the developer

The CLI itself should remain thin.

Business logic belongs in the underlying engines, not inside command parsing.
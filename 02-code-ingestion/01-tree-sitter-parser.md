# Tree-sitter Parser

## Why?

Repo AI needs to understand source code structurally instead of treating code as plain text.

```text
Source Code → Parser → Syntax Tree
```

## Key Concepts

**Parser**
Analyzes source code and produces a syntax tree.

**Grammar**
Defines the syntax rules of a programming language.

**Tree-sitter**
Parsing system that generates syntax trees and can handle incomplete/malformed source code.

**AST / Syntax Tree**
Structured representation of source code.

## Our Pipeline

```text
TypeScript Code
      ↓
Tree-sitter Parser
      +
TypeScript Grammar
      ↓
Syntax Tree
      ↓
Root Node
```

## Code

```ts
const parser = new Parser();

parser.setLanguage(TypeScript.typescript);

const tree = parser.parse(sourceCode);

const rootNode = tree.rootNode;
```

### What each part does

| Code            | Meaning                            |
| --------------- | ---------------------------------- |
| `new Parser()`  | Creates parser                     |
| `setLanguage()` | Gives parser the language grammar  |
| `parse()`       | Converts source code → syntax tree |
| `tree`          | Complete syntax tree               |
| `rootNode`      | Top-level node                     |
| `node.type`     | Type of syntax node                |
| `toString()`    | Prints tree structure              |

## Example Output

```text
Root node type: program

(program
  (import_statement ...)
  (export_statement
    (function_declaration ...)))
```

`program` is the root node.

The tree contains nodes such as:

```text
program
├── import_statement
└── export_statement
    └── function_declaration
```

## What I Learned

* Parser ≠ grammar.
* Grammar tells the parser how the language is structured.
* `parse()` produces the syntax tree.
* Nodes represent pieces of the source code.
* `node.type` tells us what kind of syntax construct a node represents.
* `toString()` is mainly useful for inspecting/debugging the tree.

## Repo AI Relevance

Tree-sitter gives Repo AI a structural view of source code.

Later:

```text
Repository
 ↓
Files
 ↓
AST
 ↓
AST Traversal
 ↓
Functions / Classes / Imports / Routes / etc.
```

## Status

✅ Parser working

**Next:** AST Node Traversal


## AST Traversal

Tree-sitter produces a tree where each node can have multiple children.

We can traverse it using preorder:

Current Node → Children from left to right

```text
program
├── import_statement
└── export_statement
    └── function_declaration
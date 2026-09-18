Compatibility & Suitability Engine

1. Purpose

The semantic retrieval layer answers:

Which existing features look relevant to the new requirement?

The Compatibility & Suitability Engine answers:

Which retrieved feature is actually suitable for this project, and
how can it be reused safely?

Semantic similarity alone is not enough for code reuse. Two features can
solve a similar problem while using different frameworks, dependencies,
architectures, schemas, or assumptions.

The suitability layer combines retrieved semantic candidates with
deterministic repository facts to estimate reuse feasibility.

2. Position in the Pipeline

Existing Repositories
        ↓
Code Ingestion
        ↓
Feature Slicing
        ↓
Feature Manifests
        ↓
Embeddings
        ↓
Feature Catalog
        ↓
Semantic Retrieval
        ↓
Compatibility & Suitability
        ↓
Reuse Strategy
        ↓
Blueprint Generation

The important boundary is:

Retrieval   → "This looks relevant."
Suitability → "This is relevant AND technically suitable."
Blueprint   → "Here is how we should reuse it."

3. Why Similarity Is Not Enough

A vector similarity score represents semantic closeness between the
query and a stored feature representation.

For example:

Requirement:
"Add JWT authentication to an Express backend."

Candidate A:
Express + JWT + MongoDB authentication

Candidate B:
Django + session-based authentication

Both may be semantically relevant to authentication, but Candidate A is
much easier to reuse directly because its framework and authentication
architecture are compatible.

Therefore:

Semantic Relevance ≠ Reusability

The system must evaluate multiple dimensions before deciding how a
feature should be reused.

4. Suitability Dimensions

4.1 Semantic Relevance

Does the existing feature solve the same or a closely related problem?

Sources:

embedding similarity

feature manifest

requirement analysis

4.2 Framework Compatibility

Does the feature use technologies compatible with the target project?

Consider:

language

runtime

backend framework

frontend framework

database

ORM/ODM

build system

package ecosystem

Example:

Target: Express + React
Candidate: Express + React
→ Compatible

Target: Express
Candidate: Django
→ Not suitable for direct reuse

4.3 Dependency Compatibility

Determine whether the candidate’s external dependencies can be used in
the target project.

Questions:

Does the target already use the dependency?

Can both versions coexist?

Does the dependency require a different runtime?

Does it introduce unnecessary complexity?

Is it tightly coupled to another package?

4.4 Architectural Compatibility

A feature can be semantically correct but structurally unsuitable.

Consider:

module boundaries

controller/service/repository structure

routing model

state management

database access pattern

dependency direction

communication mechanism

Example:

Target:
routes → controllers → services → repositories

Candidate:
routes → controllers → database directly

This candidate may still contain reusable logic, but it is more likely
ADAPTABLE than DIRECT_REUSE.

4.5 Coupling

A feature becomes harder to transplant when it depends heavily on
unrelated parts of its original repository.

Useful signals:

number of internal dependencies

number of external dependencies

dependency depth

shared utilities

shared configuration

shared database models

shared middleware

hidden assumptions

Low coupling
    ↓
Easier transplantation

High coupling
    ↓
More adaptation required

4.6 Data and Schema Compatibility

Some features depend heavily on the original project’s data model.

Example:

Candidate:
OrderService
    ↓
Order model
    ↓
Product model
    ↓
User model

If the target project has a fundamentally different schema, copying the
feature may not be appropriate.

Identify:

required models

required fields

relationships

API contracts

database assumptions

4.7 Environment and Configuration Requirements

Features may depend on:

environment variables

API keys

external services

storage providers

authentication secrets

URLs

deployment assumptions

These should be surfaced before reuse.

5. Reuse Strategies

The suitability engine should classify each candidate into a reuse
strategy.

DIRECT_REUSE

The feature can be transferred with little or no structural
modification.

ADAPTABLE

The core implementation is useful, but modifications are required.

Typical reasons:

different schemas

different module structure

different configuration

small dependency differences

naming/API differences

REIMPLEMENT

The candidate contains useful behavior or design knowledge, but copying
the original implementation is not practical.

Example:

Candidate: React authentication UI
Target: Next.js application with different architecture

→ Extract behavior/contracts
→ Reimplement using target architecture

INCOMPATIBLE

The feature should not be reused because the technical mismatch is
fundamental.

HUMAN_REVIEW

The system does not have enough evidence to make a safe decision.

Uncertainty should be an explicit output rather than hidden.

6. Candidate Scoring Model

A first version can use a weighted score:

Suitability Score =
    0.35 × Semantic Relevance
  + 0.20 × Framework Compatibility
  + 0.15 × Dependency Compatibility
  + 0.15 × Architectural Compatibility
  + 0.10 × Coupling Compatibility
  + 0.05 × Data/Schema Compatibility

All dimensions are normalized to:

0.0 → 1.0

These weights are provisional. They should eventually be validated
against real reuse outcomes.

7. Hard Constraints vs Soft Signals

Not every factor should simply be added to a weighted score.

Hard constraints

Potential rejection conditions:

unsupported runtime

fundamentally incompatible framework

impossible dependency requirement

unavailable required architecture

Soft signals

Useful for ranking:

semantic similarity

coupling

integration effort

schema similarity

dependency overlap

architectural similarity

The flow becomes:

Hard Constraints
       ↓
Filter impossible candidates
       ↓
Soft Scoring
       ↓
Rank viable candidates

This prevents semantically attractive but technically unusable features
from reaching the top.

8. Suitability Output

The engine should produce structured, explainable data.

{
  "featureName": "JWT Authentication",
  "semanticScore": 0.91,
  "suitabilityScore": 0.84,
  "strategy": "ADAPTABLE",
  "compatibility": {
    "framework": 1.0,
    "dependencies": 0.9,
    "architecture": 0.8,
    "coupling": 0.7,
    "schema": 0.6
  },
  "reasons": [
    "Uses the same backend framework",
    "JWT dependency already exists in target",
    "User schema differs",
    "Authentication middleware can be adapted"
  ],
  "requiredChanges": [
    "Adapt user model",
    "Update environment configuration"
  ]
}

The user should be able to understand why Repo AI selected a
feature.

9. Retrieval vs Suitability

These are deliberately separate stages.

Retrieval

Optimized for recall.

Find potentially relevant candidates.

It is acceptable for retrieval to return candidates that later turn out
to be unsuitable.

Suitability

Optimized for precision.

Determine which candidates are actually viable for reuse.

Therefore:

Retrieval:
"Show me things that might help."

Suitability:
"Now determine what actually fits."

10. LLM’s Role

The first version should remain deterministic-first.

Use deterministic repository facts wherever possible:

Framework
Language
Dependencies
Files
Imports
Exports
Graph structure
Configuration references
Schema references

Use an LLM where interpretation is difficult:

Does this feature perform the same business capability?
What architectural assumptions does it make?
Which parts are reusable?
What modifications are likely required?

The LLM should reason over structured repository intelligence instead of
receiving an entire repository unnecessarily.

11. Reuse Decision Flow

Requirement
    ↓
Semantic Retrieval
    ↓
Top-K Candidates
    ↓
Hard Compatibility Checks
    ↓
Remove Impossible Candidates
    ↓
Suitability Scoring
    ↓
Determine Reuse Strategy
    ↓
Explain Reasons
    ↓
Generate Blueprint

This creates a clean separation between:

Discovery → Evaluation → Planning

12. Important Principle: Maximum Useful Reuse

Repo AI should not optimize for:

“Reuse as much existing code as possible.”

It should optimize for:

“Reuse as much useful code as safely possible.”

A feature that requires more effort to adapt than implementing the same
capability cleanly from scratch should not be selected merely because it
exists in the library.

Therefore the suitability layer must eventually consider:

Reuse effort
vs
Scratch implementation effort

This is one of the most important product-level signals for validating
Repo AI.

13. Current Implementation Scope

Build now

1. Requirement representation
2. Candidate retrieval
3. Framework compatibility
4. Dependency compatibility
5. Basic architectural compatibility
6. Coupling signal
7. Weighted suitability score
8. Reuse strategy classification
9. Explainable output

Do not build yet

- Autonomous code modification
- Full semantic architecture reconstruction
- Complex machine-learned ranking
- Microservices
- Distributed processing
- Production cloud infrastructure

The goal is to validate the decision-making loop before adding
infrastructure.

14. Validation

For each real reuse attempt, record:

Candidate selected
Suitability score
Actual reuse strategy
Adaptation time
Scratch implementation time
Problems discovered
Human intervention
Final outcome

The most valuable metric is not the score.

Did Repo AI help the developer complete the feature faster and with
acceptable quality?

The scoring model should evolve based on these real outcomes.

15. Current Milestone

The system currently has:

✓ Repository scanning
✓ AST analysis
✓ Dependency graph
✓ Feature slicing
✓ Feature manifests
✓ Embeddings
✓ Feature catalog
✓ Multi-feature indexing
✓ Semantic search
✓ Feature extraction

The next milestone is:

→ Compatibility & Suitability Engine
→ Reuse Strategy
→ Blueprint Generation

The architectural transition is:

"What feature is similar?"
            ↓
"Which feature actually fits?"
            ↓
"How should we reuse it?"

That is the foundation of the Blueprint layer.
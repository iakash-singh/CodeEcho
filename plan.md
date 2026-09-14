# CodeEcho: Compression-Distance Code Duplicate Detector
## Implementation Plan & Milestone Roadmap

`codeecho` is a fast, language-agnostic command-line tool in Rust that detects semantically similar and duplicate code across codebases using **Normalized Compression Distance (NCD)**.

---

## 1. Core Mathematical Foundation

For two code segments $A$ and $B$, using compressor $C$ (Zstandard `zstd` level 19):

$$NCD(A, B) = \frac{C(A + B) - \min(C(A), C(B))}{\max(C(A), C(B))}$$

- $NCD \in [0, 1+\epsilon]$
- $NCD \approx 0$: Chunks share almost all information (identical / duplicates).
- $NCD \approx 1$: Chunks share no mutual redundancy (independent / unrelated).
- $NCD < \text{threshold}$ (default: $0.20$): Flagged as duplicates.

---

## 2. Project Architecture & Modular Design

```
codeecho/
├── Cargo.toml
├── src/
│   ├── main.rs               # CLI entrypoint (clap derive subcommands)
│   ├── lib.rs                # Library exports
│   ├── ncd.rs                # zstd compression and NCD calculation
│   ├── chunk.rs              # Chunk data structure (file, line range, AST metadata)
│   ├── normalize.rs          # Comment removal, whitespace collapse, identifier renaming
│   ├── prefilter.rs          # Fast MinHash signature & Jaccard similarity gating
│   ├── cluster.rs            # DisjointSet / Union-Find duplicate family clustering
│   ├── scanner.rs            # Recursive directory walker + Rayon parallel pipeline
│   ├── report.rs             # Terminal ANSI color & JSON formatted output
│   └── languages/
│       ├── mod.rs            # LanguageExtractor trait & registry
│       ├── javascript.rs     # JS/TS Tree-Sitter AST chunk extractor
│       └── python.rs         # Python Tree-Sitter AST chunk extractor
└── tests/
    ├── fixtures/             # Test datasets (identical, renamed, polyglot, large)
    └── integration_tests.rs
```

---

## 3. Step-by-Step Milestone Roadmap

### Milestone 1: Project Scaffold + Raw NCD on Two Whole Files
- **Goal**: Implement and verify raw NCD calculation on file pairs.
- **Components**:
  - Rust crate setup with `clap`, `zstd`, `anyhow`.
  - `compressed_size(data: &[u8]) -> usize` with `zstd::encode_all`.
  - `ncd(a: &[u8], b: &[u8]) -> f64`.
  - CLI subcommand: `codeecho compare <file_a> <file_b>`.
  - Unit tests: identical files ($NCD \approx 0$), unrelated files ($NCD \approx 1$), minor edits ($NCD \approx 0.1 - 0.3$).
- **Verification Demo**: `codeecho compare file_a.txt file_b.txt`

### Milestone 2: Function-Level Chunking (JavaScript / Tree-Sitter)
- **Goal**: Extract meaningful function-level chunks from source code via Tree-Sitter AST.
- **Components**:
  - Add `tree-sitter` and `tree-sitter-javascript`.
  - `Chunk` struct: `{ file_path, name, start_line, end_line, source_text, language }`.
  - Traversal of AST for `function_declaration`, `method_definition`, `arrow_function`, `function_expression`.
  - CLI subcommand: `codeecho extract <file>`.
  - Unit tests on multi-function JS files.
- **Verification Demo**: `codeecho extract sample.js`

### Milestone 3: Normalization Pass
- **Goal**: Remove noise (comments, indentation, variable names) that masks semantic duplicates.
- **Components**:
  - AST-based comment stripping (`comment` nodes).
  - Whitespace canonicalization.
  - `--normalize-identifiers`: local identifier mapping to `$v1`, `$v2`, etc.
  - CLI subcommand: `codeecho normalize <file> [--normalize-identifiers]`.
  - Unit tests verifying renamed/reformatted variants produce equivalent normalized output.
- **Verification Demo**: `codeecho normalize sample.js --normalize-identifiers`

### Milestone 4: Pairwise NCD Across All Chunks in a Directory
- **Goal**: Perform comprehensive pairwise comparisons across an entire directory.
- **Components**:
  - Recursive directory scanner discovering supported files.
  - Pairwise iteration $O(n^2)$ comparing every normalized chunk pair.
  - CLI subcommand: `codeecho scan <dir> [--threshold 0.2]`.
  - Ranked duplicate pair reporting.
- **Verification Demo**: `codeecho scan ./tests/fixtures/synthetic_repo`

### Milestone 5: Pre-Filter to Avoid Full $O(n^2)$ Compression
- **Goal**: Avoid expensive Zstd compression for obviously dissimilar chunk pairs.
- **Components**:
  - MinHash signature generator using $K=64$ hash permutations over $k$-shingles.
  - Fast Jaccard similarity estimation between chunk signatures.
  - Filter gate: only compute full NCD if MinHash estimated similarity $\ge \text{prefilter\_threshold}$.
  - `--stats` flag displaying candidate pair reduction and timing.
- **Verification Demo**: `codeecho scan ./tests/fixtures/large_repo --stats`

### Milestone 6: Clustering Into Duplicate Families
- **Goal**: Merge duplicate pairs into cohesive duplicate families / clusters.
- **Components**:
  - Disjoint-Set / Union-Find data structure with rank and path compression.
  - Transitive cluster aggregation ($A \sim B \land B \sim C \implies \{A, B, C\}$).
  - Cluster statistics (size, average NCD, representative chunk).
  - `--min-cluster-size` flag (default 2).
- **Verification Demo**: `codeecho scan ./tests/fixtures/clusters --min-cluster-size 2`

### Milestone 7: Report Output + Polish
- **Goal**: Deliver a polished, production-grade CLI user experience.
- **Components**:
  - `--format text` with ANSI colored severity badges and summary statistics.
  - `--format json` with structured schema for CI/CD integrations.
  - `--exclude` flag with built-in ignores (`.git`, `node_modules`, `target`, `vendor`).
  - Rayon multi-threading for parallel chunk processing and pair evaluation.
  - Comprehensive `README.md`.
- **Verification Demo**: `codeecho scan . --format json` & `codeecho scan . --format text`

### Milestone 8 (Stretch): Second Language Support (Python & Polyglot Engine)
- **Goal**: Add Python Tree-Sitter support to prove language-agnostic architecture.
- **Components**:
  - `LanguageExtractor` trait abstraction.
  - `tree-sitter-python` integration for `function_definition` and `async_function_definition`.
  - Polyglot scanner handling mixed `.js` and `.py` repositories.
- **Verification Demo**: `codeecho scan ./tests/fixtures/polyglot`

---

## 4. Verification & Testing Strategy

1. **Unit Tests**:
   - `cargo test --lib` (NCD math precision, AST parsing, MinHash Jaccard estimation, Union-Find).
2. **Integration Tests**:
   - Planted duplicate fixtures across all milestones.
   - Large codebase benchmark measuring pre-filter efficiency and execution time.
3. **Milestone Demos**:
   - Independent verification command execution after each milestone before advancing.

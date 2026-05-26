# Pict Template Preprocessor

> Compile-once, execute-many template optimization with dependency graphing and entity batch prefetch

The Pict Template Preprocessor eliminates redundant template parsing by compiling template strings into cached segment arrays on first encounter. Every subsequent render of the same template skips the character-by-character trie walk and executes a fast path that iterates pre-built segments directly. As a side effect of compilation, the preprocessor builds a directed dependency graph that captures template-to-template, template-to-data, and template-to-entity relationships for visualization and analysis.

For templates that render entities inside TemplateSet iterations, the preprocessor scans the template tree to discover entity expressions, resolves IDs across the dataset, and batch-fetches them before iteration begins -- converting N+1 individual fetches into a single batch request per entity type.

## Features

- **Compiled Template Cache** - Template strings are parsed once and stored as segment arrays; the trie state machine never runs twice for the same string
- **Sync and Async Fast Paths** - Compiled segments execute by iterating an array and calling Parse functions by direct reference
- **Expression Dependency Graph** - Directed graph with typed nodes (template, data, entity) and typed edges (renders, reads, reads-entity) with JSON and DOT export
- **Entity Batch Prefetch** - TemplateSet boundaries trigger a waterfall scan that discovers entity expressions, resolves IDs, checks the cache, and batch-fetches uncached entities via Meadow's filter endpoint
- **Transparent Integration** - Wraps Pict methods without modifying source; instantiate the service and the optimization is active
- **Extensible Classifiers** - Register custom edge classifiers for new template expression types

## Quick Start

```javascript
const libPict = require('pict');
const libPreprocessor = require('pict-template-preprocessor');

let _Pict = new libPict();

// Register and instantiate the preprocessor
_Pict.addServiceType('PictTemplatePreprocessor', libPreprocessor);
let _Preprocessor = _Pict.instantiateServiceProvider('PictTemplatePreprocessor');

// All parseTemplate calls now use the compiled fast path
_Pict.AppData.UserName = 'Alice';
let tmpResult = _Pict.parseTemplate('Welcome, {~D:AppData.UserName~}!');
// => "Welcome, Alice!"
```

## Documentation

- [Quickstart Guide](quickstart.md) - Step-by-step setup and first use
- [Architecture & Design](architecture.md) - Internals, trie caching, mermaid diagrams
- [Implementation Reference](implementation-reference.md) - Detailed behavioral documentation

## API Reference

Per-function documentation for all public methods:

- [compile](api/compile.md) - Compile a template string into a segment array
- [executeCompiled](api/execute-compiled.md) - Execute compiled segments synchronously
- [executeCompiledAsync](api/execute-compiled-async.md) - Execute compiled segments asynchronously
- [classifyEdges](api/classify-edges.md) - Populate the dependency graph from segments
- [addEdgeClassifier](api/add-edge-classifier.md) - Register custom graph edge classifiers
- [prefetchEntitiesForSet](api/prefetch-entities-for-set.md) - Batch-prefetch entities for a template set
- [clearCache](api/clear-cache.md) - Clear the compiled template cache
- [clear](api/clear.md) - Clear cache and graph
- [unwrapTemplateFunctions](api/unwrap-template-functions.md) - Remove wrappers and restore Pict methods
- [TemplateGraph](api/template-graph.md) - Graph data structure and query API

## Related Packages

- [pict](https://github.com/fable-retold/pict) - MVC application framework
- [pict-template](https://github.com/fable-retold/pict-template) - Template expression base class
- [pict-template-audit](https://github.com/fable-retold/pict-template-audit) - Template performance auditing
- [precedent](https://github.com/fable-retold/precedent) - Pattern trie engine used for template matching
- [fable](https://github.com/fable-retold/fable) - Application services framework

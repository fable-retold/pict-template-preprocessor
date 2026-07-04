# Pict Template Preprocessor

> **[Read the Pict-Template-Preprocessor Documentation](https://fable-retold.github.io/pict-template-preprocessor/)** - interactive docs with the full API reference.

A compile-once, execute-many template optimizer for the Pict framework. Compiles template strings into cached segment arrays so the character-by-character trie walk only happens once per unique template, builds an expression dependency graph for visualization and analysis, and batch-prefetches entities at TemplateSet boundaries to eliminate N+1 fetch patterns.

[MIT License](LICENSE)

---

## Features

- **Compiled Template Cache** - Compiles template strings into segment arrays on first parse; subsequent renders skip the trie walk entirely
- **Sync and Async Fast Paths** - Executes compiled segments directly, calling Parse functions by reference instead of re-scanning
- **Expression Dependency Graph** - Builds a directed graph of template-to-template and template-to-data relationships with JSON and Graphviz DOT export
- **Entity Batch Prefetch** - Scans templates at TemplateSet boundaries to discover entity expressions and batch-fetch them before iteration begins
- **Transparent Wrapper Pattern** - Installs method wrappers on Pict without modifying Pict's source; follows the same pattern as Pict-Template-Audit
- **Extensible Edge Classifiers** - Register custom classifiers for new template expression types to populate the dependency graph

## Installation

```bash
npm install pict-template-preprocessor
```

## Quick Start

```javascript
const libPict = require('pict');
const libPreprocessor = require('pict-template-preprocessor');

// Create a Pict instance
let _Pict = new libPict();

// Register the preprocessor service type and instantiate it
_Pict.addServiceType('PictTemplatePreprocessor', libPreprocessor);
let _Preprocessor = _Pict.instantiateServiceProvider('PictTemplatePreprocessor');

// Templates now use the compiled fast path automatically
_Pict.AppData.Name = 'World';
let tmpResult = _Pict.parseTemplate('Hello {~D:AppData.Name~}!');
// => "Hello World!"

// The second render of the same template skips compilation
let tmpResult2 = _Pict.parseTemplate('Hello {~D:AppData.Name~}!');
// => cache hit, fast-path execution only
```

## How It Works

When the preprocessor is instantiated, it wraps Pict's `parseTemplate`, `parseTemplateByHash`, `parseTemplateSet`, and `parseTemplateSetByHash` methods. On the first call with a given template string:

1. **Compile** - The trie state machine walks the string once, recording segments instead of executing parse functions
2. **Cache** - The segment array is stored in a `Map` keyed by the raw template string
3. **Execute** - The fast path iterates segments, concatenating literals and calling Parse functions directly

On subsequent calls with the same template string, steps 1-2 are skipped entirely.

### Compiled Segment Format

```javascript
// Template: "Hello {~Data:Name~}! See {~T:Footer~}"
// Compiles to:
[
    { Type: 'Literal', Value: 'Hello ' },
    { Type: 'Expression', Hash: 'Name', Leaf: <trie leaf>, Tag: '{~Data:' },
    { Type: 'Literal', Value: '! See ' },
    { Type: 'Expression', Hash: 'Footer', Leaf: <trie leaf>, Tag: '{~T:' },
]
```

### Expression Dependency Graph

As templates are compiled, the preprocessor classifies each expression and builds a directed graph:

```javascript
// After rendering templates that reference other templates and data
let tmpGraph = _Preprocessor.graph;

// Export as JSON for visualization tools
console.log(JSON.stringify(tmpGraph.toJSON(), null, 2));

// Export as Graphviz DOT format
console.log(tmpGraph.toDOT());

// Query specific relationships
let tmpEdges = tmpGraph.getEdgesFrom('template:MainPage');
```

### Entity Batch Prefetch

When a TemplateSet is rendered asynchronously, the preprocessor scans the template for `{~Entity:~}` expressions, resolves IDs across the dataset, and batch-fetches them using Meadow's filter endpoint before iteration begins:

```javascript
// Without preprocessor: N+1 fetches (one per record)
// With preprocessor: 1 batch fetch per entity type, then N cache hits

_Pict.TemplateProvider.addTemplate('CityRow', '<li>{~E:City^Record.IDCity^CityName~}</li>');

// Async template set automatically prefetches all City entities
_Pict.parseTemplateSetByHash('CityRow', records,
    (pError, pOutput) =>
    {
        // All City entities were batch-fetched before iteration began
        console.log(pOutput);
    });
```

## API Overview

### PictTemplatePreprocessor

| Method | Description |
|--------|-------------|
| `compile(pString, pParseTree)` | Compile a template string into a segment array |
| `executeCompiled(pSegments, pData, pContextArray, pScope, pState)` | Execute compiled segments synchronously |
| `executeCompiledAsync(pSegments, pData, fCallback, pContextArray, pScope, pState)` | Execute compiled segments asynchronously |
| `classifyEdges(pSegments, pSourceTemplateID)` | Populate the graph from compiled segments |
| `addEdgeClassifier(pTag, fClassifier)` | Register a custom graph edge classifier |
| `prefetchEntitiesForSet(pTemplateString, pDataSet, fCallback, pContextArray, pScope, pState)` | Batch-prefetch entities for a template set |
| `clearCache()` | Clear the compiled template cache |
| `clear()` | Clear cache and graph |
| `unwrapTemplateFunctions()` | Remove wrappers, restore original Pict methods |

### TemplateGraph

| Method | Description |
|--------|-------------|
| `addNode(pType, pID)` | Add a node to the graph |
| `addEdge(pFromKey, pToKey, pEdgeType)` | Add a directed edge |
| `getNodes()` | Get all nodes |
| `getEdges()` | Get all edges |
| `getEdgesFrom(pNodeKey)` | Get outgoing edges from a node |
| `getEdgesTo(pNodeKey)` | Get incoming edges to a node |
| `toJSON()` | Export graph as serializable JSON |
| `toDOT()` | Export graph as Graphviz DOT |
| `clear()` | Clear all nodes and edges |

## Testing

Run the test suite:

```bash
npm test
```

Run with coverage:

```bash
npm run coverage
```

## Related Packages

- [pict](https://github.com/fable-retold/pict) - MVC application framework
- [pict-template](https://github.com/fable-retold/pict-template) - Template expression base class
- [pict-template-audit](https://github.com/fable-retold/pict-template-audit) - Template performance auditing
- [precedent](https://github.com/fable-retold/precedent) - Pattern trie engine used for template matching
- [fable](https://github.com/fable-retold/fable) - Application services framework

## License

MIT

## Contributing

Pull requests are welcome. For details on our code of conduct, contribution process, and testing requirements, see the [Retold Contributing Guide](https://github.com/fable-retold/retold/blob/main/docs/contributing.md).

# Architecture & Design

This document describes the internal architecture of the Pict Template Preprocessor, including how the trie-based template engine works, how the preprocessor compiles templates into cached segments, how the dependency graph is built, and how entity batch prefetch operates at TemplateSet boundaries.

## The Problem

Pict's MetaTemplate engine parses every template string character-by-character through a trie-based state machine on every render. For templates rendered repeatedly -- list items, re-renders, reactive updates -- this is redundant work because the structure of the template never changes, only the data does. The preprocessor eliminates this redundancy.

## System Overview

```mermaid
flowchart TD
    A["pict.parseTemplate(string, data)"] --> B{Cached?}
    B -->|No| C["compile(string, parseTree)"]
    C --> D["Cache segment array"]
    D --> E["classifyEdges(segments, source)"]
    E --> F["executeCompiled(segments, data)"]
    B -->|Yes| F
    F --> G["Rendered output string"]

    style C fill:#f9f,stroke:#333
    style F fill:#9f9,stroke:#333
```

The preprocessor sits between Pict's public API and the MetaTemplate engine. On first encounter with a template string, it runs the full trie walk to produce segments. On subsequent encounters, it skips directly to execution.

## How the Trie Works

Pict's template engine uses [Precedent](https://fable-retold.github.io/precedent/) to build a trie (prefix tree) from registered pattern delimiters. Every template expression type registers a start and end pattern pair.

### Pattern Registration

When a template expression class calls `this.addPattern('{~Data:', '~}')`, Precedent inserts each character of the start pattern into the trie as a path of nodes. The final node stores the Parse function and the end pattern.

```mermaid
graph LR
    Root["{"] --> Tilde["~"]
    Tilde --> D1["D"]
    Tilde --> T1["T"]
    Tilde --> E1["E"]

    D1 --> Colon1[":"]
    Colon1 --> LeafD["Leaf: Data\nParse: render()\nEnd: ~}"]

    D1 --> a["a"]
    a --> t["t"]
    t --> a2["a"]
    a2 --> Colon2[":"]
    Colon2 --> LeafData["Leaf: Data (long)\nParse: render()\nEnd: ~}"]

    T1 --> Colon3[":"]
    Colon3 --> LeafT["Leaf: Template\nParse: render()\nEnd: ~}"]

    E1 --> Colon4[":"]
    Colon4 --> LeafE["Leaf: Entity\nParse: render()\nEnd: ~}"]

    style LeafD fill:#ffd,stroke:#333
    style LeafData fill:#ffd,stroke:#333
    style LeafT fill:#ffd,stroke:#333
    style LeafE fill:#ffd,stroke:#333
```

### Character-by-Character State Machine

The MetaTemplate string parser processes each character through a state machine with these transitions:

```mermaid
stateDiagram-v2
    [*] --> Scanning: Start
    Scanning --> PatternMatch: Char in trie root
    Scanning --> Scanning: Char not in trie root (literal)

    PatternMatch --> PatternMatch: Char continues trie path
    PatternMatch --> ContentCapture: Start pattern complete (leaf found)
    PatternMatch --> Scanning: Char breaks trie path (flush as literal)

    ContentCapture --> ContentCapture: Char not in end pattern
    ContentCapture --> EndMatch: Char begins end pattern

    EndMatch --> EndMatch: Char continues end pattern
    EndMatch --> ContentCapture: Char breaks end pattern
    EndMatch --> Dispatch: End pattern complete

    Dispatch --> Scanning: Call Parse(), reset state
```

**Without the preprocessor**, this state machine runs on every `parseTemplate()` call. For a template rendered 1000 times with 200 characters, that is 200,000 character transitions where the structure is identical each time.

**With the preprocessor**, the state machine runs once to produce a segment array. The 999 subsequent renders iterate the segment array directly.

## Compiled Segment Format

A compiled template is an array of segment objects. There are two types:

### Literal Segments

```javascript
{ Type: 'Literal', Value: 'Hello ' }
```

Pre-extracted string content between expressions. During execution, the value is concatenated directly to the output with no processing.

### Expression Segments

```javascript
{
    Type: 'Expression',
    Hash: 'AppData.Name',     // Content between start/end tags
    Leaf: <trie leaf node>,   // Direct reference to the trie leaf
    Tag: '{~Data:'            // PatternStartString for classification
}
```

The `Leaf` property holds a direct reference to the trie leaf node, which contains:
- `Parse` - The synchronous render function
- `ParseAsync` - The asynchronous render function
- `ParserContext` - The `this` context for calling Parse (the template expression instance)
- `PatternStartString` / `PatternEndString` - The delimiter strings
- `isAsync` - Whether this expression requires async execution

By storing a direct reference to the leaf, the fast path avoids re-traversing the trie entirely.

## Compilation Algorithm

The `compile()` method mirrors the MetaTemplate string parser's state machine but records segments instead of executing Parse functions:

```mermaid
flowchart TD
    Start["Start: empty segments[], literalBuffer"] --> Loop["For each character"]
    Loop --> InMatch{In pattern\nmatch?}

    InMatch -->|No| CheckRoot{Char in\ntrie root?}
    CheckRoot -->|Yes| BeginMatch["Start pattern match\nrecord char in outputBuffer"]
    CheckRoot -->|No| AppendLiteral["Append char to literalBuffer"]

    InMatch -->|Yes| ContinueStart{Start pattern\ncomplete?}
    ContinueStart -->|No, continues| AdvanceStart["Follow trie path\nrecord char"]
    ContinueStart -->|No, breaks| FlushFailed["Flush outputBuffer to literalBuffer\nReset state"]
    ContinueStart -->|Yes| CaptureContent["Capture content chars"]

    CaptureContent --> CheckEnd{End pattern\nbegins?}
    CheckEnd -->|No| CaptureContent
    CheckEnd -->|Yes| MatchEnd{End pattern\ncomplete?}
    MatchEnd -->|No| CaptureContent
    MatchEnd -->|Yes| RecordExpr["Flush literalBuffer as Literal segment\nPush Expression segment\nReset state"]

    AppendLiteral --> Loop
    BeginMatch --> Loop
    AdvanceStart --> Loop
    FlushFailed --> Loop
    RecordExpr --> Loop

    Loop --> Done["Flush remaining literalBuffer\nReturn segments[]"]

    style RecordExpr fill:#9f9,stroke:#333
    style Done fill:#9f9,stroke:#333
```

The key difference from the original parser: where the original calls `Parse(hash, data, ...)`, the compiler pushes `{ Type: 'Expression', Hash, Leaf, Tag }` to the segment array. The state transitions are identical to ensure perfect fidelity.

## Fast-Path Execution

### Synchronous Path

```mermaid
flowchart LR
    Segments["Segment Array"] --> Loop["For each segment"]
    Loop --> Check{Type?}
    Check -->|Literal| Concat["output += segment.Value"]
    Check -->|Expression| Call["output += leaf.Parse(hash, data, ...)"]
    Concat --> Loop
    Call --> Loop
    Loop --> Result["Return joined output"]
```

The synchronous fast path is a simple array iteration. No trie traversal, no state machine, no character-by-character scanning. Each Expression segment calls its Parse function by direct reference.

### Asynchronous Path

The async fast path uses Fable's Anticipate service to schedule each segment as a step in a waterfall:

```mermaid
flowchart TD
    Segments["Segment Array"] --> Anticipate["Create Anticipate instance"]
    Anticipate --> Schedule["For each segment, schedule a step"]

    Schedule --> StepCheck{Type?}
    StepCheck -->|Literal| StepLit["outputParts[i] = segment.Value\ncallback()"]
    StepCheck -->|Expression, sync| StepSync["outputParts[i] = leaf.Parse(...)\ncallback()"]
    StepCheck -->|Expression, async| StepAsync["leaf.ParseAsync(hash, data, (err, result) => {\n  outputParts[i] = result\n  callback()\n})"]

    StepLit --> Wait
    StepSync --> Wait
    StepAsync --> Wait

    Wait["anticipate.wait()"] --> Join["fCallback(null, outputParts.join(''))"]
```

Only N steps are created (one per segment), compared to the original async parser which creates one step per character. For a template with 200 characters and 5 expressions, this is 7 steps instead of 200.

## Expression Dependency Graph

As templates are compiled, the preprocessor classifies each Expression segment and populates a directed graph.

### Graph Structure

```mermaid
graph TD
    subgraph Nodes
        T1["template:MainPage"]
        T2["template:Header"]
        T3["template:UserList"]
        D1["data:Record.Title"]
        D2["data:AppData.Theme"]
        E1["entity:Author"]
    end

    T1 -->|renders| T2
    T1 -->|renders-set| T3
    T1 -->|reads| D1
    T2 -->|reads| D2
    T3 -->|reads-entity| E1

    style T1 fill:#adf,stroke:#333
    style T2 fill:#adf,stroke:#333
    style T3 fill:#adf,stroke:#333
    style D1 fill:#fda,stroke:#333
    style D2 fill:#fda,stroke:#333
    style E1 fill:#daf,stroke:#333
```

### Node Types

| Type | Description | Shape (DOT) |
|------|-------------|-------------|
| `template` | A named template hash | box |
| `data` | A data address path | ellipse |
| `entity` | An entity type name | diamond |

### Edge Types

| Edge Type | Source Tag | Meaning |
|-----------|-----------|---------|
| `renders` | `{~T:`, `{~Template:` | Source template renders target template |
| `renders-set` | `{~TS:`, `{~TemplateSet:` | Source renders target as a set iteration |
| `renders-if` | `{~TIf:`, `{~TemplateIf:` | Source conditionally renders target |
| `renders-if-else` | `{~TIfE:`, `{~TemplateIfElse:` | Source conditionally renders one of two targets |
| `reads` | `{~D:`, `{~Data:`, formatters | Source reads a data address |
| `reads-entity` | `{~E:`, `{~Entity:` | Source fetches an entity by type |

### Edge Classification

Edge classifiers are functions registered by PatternStartString. Each classifier receives the template hash (content between delimiters) and returns an array of edges to add:

```mermaid
flowchart LR
    Segment["Expression Segment\nTag: '{~T:'\nHash: 'Header:Record'"] --> Lookup["Look up classifier\nfor '{~T:'"]
    Lookup --> Classify["classifier('Header:Record')"]
    Classify --> Edges["[{ EdgeType: 'renders',\n   NodeType: 'template',\n   NodeID: 'Header' }]"]
    Edges --> Graph["graph.addEdge(\n  'template:Page',\n  'template:Header',\n  'renders')"]
```

Custom classifiers can be registered for application-specific template expression types via `addEdgeClassifier()`.

## Entity Batch Prefetch

### The N+1 Problem

When a TemplateSet renders N records and each record contains an `{~Entity:~}` expression, the standard rendering path makes N individual HTTP requests -- one per record per entity type.

```mermaid
sequenceDiagram
    participant TS as TemplateSet
    participant EP as EntityProvider
    participant API as REST API

    TS->>EP: getEntity('Author', 1)
    EP->>API: GET /1.0/Authors/1
    API-->>EP: { IDAuthor: 1, Name: 'Alice' }

    TS->>EP: getEntity('Author', 2)
    EP->>API: GET /1.0/Authors/2
    API-->>EP: { IDAuthor: 2, Name: 'Bob' }

    TS->>EP: getEntity('Author', 3)
    EP->>API: GET /1.0/Authors/3
    API-->>EP: { IDAuthor: 3, Name: 'Carol' }

    Note over TS,API: N records = N HTTP requests
```

### Prefetch Solution

The preprocessor intercepts `parseTemplateSet()` on the async path and runs a prefetch phase before iteration:

```mermaid
sequenceDiagram
    participant App as Application
    participant PP as Preprocessor
    participant EP as EntityProvider
    participant API as REST API
    participant TS as TemplateSet

    App->>PP: parseTemplateSet(template, dataset, callback)

    Note over PP: Prefetch Phase
    PP->>PP: Scan template for {~E:~} expressions
    PP->>PP: Follow {~T:~} and {~TIf:~} references one level deep
    PP->>PP: Resolve IDs across dataset, deduplicate
    PP->>PP: Check EntityProvider cache, filter cached IDs

    PP->>EP: getEntitySet('Author', 'FBL~IDAuthor~INN~1,2,3')
    EP->>API: GET /1.0/Authors/FilteredTo/FBL~IDAuthor~INN~1,2,3
    API-->>EP: [{ IDAuthor: 1 }, { IDAuthor: 2 }, { IDAuthor: 3 }]
    EP->>EP: Cache individual records

    Note over PP: Iteration Phase
    PP->>TS: Original parseTemplateSet(template, dataset, callback)

    TS->>EP: getEntity('Author', 1)
    Note over EP: Cache hit!
    TS->>EP: getEntity('Author', 2)
    Note over EP: Cache hit!
    TS->>EP: getEntity('Author', 3)
    Note over EP: Cache hit!

    Note over App,API: 1 batch request instead of N
```

### Prefetch Depth

The prefetch scan follows template references one level deep. This covers the common case where entity expressions are in a child template rendered by the set's iteration template:

```mermaid
graph TD
    SetTemplate["Set Template\n{~T:BookRow:Record~}"] -->|follows reference| BookRow["BookRow Template\n{~D:Record.Title~}\n{~E:Author^Record.IDAuthor^AuthorName~}"]

    BookRow -->|entity discovered| Plan["Prefetch Plan\nEntityType: Author\nIDAddress: Record.IDAuthor"]

    style Plan fill:#9f9,stroke:#333
```

The scan follows `{~T:~}`, `{~TIf:~}`, `{~TIfE:~}`, and `{~TS:~}` references to discover entity expressions in child templates.

### ID Resolution

Entity IDs are resolved from the dataset using dot-notation path walking with support for standard address prefixes:

| Prefix | Resolves From |
|--------|---------------|
| `Record.` | Each record in the dataset |
| `AppData.` | The Pict AppData store |
| `Scope.` | The scope object |
| `Context[N].` | The Nth context array element |

IDs are deduplicated across the dataset and checked against the EntityProvider cache before fetching. Only uncached IDs are included in the batch request.

## Interaction with Template Audit

Both the preprocessor and [pict-template-audit](https://github.com/fable-retold/pict-template-audit) wrap Pict's template methods. If both are active, instantiation order matters:

```mermaid
flowchart LR
    App["parseTemplate()"] --> Audit["Audit Wrapper\n(timing, counting)"]
    Audit --> Preprocessor["Preprocessor Wrapper\n(compile, cache, execute)"]
    Preprocessor --> Original["Original Pict\nparseTemplate"]

    style Audit fill:#ffd,stroke:#333
    style Preprocessor fill:#dff,stroke:#333
```

The preprocessor should be instantiated first (inner wrapper), and the audit second (outer wrapper). This way the audit measures the time of the fast path rather than the original slow path.

## Cache Semantics

The compiled template cache is a `Map<string, Array<Segment>>` keyed by the raw template string. Cache behavior:

- **Key**: The exact template string passed to `parseTemplate()`. Two templates with different whitespace are different cache keys.
- **Lifetime**: Cache entries persist for the lifetime of the preprocessor instance. Call `clearCache()` to invalidate all entries.
- **Thread safety**: JavaScript is single-threaded; no concurrency concerns.
- **Memory**: Each cached entry stores the segment array plus references to existing trie leaf nodes. The overhead per template is proportional to the number of segments, not the string length.
- **Invalidation**: If template expressions are registered or unregistered after compilation, cached segments may reference stale trie leaves. Call `clearCache()` after modifying the pattern trie.

## Module Architecture

```
pict-template-preprocessor/
    source/
        Pict-Template-Preprocessor.js         # Service class, compile, execute, prefetch, wrappers
        Pict-Template-Preprocessor-Graph.js   # TemplateGraph class (nodes, edges, query, export)
    test/
        Pict-Template-Preprocessor_test.js    # 40 unit tests
    docs/
        README.md                             # Documentation landing page
        quickstart.md                         # Getting started guide
        architecture.md                       # This document
        implementation-reference.md           # Behavioral details
        api/                                  # Per-function reference docs
    package.json
```

The preprocessor is a standalone npm package with a single runtime dependency (`fable-serviceproviderbase`). It consumes Pict as a dev dependency for testing. It does not modify any Pict source files; all integration is through runtime method wrapping.

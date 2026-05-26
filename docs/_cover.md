# Pict Template Preprocessor

> Compile-once, execute-many template optimization with dependency graphing and entity batch prefetch

- **Compiled Cache** -- Template strings parsed once, executed many times via segment arrays
- **Dependency Graph** -- Directed graph of template, data, and entity relationships with DOT and JSON export
- **Entity Prefetch** -- Batch-fetch entities at TemplateSet boundaries to eliminate N+1 patterns
- **Transparent** -- Wraps Pict methods without modifying source; one instantiation activates optimization

[Get Started](README.md)
[GitHub](https://github.com/fable-retold/pict-template-preprocessor)

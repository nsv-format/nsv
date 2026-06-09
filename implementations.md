## Implementations

All implementations listed here are licensed under MIT.

If you have ideas on how to make any of these more ergonomic or feel like implementing a brutally optimised version, I very much welcome that.

Disclaimer on performance: multiple implementations/integrations here outperform SOTA CSV parsers in their respective ecosystems when used on the same data.  
This is in no way indicative of quality of said parsers. Modern CSV parsers are impressive pieces of technology, and whatever performance claims are made below are due to the simplicity of NSV parsing, not anyone's ability as implementer.

### Libraries

Repository | Language | Installation | Notes
--- | --- | --- | ---
[nsv-format/nsv-python](https://github.com/nsv-format/nsv-python) | Python | `pip install nsv` | Can monkeypatch itself onto Pandas; uses the Rust implementation on non-resumable paths
[nsv-format/nsv-rust](https://github.com/nsv-format/nsv-rust) | Rust | `cargo add nsv` | Reasonably fast
[nsv-format/nsv-java](https://github.com/nsv-format/nsv-java) | Java | GitHub publishing only | 
[nsv-format/nsv-scala](https://github.com/nsv-format/nsv-scala) | Scala | GitHub publishing only | Does not depend on Java one in any way
[nsv-format/nsv-js](https://github.com/nsv-format/nsv-js) | JavaScript | `npm install @nsv-format/nsv` | It does have type annotations, if you're into that

### Integrations & Tools

Repository | What it is | Installation | Notes
--- | --- | --- | ---
[nsv-format/nsv-cli](https://github.com/nsv-format/nsv-cli) | Standalone utility | `cargo install nsv-cli` | -
[nsv-format/nsv-duckdb](https://github.com/nsv-format/nsv-duckdb) | DuckDB community extension | `INSTALL nsv FROM community; LOAD nsv;` | Faster than CSV on same data
[nsv-format/nsv-format.github.io](https://github.com/nsv-format/nsv-format.github.io) | Web | - | Backs https://nsv-format.github.io
[nsv-format/nsv-tests](https://github.com/nsv-format/nsv-tests) | Test infrastructure | - | Shared test cases, scheduled verification runs


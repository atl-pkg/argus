# Argus

Official test framework for Atlas — structured suites, lifecycle hooks, mocking, spies, and snapshots.

Built on top of Atlas's built-in `test.*` primitives. Argus is the power layer you reach for when `test.equal` isn't enough.

```atlas
import { describe, it, beforeEach } from 'argus';
import { spy } from 'argus';
import { pretty } from 'argus';

describe("UserService", fn(): void {
    let mut db: string = "";

    beforeEach(fn(): void { db = "fresh"; });

    it("reads state after setup", fn(): void {
        test.equal(db, "fresh");
    });

    it("spy tracks calls", fn(): void {
        let s = spy(fn(): number { return 42; });
        s.fn();
        test.equal(s.call_count(), 1);
        test.equal(s.last_return(), 42);
    });
});

pretty(run());
```

## Install

```toml
# atlas.toml
[dependencies]
argus = { git = "https://github.com/atl-pkg/argus", tag = "v0.1.0" }
```

```bash
atlas install
```

## API

### Suites

| Function | Description |
|----------|-------------|
| `describe(name, fn)` | Group related tests |
| `it(name, fn)` | Define a test case |
| `xit(name, fn)` | Skip a test case |
| `beforeEach(fn)` | Run before each `it` in this describe |
| `afterEach(fn)` | Run after each `it` in this describe |
| `beforeAll(fn)` | Run once before all tests in this describe |
| `afterAll(fn)` | Run once after all tests in this describe |
| `run()` | Execute all registered suites → `SuiteResult[]` |

### Mocking

| Function | Description |
|----------|-------------|
| `spy(fn)` | Wrap a function to record calls and return values |
| `stub(value)` | Create a function that always returns `value` |
| `stub_seq(values[])` | Create a function that cycles through `values` in order |

### Snapshots

| Function | Description |
|----------|-------------|
| `matchSnapshot(name, value)` | Assert value matches stored snapshot (writes on first run) |
| `updateSnapshot(name, value)` | Force-overwrite a stored snapshot |

### Reporters

| Function | Description |
|----------|-------------|
| `pretty(results)` | Human-readable colored output |
| `dot(results)` | Compact dot notation (CI-friendly) |
| `json_report(results)` | JSON output to stdout |

## Design

Argus is the **power layer**. The built-in `test.*` namespace handles raw assertions and is always available without any import. Argus adds structure for larger codebases:

| Layer | What | When to use |
|-------|------|-------------|
| `test.*` (built-in) | Raw assertions | Simple scripts, small modules |
| `argus` (this package) | Suites, hooks, mocks, snapshots | Full projects, integration tests |

## License

MIT

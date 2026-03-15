# Argus

**Official test framework for Atlas.** Structured suites, lifecycle hooks, parametric tests, fluent assertions, mocking, fixtures, snapshots, and pluggable reporters — all built on top of Atlas's built-in `test.*` primitives.

Named after the hundred-eyed giant of Greek mythology. Argus sees everything.

---

## Install

```toml
# atlas.toml
[dependencies]
argus = { git = "https://github.com/atl-pkg/argus", tag = "v0.1.0" }
```

```bash
atlas install
```

---

## Quick Start

```atlas
import { describe, it, beforeEach, run } from 'argus';
import { expect } from 'argus';
import { spy, verify } from 'argus';
import { pretty } from 'argus';

describe("Math", fn(): void {
    it("adds correctly", fn(): void {
        expect(1 + 1).toEqual(2);
    });

    it("handles negatives", fn(): void {
        expect(-5 + 3).toBeLessThan(0);
    });
});

describe("UserService", fn(): void {
    let mut state = "";

    beforeEach(fn(): void { state = "fresh"; });

    it("starts fresh each test", fn(): void {
        expect(state).toEqual("fresh");
    });

    it("tracks function calls", fn(): void {
        let s = spy(fn(): string { return "hello"; });
        s.fn();
        s.fn();
        verify(s).calledTimes(2);
        expect(s.last_return()).toEqual("hello");
    });
});

pretty(run());
```

---

## API Reference

### Suites

```atlas
describe(name, fn)                           // Group related tests
describeTags(name, tags[], fn)               // Suite with tags for filtering
it(name, fn)                                 // Define a test case
itOpts(name, fn, tags[], timeout_ms, retry)  // Test with options
xit(name, fn)                                // Skip a test (marked pending)
each(rows[][], fn(row))                      // Parametric tests
beforeEach(fn)   afterEach(fn)               // Per-test lifecycle hooks
beforeAll(fn)    afterAll(fn)                // Per-suite lifecycle hooks
filterTags(tags[])                           // Only run tests matching tags
run()                                        // Execute all suites → SuiteResult[]
reset()                                      // Clear all registered suites
```

**Parametric tests (`each`):**
```atlas
let cases = [
    ["alice", 5],
    ["bob", 3],
    ["charlie", 7],
];

each(cases, fn(row: any[]): void {
    it("name length: " + row[0], fn(): void {
        expect(row[0].length()).toEqual(row[1]);
    });
});
```

**Tags + filtering:**
```atlas
filterTags(["integration"]);  // only run tests tagged "integration"

describeTags("DB tests", ["integration", "slow"], fn(): void {
    itOpts("connects", fn(): void { ... }, ["integration"], 10000, 2);
});
```

---

### Expect (fluent assertions)

```atlas
expect(value)                    // create Expect
  .not()                         // invert next assertion
  .as("label")                   // attach label to failure messages
  .toEqual(expected)             // deep equality
  .toBe(expected)                // value identity
  .toBeTrue()   .toBeFalse()
  .toBeGreaterThan(n)            .toBeGreaterThanOrEqual(n)
  .toBeLessThan(n)               .toBeLessThanOrEqual(n)
  .toApprox(expected, epsilon)   // |actual - expected| <= epsilon
  .toContain(substring)          // string contains
  .toStartWith(prefix)           .toEndWith(suffix)
  .toMatch(pattern)
  .toHaveLength(n)               .toBeEmpty()
  .toBeOk()                      // Result is Ok — returns unwrapped value
  .toBeErr()                     // Result is Err — returns unwrapped error
  .toBeSome()                    // Option is Some — returns inner value
  .toBeNone()
  .toContainItem(item)           // array contains item (deep equality)
```

**Chaining:**
```atlas
expect("hello world")
    .toContain("hello")
    .toHaveLength(11)
    .not().toEndWith("x");
```

**Negation:**
```atlas
expect(user.name).not().toEqual("anonymous");
expect(errors).not().toBeEmpty();
```

---

### Mocking

**Spies** — wrap a function and record every call:

```atlas
// Zero-arg spy
let s = spy(fn(): number { return computeAnswer(); });
s.fn();
s.fn();
expect(s.call_count()).toEqual(2);
expect(s.last_return()).toEqual(42);
expect(s.nth_return(0)).toEqual(42);

// One-arg spy (captures args too)
let s1 = spy1(fn(x: number): number { return x * 2; });
s1.fn(5);
expect(s1.last_arg()).toEqual(5);
expect(s1.last_return()).toEqual(10);
expect(s1.called_with(5)).toBeTrue();
```

**Stubs** — controlled return values:

```atlas
let always = stub(42);             // always returns 42
let seq = stub_seq([1, 2, 3]);     // returns 1, 2, 3, then 3, 3...
let dynamic = stub_fn(fn(): string { return buildValue(); });
```

**Verify** — fluent call-count assertions:

```atlas
verify(s).calledTimes(3);
verify(s).calledOnce();
verify(s).calledAtLeast(1);
verify(s).calledAtMost(5);
verify(s).neverCalled();

verifyAs(s, "fetchUser").calledTimes(2);   // attach label for clear failures
```

---

### Fixtures

Fixtures build a fresh value before each test. No shared mutable globals.

```atlas
import { fixture, fixtureWithTeardown } from 'argus';

describe("UserService", fn(): void {
    let db = fixture(fn(): string { return "empty_db"; });

    beforeEach(fn(): void { db.setup(); });

    it("reads from db", fn(): void {
        expect(db.value()).toEqual("empty_db");
    });
});
```

With teardown:
```atlas
let conn = fixtureWithTeardown(
    fn(): string { return openConnection(); },
    fn(c: any): void { closeConnection(c); },
);

beforeEach(fn(): void { conn.setup(); });
afterEach(fn(): void  { conn.teardown(); });
```

---

### Snapshots

```atlas
import { matchSnapshot, updateSnapshot } from 'argus';

matchSnapshot("api response", response_body);   // writes on first run, asserts after
updateSnapshot("api response", new_body);        // force-overwrite
```

Snapshots are stored as plain text files at `__snapshots__/<name>.snap`.

---

### Reporters

```atlas
import { run, pretty, dot, tap, json_report, verbose, summarize } from 'argus';

let results = run();

pretty(results);       // human-readable: suite tree + totals
verbose(results);      // like pretty but prints every test name
dot(results);          // compact: .....F..S.. (CI-friendly)
tap(results);          // TAP v13 — compatible with any TAP consumer
json_report(results);  // JSON: { summary, suites } to stdout

let s = summarize(results);   // → RunSummary { passed, failed, skipped, all_passed }
```

---

## Design Philosophy

**Two tiers, clear boundary:**

| Layer | What | When |
|-------|------|------|
| `test.*` built-in | Raw assertions. Always available, zero imports. | Small scripts, compiler corpus tests |
| `argus` | Structure, mocking, fixtures, reporters. | Real projects, integration tests, multi-file suites |

**AI-first:** Every API is designed to be trivially generatable by AI agents. Bare specifier imports, consistent naming, no magic configuration files, no CLI flags needed for basic use.

**For AI agents writing Atlas tests:**
- Compiler corpus tests (`pass/*.atlas`, `fail/*.atlas`) — use built-in `test.*` only. No package deps.
- Integration/scenario tests in Atlas projects — use Argus. The structure makes failures readable and test intent clear.

---

## License

MIT

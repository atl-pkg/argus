# Argus

**Official test framework for Atlas.** Structured suites, lifecycle hooks, parametric tests, fluent assertions, mocking, fixtures, snapshots, and pluggable reporters.

Named after the hundred-eyed giant of Greek mythology. Argus sees everything.

---

## Install

```toml
# atlas.toml
[dependencies]
argus = { git = "https://github.com/atl-pkg/argus", tag = "v0.2.0" }
```

```bash
atlas install
```

---

## Import Surface

```atlas
import { describe, it, beforeEach, run } from 'argus';  // suite structure — bare
import { expect } from 'argus';                          // fluent assertions — bare
import { mock } from 'argus';                            // mock.spy(), mock.stub(), mock.verify()
import { snapshot } from 'argus';                        // snapshot.match(), snapshot.update()
import { report } from 'argus';                          // report.pretty(), report.dot(), etc.
import { fixture } from 'argus';                         // fixture factory — bare
```

**Design rule:** Test structure and `expect` are bare — they're domain vocabulary with no collision risk and no framework reads better than `describe("...", fn() { it("...", fn() { ... }) })`. Utilities (`mock`, `snapshot`, `report`) use namespaces to prevent collision with user code.

---

## Quick Start

```atlas
import { describe, it, beforeEach, run } from 'argus';
import { expect } from 'argus';
import { mock } from 'argus';
import { report } from 'argus';

describe("UserService", fn(): void {
    let mut state = "";

    beforeEach(fn(): void { state = "fresh"; });

    it("starts fresh each test", fn(): void {
        expect(state).toEqual("fresh");
    });

    it("tracks function calls", fn(): void {
        let s = mock.spy(fn(): string { return "hello"; });
        s.fn();
        s.fn();
        mock.verify(s).calledTimes(2);
        expect(s.last_return()).toEqual("hello");
    });
});

report.pretty(run());
```

---

## API Reference

### Suite Structure (bare imports)

```atlas
describe(name, fn)                           // group related tests
describeTags(name, tags[], fn)               // suite with tags for filtering
it(name, fn)                                 // define a test case
itOpts(name, fn, tags[], timeout_ms, retry)  // test with options
xit(name, fn)                                // skip (marked pending)
each(rows[][], fn(row))                      // parametric / table-driven tests
beforeEach(fn)   afterEach(fn)               // per-test lifecycle hooks
beforeAll(fn)    afterAll(fn)                // per-suite lifecycle hooks
filterTags(tags[])                           // only run tests matching tags
run()                                        // execute all suites → SuiteResult[]
reset()                                      // clear registered suites
```

**Parametric tests:**
```atlas
each([
    ["alice", 5],
    ["bob", 3],
], fn(row: any[]): void {
    it("name length: " + row[0], fn(): void {
        expect(row[0].length()).toEqual(row[1]);
    });
});
```

---

### expect — fluent assertions (bare import)

```atlas
expect(value)
  .not()                          // invert next assertion
  .as("label")                    // attach label to failure messages
  .toEqual(expected)              // deep equality
  .toBe(expected)                 // value identity
  .toBeTrue()   .toBeFalse()
  .toBeGreaterThan(n)             .toBeGreaterThanOrEqual(n)
  .toBeLessThan(n)                .toBeLessThanOrEqual(n)
  .toApprox(expected, epsilon)    // |actual - expected| <= epsilon
  .toContain(substring)
  .toStartWith(prefix)            .toEndWith(suffix)
  .toHaveLength(n)                .toBeEmpty()
  .toBeOk()                       // Result is Ok — returns unwrapped value
  .toBeErr()                      // Result is Err — returns unwrapped error
  .toBeSome()                     // Option is Some — returns inner value
  .toBeNone()
  .toContainItem(item)            // array contains item (deep equality)
```

**Chaining + negation:**
```atlas
expect("hello world").toContain("hello").toHaveLength(11);
expect(errors).not().toBeEmpty();
expect(user.name).not().toEqual("anonymous");
```

---

### mock — mocking namespace

```atlas
import { mock } from 'argus';

// Spies — wrap a function and record every call
let s  = mock.spy(fn(): number { return 42; });      // 0-arg
let s1 = mock.spy1(fn(x: number): number { x * 2; }); // 1-arg, captures args

s.fn();
expect(s.call_count()).toEqual(1);
expect(s.last_return()).toEqual(42);

s1.fn(5);
expect(s1.last_arg()).toEqual(5);
expect(s1.last_return()).toEqual(10);
expect(s1.called_with(5)).toBeTrue();

// Stubs — controlled return values
let always  = mock.stub(42);             // always returns 42
let seq     = mock.stubSeq([1, 2, 3]);  // returns 1, 2, 3, then 3, 3...
let dynamic = mock.stubFn(fn(): string { return build(); });

// Verify — fluent call-count assertions
mock.verify(s).calledTimes(3);
mock.verify(s).calledOnce();
mock.verify(s).calledAtLeast(1);
mock.verify(s).calledAtMost(5);
mock.verify(s).neverCalled();
mock.verifyAs(s, "fetchUser").calledTimes(2);  // attach label for clearer failures
```

---

### snapshot — snapshot namespace

```atlas
import { snapshot } from 'argus';

snapshot.match("api response", value);   // writes on first run, asserts after
snapshot.update("api response", value);  // force-overwrite stored snapshot
```

Snapshots stored at `__snapshots__/<name>.snap` as plain text (JSON-serialized).

---

### report — reporter namespace

```atlas
import { report } from 'argus';

let results = run();

report.pretty(results);    // human-readable: suite tree + failure details + totals
report.verbose(results);   // like pretty, but also prints passing test names
report.dot(results);       // compact: .....F..S.. (CI-friendly)
report.tap(results);       // TAP v13 — works with prove, tap-reporter, etc.
report.json(results);      // JSON: { summary, suites } to stdout

let s = report.summarize(results);
// s.passed  s.failed  s.skipped  s.duration_ms  s.all_passed
```

---

### fixture — fresh per-test state (bare import)

```atlas
import { fixture, fixtureWithTeardown } from 'argus';

describe("DB tests", fn(): void {
    let db = fixture(fn(): string { return "empty_db"; });

    beforeEach(fn(): void { db.setup(); });

    it("reads from db", fn(): void {
        expect(db.value()).toEqual("empty_db");
    });
});

// With teardown:
let conn = fixtureWithTeardown(
    fn(): string { return openConnection(); },
    fn(c: any): void { closeConnection(c); },
);
beforeEach(fn(): void { conn.setup(); });
afterEach(fn(): void  { conn.teardown(); });
```

---

## Design

**Two tiers, clear boundary:**

| Layer | Import | When |
|-------|--------|------|
| `test.*` built-in | nothing | Raw assertions, compiler corpus tests |
| `argus` | named imports | Real projects, integration tests, mocking |

**Namespace rule:**
- Bare: `describe`, `it`, `expect`, `fixture` — no collision risk, universal test vocabulary
- Namespaced: `mock.*`, `snapshot.*`, `report.*` — collision-prone generic names isolated cleanly

**For AI agents writing Atlas tests:**
- Compiler corpus (`pass/*.atlas`, `fail/*.atlas`) → `test.*` only, no package deps
- Integration/scenario `.atl` files → Argus

---

## License

MIT

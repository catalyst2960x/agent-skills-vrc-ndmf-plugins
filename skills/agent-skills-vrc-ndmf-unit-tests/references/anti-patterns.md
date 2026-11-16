# Anti-Patterns

Patterns that make a test suite pass while the product is
wrong, or fail while the product is right. For review of
test code.

## 1. Asserting on the scene side

The build writes to a clone; the author's scene objects keep
their values by design. A case that reads them after a build
asserts the non-destructiveness property by accident and
the generation property not at all. Read the built avatar.

## 2. Severity by message text

Asserting that a report's message contains a phrase couples
the suite to diagnostic wording. The contract is the
severity class; assert Information or NonFatal and the
absence of Error.

## 3. Presence-only assertions

A layer that exists with weight 0, a clip with no curves, a
parameter declared with the wrong type all pass a presence
check. Where a value decides the behaviour, the case
compares the value.

## 4. The skipped component that passes

A case asserting only "no Error was reported" also passes
when the component was silently skipped and nothing was
generated. Pair every report assertion with the generation
assertion for the same build.

## 5. Pipeline-only testing

Every case through the full pipeline makes each failure
ambiguous between the pass, the ordering, and the other
systems. Pass-level properties are tested at the direct
context; the pipeline is reserved for agreement and
interoperation cases.

## 6. Shared mutable fixtures

One fixture built once and mutated by several cases makes
the suite order-dependent. A fixture is built per test and
tracked; teardown returns the project to its prior state.

## 7. Tracking a persistent asset

Teardown destroys what the base tracks. Tracking a project
asset — a prefab, the author's controller — destroys it.
The base rejects persistent objects; instances of assets
are tracked, never the assets.

## 8. Disposing what the framework owns

Wrapping framework objects in using blocks or disposing
them by hand fights the framework's own lifetime handling —
a BuildContext is not disposable, and disposing one of its
scopes early unregisters what later passes read. Create,
use, and leave lifetimes to the framework.

## 9. Exact float equality on computed values

Weights and thresholds that pass through computation are
compared with a tolerance; exact equality is reserved for
values the mechanism copies unchanged, such as a constant
curve's key value.

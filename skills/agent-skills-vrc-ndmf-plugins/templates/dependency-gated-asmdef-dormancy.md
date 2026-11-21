# Template: Dormancy gated on a required dependency package

## When it applies

A product's Editor assembly references a required dependency
package's assembly directly: the editor code names the
dependency's types throughout, so separating the
dependency-touching code into an assembly of its own is not
the shape. The product must still import without compile
errors, and stay inert, when the dependency is absent — the
product installed before the dependency, or the dependency
removed afterwards. An asmdef reference is not conditional: a
reference to an assembly that is not installed fails
compilation of the whole referencing assembly, and #if
cannot gate a reference. The only gate that holds is
excluding the referencing assembly from compilation while
the dependency is absent.

This differs from the optional-integration form (P-22),
where the gated assembly is a separated extra holding only
an optional package's code and the product's own assemblies
never reference that package. Here the dependency is
required and the product's own Editor assembly carries the
reference, so the Editor assembly itself is gated. It also
differs from making a package dormant until its own
generation step deletes the gating asmdefs: that switch is
one-way and thrown by the package itself, while this gate
follows the dependency package's presence and opens and
closes on its own.

## Requirements

- The Editor asmdef keeps its direct reference to the
  dependency's assembly. Its versionDefines define a symbol
  keyed to the dependency package, with the expression
  stating the declared version range in interval notation —
  [2.4.1,3.0.0) for a floor of 2.4.1 below 3.0.0 — and the
  symbol is listed in the same asmdef's defineConstraints.
  Unity compiles an assembly only when every define
  constraint holds; with the package absent the symbol is
  undefined and the assembly is excluded whole, so the
  unresolved reference is never reached and produces no
  error.
- The expression grammar has no npm-style operators: a
  >=2.4.1 form is rejected with ExpressionNotValidException
  and stops compilation altogether. The interval notation
  is part of the mechanism, not a style choice.
- No #if gating for presence inside the gated assembly: with
  the constraint met, the code compiles against the
  dependency as written.
- The Runtime assembly that carries the product's components
  is not gated and does not reference the dependency.
  Components already placed in scenes stay as data while the
  Editor assembly is excluded, and do not become Missing
  Scripts.
- The gate keys on the dependency's VPM package
  registration. A dependency placed under Assets (installed
  from a unitypackage) does not open the gate.
- Judge the gate by the compilation graph (for example
  CompilationPipeline.GetAssemblies()), not by editor
  behaviour or console error counts: in a running editor
  session the previously compiled assembly remains loaded
  and keeps executing, and an excluded assembly reports
  nothing.
## Skeleton

The Editor assembly's asmdef:

```json
{
  "name": "Example.Editor",
  "references": ["Example.Dependency"],
  "includePlatforms": ["Editor"],
  "versionDefines": [
    {
      "name": "com.example.dependency",
      "expression": "[1.0.0,2.0.0)",
      "define": "EXAMPLE_DEPENDENCY_PRESENT"
    }
  ],
  "defineConstraints": ["EXAMPLE_DEPENDENCY_PRESENT"]
}
```

The Runtime assembly's asmdef — no gate, and no reference to
the dependency:

```json
{
  "name": "Example.Runtime",
  "references": []
}
```

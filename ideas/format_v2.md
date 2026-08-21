# API Notes Format V2: Selector-Based Overload Matching

Status: post-meeting design proposal for discussion. This document proposes a
selector/effect model for C++ overload-aware API Notes, while keeping the
existing `Tags` / `Methods` hierarchy as the main public shape for this work.
The first upstreamable slice keeps `Name` as the primary method-entry lookup key
and introduces `Where.Parameters` for exact explicit parameter matching. The
broader v2 selector model remains future work.

## How To Read This Document

This document has two syntax layers:

1. **First upstreamable slice:** existing `Name` fields inside `Tags` /
   `Methods` entries, with optional `Where.Parameters` blocks for exact explicit
   parameter lists.
2. **Future v2 direction:** additional selector fields such as `Object`,
   `ParameterConstraints`, match cardinality, and language gates.

The important point is that selectors do not require a complete top-level YAML
redesign. `Tags` / `Methods` can remain the enclosing API Notes structure while
individual method entries gain clearer selector semantics.

Syntax status used below:

- `Tags` / `Methods` is current API Notes syntax and is the main C++ method
  proposal surface.
- `Name` remains current API Notes syntax and is the primary C++ method lookup
  key in the first upstreamable overload-aware C++ method patch.
- `Where.Parameters` is proposed public syntax for the first residual narrowing
  constraint in that first patch.
- `Object`, `ParameterConstraints`, match cardinality, and `Language` are future
  v2 selector-model vocabulary.
- `Functions` is used only for standalone/free-function examples and is not part
  of the committed C++ method slice.

A broader top-level YAML redesign is out of scope for this proposal. Unless
explicitly marked as a free-function future example, examples in this document
use the existing `Tags` / `Methods` hierarchy.

## Design Goals

- Preserve existing broad matching behavior for existing API Notes.
- Make overload-specific matching an explicit narrowing operation.
- Separate declaration-selection constraints from annotation effects.
- Define what omission means: omitted constraints are wildcards.
- Define what explicit constraints mean: present constraints narrow matching.
- Represent C++ object qualifiers directly instead of hiding them behind
  `Position: -1`.
- Distinguish exact explicit parameter matching from intentionally partial
  filters.
- Provide a path to useful diagnostics for missing, ambiguous, and duplicate
  matches.
- Leave room for future templates, richer type matching, and mixed
  C/C++ headers while explicitly excluding them from this proposal's committed
  implementation scope.

## Design Decisions

### Decision 1: Keep `Name` As The Primary Method Key And Use `Where` To Narrow

Keep the existing method-entry `Name` field as the user-facing key for selecting
the initial method candidate set. Use `Where` for additional narrowing
constraints inside that existing API Notes entry.

```yaml
Name: WidgetModule
Tags:
  - Name: Widget
    Methods:
      - Name: setValue
        Where:
          Parameters:
            - int
        SwiftName: setValue(_:)
```

This means:

```text
For methods in tag `Widget` named `setValue`, where the explicit parameter
signature is `(int)`, apply `SwiftName: setValue(_:)`.
```

This shape is preferred for the first upstreamable slice because it keeps the
existing name-based lookup path intact. The `Where` block contains residual
selector constraints such as `Parameters` rather than replacing the existing
method-entry key.

Within that residual selector, `Where` is preferred over `Selector`, `Match`, and
`Matchee`.

- `Where` is Swift-like. It resembles Swift constructs such as
  `extension T where ...`, where a clause restricts the declarations or types
  being described.
- `Where` reads declaratively in YAML: where these constraints hold, apply these
  effects.
- `Selector` risks confusion with Objective-C selectors, which are already a
  concept in API Notes and Clang.
- `Match` is a good internal or conceptual term, but it sounds more like an
  operation than a declarative clause.
- `Matchee` should be avoided. It is uncommon, awkward at the point of use, and
  sounds like the entity being matched rather than the rule used to match it.

Recommended terminology:

```text
Name         = primary method-entry lookup key
Where clause = user-facing residual matching block inside an entry
match        = conceptual/internal operation
candidate    = declaration being tested against the Where clause
effect       = metadata applied after matching
```

### Decision 2: Keep Effects Outside `Where`

Fields outside `Where` modify or decorate the matched declarations.

```yaml
Tags:
  - Name: Widget
    Methods:
      - Name: makeWidget
        Where:
          Parameters: []
        SwiftName: Widget.make()
        Availability: nonswift
```

Current and proposed API Notes contain two different kinds of fields:

1. Fields that identify declarations.
2. Fields that change how matched declarations are imported.

Mixing these concepts makes fields such as `Type` confusing. In one context,
`Type` can mean "match this source-language type". In another context, a type
field can be read as "change the imported type" or "record a different
Swift-facing type". The v2 format avoids that ambiguity by making the split
structural:

```yaml
Where:
  # matching/filtering constraints

SwiftName:
Availability:
Nullability:
  # effects applied to matched declarations
```

If a future API Notes feature needs to describe a Swift-facing type effect, it
should use a distinct effect name such as `SwiftType` rather than overloading
`Type` inside `Where`.

Rule:

```yaml
Where:
  Type: ...
```

means source-language matching.

```yaml
SwiftType: ...
```

would mean a Swift import effect if such an effect is added later.

### Decision 3: Omitted Constraints Are Wildcards

A missing field means "do not constrain by this property."

```yaml
Tags:
  - Name: Logger
    Methods:
      - Name: log
        SwiftName: log(_:)
```

For overloaded methods, this matches every method named `log` in `Logger`:

```cpp
struct Logger {
  void log(int);
  void log(double);
  void log(const char *);
};
```

This preserves the current mental model of API Notes: a simple name-based entry
can apply to an entire overload set. Existing projects may rely on notes that
are broad enough to match several declarations, so omitted constraints must not
silently narrow existing behavior.

In the first upstreamable slice, this is why `Where` is absent from broad
name-only entries: there are no residual constraints to write. `Where` remains
the public place for overload-specific narrowing when a note needs it, such as
`Where.Parameters`.

### Decision 4: Explicit Constraints Narrow Matching

If a field is present, it constrains matching.

```yaml
Tags:
  - Name: Gauge
    Methods:
      - Name: value
        Where:
          Parameters: []
        SwiftName: currentValue()
```

This matches `Gauge::value()` and does not match `Gauge::value(int)`.

The predictable rule is:

```text
field omitted = wildcard
field present = exact constraint
```

This exactness rule applies to explicit constraints in the v2 model. In the
first implementation slice, exact explicit parameter matching is expressed with
public `Where.Parameters` syntax rather than by reinterpreting existing
method-note `Parameters` entries outside `Where`.

### Decision 5: Use `Where.Parameters` For Exact Explicit Parameters

Use `Where.Parameters` when the user intends to describe the full explicit
parameter list of an overload. This is the first residual narrowing constraint
supported under `Where`.

```yaml
Tags:
  - Name: Image
    Methods:
      - Name: resize
        Where:
          Parameters:
            - int
            - int
        SwiftName: resize(width:height:)
```

`Where.Parameters` means exact arity and exact parameter order. It does not mean
"some parameter is `int`".

For zero parameters:

```yaml
Tags:
  - Name: Builder
    Methods:
      - Name: build
        Where:
          Parameters: []
        SwiftName: build()
```

`Where.Parameters: []` is different from omitting `Where.Parameters`. It
explicitly means "no explicit source parameters."

The first implementation slice only supports exact explicit-parameter matching.
Templates, receiver/object constraints, cv/ref-qualified member matching,
explicit object parameters, partial parameter filters, and advanced diagnostics
remain future work.

### Decision 6: Use `ParameterConstraints` For Intentional Partial Matching

Use `ParameterConstraints` when the user intentionally gives only partial
constraints over an overload set.

```yaml
Tags:
  - Name: Config
    Methods:
      - Name: configure
        Where:
          ParameterConstraints:
            - Position: 0
              Type: int
        SwiftName: configure(_:)
```

This matches overloads whose first parameter is `int`, regardless of the other
parameters. First-wave `Where.Parameters` is exact by design. If partial matching
is needed later, a distinct field such as `ParameterConstraints` can make that
intent explicit instead of overloading the first-wave `Parameters` list:

```text
Where.Parameters     = exact explicit parameter list constraint
ParameterConstraints = partial filtering over an overload set
```

Review status: this split still needs John and Egor's input. The first
upstreamable slice commits only to exact `Where.Parameters` matching.
`ParameterConstraints` remains future design vocabulary unless reviewers agree
that both spellings should become part of the public model.

Multiple `ParameterConstraints` entries in one `Where` clause are conjunctive. A
candidate declaration must satisfy every listed parameter constraint.

```cpp
struct Mixer {
  void f(int,   int,   float);  // A
  void f(float, int,   float);  // B
  void f(float, float, int);    // C
};
```

```yaml
Tags:
  - Name: Mixer
    Methods:
      - Name: f
        Where:
          ParameterConstraints:
            - Position: 0
              Type: float
            - Position: 1
              Type: int
        SwiftName: fFloatInt(_:_:_:)
```

Candidate-set view:

```text
all f overloads               = {A, B, C}
Position 0 is float           = {B, C}
Position 1 is int             = {A, B}
both constraints in one Where = {B}
```

Disjunctive behavior should not be implicit. If OR-style matching is needed
later, users can write separate entries, or a future design can add an explicit
construct such as `AnyOf`.

Separate entries can overlap:

```yaml
Tags:
  - Name: Mixer
    Methods:
      - Name: f
        Where:
          ParameterConstraints:
            - Position: 0
              Type: float
        Availability: nonswift

      - Name: f
        Where:
          ParameterConstraints:
            - Position: 1
              Type: int
        Availability: nonswift
```

The first entry matches B and C. The second entry matches A and B. Overload B is
matched by both entries. This is fine when effects are compatible. If the two
entries instead applied conflicting effects, such as different `SwiftName`
values, future v2 should diagnose the conflict instead of relying on ordering.

`ParameterConstraints` is future v2 design material. It is not part of the first
upstreamable slice.

### Decision 7: Use `Object` For C++ Member-Function Object Qualifiers

C++ member-function qualifiers apply to the implicit object parameter, not to
the normal explicit parameter list. Therefore, represent them under `Object`.

```yaml
Tags:
  - Name: Builder
    Methods:
      - Name: build
        Where:
          Parameters: []
          Object:
            Const: true
            Ref: lvalue
        SwiftName: buildImmutable()
```

This matches:

```cpp
struct Builder {
  Result build() const &;
};
```

This is clearer than a generic `Qualifiers` block because it says what is being
qualified: the object or receiver.

Conceptually:

```cpp
void f() &;
```

acts like a method whose implicit object parameter is an lvalue reference.

```cpp
void f() &&;
```

acts like a method whose implicit object parameter is an rvalue reference.

The YAML should reflect that these qualifiers are not ordinary parameter types.

C++23 explicit object parameters should fit this same conceptual receiver/object
space in a future design. Gábor's review pointed at this C++ feature as an
important future generalization point with a Godbolt example:
https://godbolt.org/z/xv8MjoTGr

Explicit object parameter syntax is future design material and should not block
the first overload-matching work. One important constraint for that future design
is semantic stability: if a C++ API changes between implicit-object member
syntax and explicit-object syntax without changing the meaning of the function,
the same API Notes entry should ideally continue to match.

Proposed implicit-object fields:

```yaml
Object:
  Const: true | false
  Volatile: true | false
  Ref: none | lvalue | rvalue
```

Semantics:

| Field | Omitted | Present |
| --- | --- | --- |
| `Const` | wildcard | exact const / non-const constraint |
| `Volatile` | wildcard | exact volatile / non-volatile constraint |
| `Ref` | wildcard | exact ref-qualifier constraint |

Ref values:

| Value | C++ spelling | Meaning |
| --- | --- | --- |
| `none` | no `&` or `&&` | explicitly unqualified member function |
| `lvalue` | `&` | lvalue-qualified member function |
| `rvalue` | `&&` | rvalue-qualified member function |

Edge case: omitted versus explicit ref qualifier.

```cpp
struct S {
  void f();
  void f() &;
  void f() &&;
};
```

This matches every overload named `f`:

```yaml
Tags:
  - Name: S
    Methods:
      - Name: f
        SwiftName: f()
```

This matches only the unqualified member function:

```yaml
Tags:
  - Name: S
    Methods:
      - Name: f
        Where:
          Parameters: []
          Object:
            Ref: none
        SwiftName: fUnqualified()
```

This matches only the lvalue-qualified overload:

```yaml
Tags:
  - Name: S
    Methods:
      - Name: f
        Where:
          Parameters: []
          Object:
            Ref: lvalue
        SwiftName: fLValue()
```

This matches only the rvalue-qualified overload:

```yaml
Tags:
  - Name: S
    Methods:
      - Name: f
        Where:
          Parameters: []
          Object:
            Ref: rvalue
        SwiftName: fRValue()
```

Edge case: composing `Const` and `Ref`.

```cpp
struct S {
  void f() &;
  void f() const &;
};
```

`Object.Ref: lvalue` matches both overloads. Adding `Object.Const` narrows the
candidate set:

```yaml
Tags:
  - Name: S
    Methods:
      - Name: f
        Where:
          Object:
            Const: false
            Ref: lvalue
        SwiftName: fMutableLValue()
```

matches only `f() &`, while:

```yaml
Tags:
  - Name: S
    Methods:
      - Name: f
        Where:
          Object:
            Const: true
            Ref: lvalue
        SwiftName: fConstLValue()
```

matches only `f() const &`.

Edge case: invalid `Object`.

```cpp
struct S {
  static void f();
};
```

This should diagnose in the future v2 model:

```yaml
Tags:
  - Name: S
    Methods:
      - Name: f
        Where:
          Object:
            Ref: lvalue
        SwiftName: invalid()
```

Static methods and non-member functions do not have C++ member-function object
qualifiers.

`Object` is future v2 design material. It is not part of the first upstreamable
slice.

### Decision 8: Parameter References Are Not Object References

C++ uses similar-looking syntax in two different places. A `&`, `&&`, `*`, or `**` that appears in an explicit parameter type is part of the function signature. A `&` or `&&` that appears after a member function's parameter list is a qualifier on the implicit object/receiver, often thought of as `this`.

That means these two cases are different:

```cpp
struct Buffer {};

struct Consumer {
  void consume(Buffer &);
  void consume(Buffer &&);
};
```

These are ordinary parameter-type overloads. The first-wave spelling describes
them inside `Where.Parameters`:

```yaml
Tags:
  - Name: Consumer
    Methods:
      - Name: consume
        Where:
          Parameters:
            - "Buffer &"
        SwiftName: consumeBorrowed(_:)

      - Name: consume
        Where:
          Parameters:
            - "Buffer &&"
        SwiftName: consumeOwned(_:)
```

By contrast:

```cpp
struct Builder {
  Result build() &;
  Result build() &&;
};
```

These are receiver/object-qualified method overloads. They have no explicit
source parameters. A future v2 spelling should describe them with `Object.Ref`:

```yaml
Tags:
  - Name: Builder
    Methods:
      - Name: build
        Where:
          Parameters: []
          Object:
            Ref: lvalue
        SwiftName: buildFromLValue()

      - Name: build
        Where:
          Parameters: []
          Object:
            Ref: rvalue
        SwiftName: buildFromRValue()
```

`Object.Ref` describes the implicit object parameter of a non-static C++ member
function. It does not describe an explicit `T &` or `T &&` parameter.

Structured reference type objects are future v2 syntax. In the first
upstreamable slice, explicit parameter reference types are handled only as source
type spellings in `Where.Parameters`.

### Future Idea: Match Cardinality

Match cardinality describes how many declarations a `Where` clause is allowed to
match. One possible future spelling is `Expect`:

```yaml
Expect: any
Expect: atLeastOne
Expect: one
```

Semantics:

| Value | Meaning |
| --- | --- |
| `any` | zero, one, or many matches are allowed |
| `atLeastOne` | zero matches is a diagnostic; multiple matches are allowed |
| `one` | exactly one match is required; zero or multiple matches diagnose |

The current behavior allows broad entries to apply to multiple overloads, and
this should not be broken. Explicit parameter selection often expresses stronger
intent, but cardinality should not be part of the first design.

This is deliberately future syntax. It can improve diagnostics, but it is also a
footgun: API Notes files could start failing when a later library version adds a
new overload that changes match cardinality. That brittleness may be useful for
some authors and surprising for others, so this idea should stay easy to cut or
soften after further review.

For the first upstreamable slice, the important behavior is narrower: an
explicit `Where.Parameters` selector must not silently fall back to broad
name-only matching, and malformed or duplicate exact-parameter selectors should
diagnose.

Later `ParameterConstraints` may cover many useful cases more directly by
letting authors narrow the candidate set instead of asserting how many matches a
selector should produce. Advanced cardinality controls should therefore remain
future design material.

### Decision 10: Use Spelling-Based Type Matching With Limited Normalization

Inside `Where`, `Type` means source-language type matching.

For the committed scope, parameter type matching should compare written or sugared source type spellings after limited normalization. This is deliberately less ambitious than full semantic type equivalence.

Current proposal boundary:

- `int` matches `int`
- whitespace normalization is in scope
- simple qualifier-order normalization, such as `const int *` and `int const *`, is desired but not required for the first upstreamable slice
- top-level `const` on by-value parameters should not create a distinct overload identity
- `using Count = int;` does not make `Count` automatically match `int`
- template semantic equivalence is out of scope
- typedef / using alias equivalence is out of scope

Examples:

```cpp
using Count = int;

struct Counter {
  void setCount(Count);
  void setRawInt(int);
};
```

`Type: int` should not automatically match `setCount(Count)`.

Top-level `const` on a by-value parameter does not make a distinct overload:

```cpp
void f(int);
void f(const int);
```

For overload matching, an `int` parameter description should match this
declaration shape regardless of whether the source spelling includes top-level
`const`. The exact API Notes spelling policy remains open: the parser could
reject top-level `const` in by-value parameter descriptions, or it could accept
and normalize it away. Either way, top-level `const` should not produce a
separate overload identity.

```cpp
struct Box {
  void setVector(std::vector<int>);
};
```

Template spelling and template semantic equivalence are future-only questions.
This proposal does not attempt to decide whether alternative template spellings
should match. Keep the first-slice type-matching decision focused on explicit
parameter type spellings and see `format_v2_templates.md` for future-only template
selector sketches, including template parameter structure and template argument
matching.

Qualified names are also future design material. The model should leave room for namespace-aware lookup because popular short names can appear across multiple libraries used by the same application, so fully qualified names may become important relatively soon. Inline namespaces and spelling choices around fully qualified names need more thought before becoming public syntax.

### Decision 11: Add Future Language Restrictions

Use `Language` or `Languages` when an entry only applies in certain language modes. For C++ methods, the selector can stay inside the existing `Tags` / `Methods` hierarchy:

```yaml
Tags:
  - Name: Parser
    Methods:
      - Name: parse
        Where:
          Language: cxx
          Parameters:
            - int
        SwiftName: parseInt(_:)
```

This means the note is intended only when the selected declaration is considered in a C++ language context.

The same idea can apply to future free-function selectors as well. Mixed C/C++ headers are the clearest motivation there, because C has no overloads while C++ does.

Free-function example, future-only:

```c
#ifdef __cplusplus
extern "C" {
#endif

void foo(size_t count);

#ifdef __cplusplus
}
#endif
```

```yaml
Functions:
  - Name: foo
    Where:
      Languages: [c, cxx]
      Parameters:
        - size_t
    SwiftName: foo(_:)
```

If an explicit parameter constraint is present and misspelled, it should not be
silently ignored just because C has no overloads. A future cardinality mechanism
such as `Expect` should control whether that is diagnosed.

`Language` and `Languages` are future v2 design material. The first upstreamable
slice can defer public language constraints.

If language gates become more relevant, collect concrete examples from Gábor and
Egor where Clang currently needs hardcoded C/C++ differences, such as library
macros that expand differently in C and C++. Those examples would make it easier
to judge whether some hardcoded compiler logic could eventually move into API
Notes.

## Proposed V2 Shape Within `Tags` / `Methods`

This is the main proposed v2 direction. It keeps `Tags` / `Methods` as the enclosing API Notes structure and introduces an explicit selector block inside each method note.

```yaml
Name: WidgetModule
Tags:
  - Name: Widget
    Methods:
      - Name: setValue
        Availability: nonswift

      - Name: setValue
        Where:
          Parameters:
            - int
        SwiftName: setIntValue(_:)

      - Name: status
        Where:
          Parameters: []
          Object:
            Const: true
        SwiftName: immutableStatus()
```

Expected behavior:

- the first method note is broad over all `setValue` overloads
- the second method note selects `setValue(int)`
- the third method note is future receiver-sensitive matching
- the shape keeps `Tags` / `Methods` and changes only the inside of a method note

## Short Reference

| Field | Meaning |
| --- | --- |
| `Tags` | existing API Notes type-context list |
| `Methods` | existing API Notes method list inside a tag |
| method-entry `Name` | primary source method name selector |
| `Where` | residual selector constraints used to narrow source declarations |
| `Parameters` inside `Where` | exact explicit parameter list |
| `ParameterConstraints` | intentionally partial parameter filters |
| `Object` | C++ member-function object qualifiers |
| match cardinality / `Expect` | future expected-match-count idea |
| fields outside `Where` | effects applied to matched declarations |

## Compatibility Model

Existing API Notes remain valid and keep their current matching behavior. The
first upstreamable slice adds a new selector form instead of changing the meaning
of existing method entries.

Legacy source:

```yaml
Methods:
  - Name: setValue
    SwiftName: setValue(_:)
```

Internal v2-like interpretation:

```yaml
Methods:
  - Name: setValue
    SwiftName: setValue(_:)
```

This preserves the current broad behavior: a name-only method note can still
apply to all overloads named `setValue`.

Existing source syntax does not need to migrate immediately. A project that wants
overload-specific matching opts in by writing `Where.Parameters`:

```yaml
Methods:
  - Name: setValue
    Where:
      Parameters:
        - int
    SwiftName: setIntValue(_:)
```

Current method entries that use `Parameters` without `Where` are not
reinterpreted as overload selectors in this slice:

```yaml
Methods:
  - Name: setValue
    Parameters:
      - Position: 0
        Type: int
    SwiftName: setValue(_:)
```

The entry above remains current API Notes syntax. The first upstreamable patch
does not assign new overload-selection semantics to `Parameters[].Type`. That
avoids changing the behavior of existing API Notes files.

## First Upstreamable Slice

The first implementation should introduce public `Where.Parameters` syntax for
C++ methods under `Tags`. This is distinct from existing method-note
`Parameters` entries outside `Where`, so it gives the feature a clean user model
and avoids redefining existing API Notes fields.

Goal:

```text
Allow `Tags` / `Methods` entries with `Where.Parameters` to distinguish ordinary C++ method overloads by exact explicit parameter lists.
```

Supported source syntax:

```yaml
Name: WidgetModule
Tags:
  - Name: Widget
    Methods:
      - Name: setValue
        Where:
          Parameters:
            - int
        SwiftName: setIntValue(_:)

      - Name: setValue
        Where:
          Parameters:
            - double
        SwiftName: setDoubleValue(_:)
```

This syntax is already the public selector form. It can be lowered internally as
an exact selector:

```yaml
Name: setValue
Where:
  Parameters:
    - int
SwiftName: setIntValue(_:)
```

### First-Slice Rules

- Public `Where.Parameters` syntax is required for the first upstreamable patch.
- Legacy method entries without `Where` keep existing behavior.
- Existing `Parameters[].Type` entries without `Where` are not overload
  selectors.
- Method-entry `Name` selects the initial C++ method candidate set inside the
  enclosing tag.
- Omitted `Where.Parameters` keeps broad name-based matching.
- `Where.Parameters: []` means exact zero-parameter list.
- `Where.Parameters` means exact arity, exact order, and exact explicit
  source parameter type sequence under the chosen normalization policy.
- `Position: -1` keeps existing receiver-side annotation behavior in legacy
  entries and does not participate in first-slice overload identity.
- Duplicate `Where.Parameters` selectors in the same context diagnose.
- Same-name methods with different `Where.Parameters` selectors are allowed.
- A `Where.Parameters` selector must not silently fall back to broad name-only
  matching.

Example:

```yaml
Methods:
  - Name: value
    SwiftName: value()

  - Name: value
    Where:
      Parameters: []
    SwiftName: currentValue()
```

The first note is legacy syntax and remains broad. The second note selects only
`value()`.

### First-Slice Non-Goals

- receiver/object constraints for cv/ref-qualified methods
- explicit object parameters
- partial `ParameterConstraints`
- public cardinality / `Expect` syntax
- templates
- typedef / using alias equivalence
- globals and fields
- C/C++ language gates
- advanced diagnostics beyond malformed, duplicate, and unmatched exact
  `Where.Parameters` selectors
- Objective-C selector redesign

### Upstream Patch Plan

The first slice should be split into small, reviewable LLVM patches:

1. Add YAML parser/model support for method-entry `Name` plus optional
   `Where.Parameters`, with parser tests only.
2. Add binary serialization and round-trip support for the residual selector,
   proving same-name/different-parameter entries survive.
3. Add lookup/Sema support for exact explicit-parameter residual matching, with
   behavior tests and legacy broad-matching tests.
4. Add core diagnostics for malformed selectors, duplicate exact-parameter
   selectors, unmatched exact selectors where Clang can form the candidate set,
   and no silent fallback to broad name-only matching.
5. Add edge-case coverage for empty parameter lists, default arguments, static
   methods, supported spelling normalization, and by-value top-level `const`.

## Implementation Shape

The implementation should extend the existing API Notes pipeline:

```text
YAML
  -> API Notes model
  -> binary API Notes storage
  -> lookup by context and method name
  -> residual selector filtering by exact explicit parameters
  -> Sema applies effects to matching CXXMethodDecl
```

The important architectural shift is that `(context, method name)` is no longer the whole identity for all C++ method notes. It becomes the initial candidate set. The residual selector determines whether a note is broad or parameter-specific.

Recommended internal model for the first slice:

```text
primary lookup:
  C++ tag context + method name

residual selector:
  legacy broad name-only
  exact explicit parameter list

effects:
  existing API Notes payload
```

The YAML model should accept two C++ method-entry forms:

```text
legacy form:
  Name + existing method-note fields

selector form:
  Name + optional Where.Parameters + existing effect fields
```

Legacy entries remain on the existing broad path. Selector-form entries use
`Name` to build the candidate set and `Where` for the residual selector. This
keeps the prototype's useful implementation insight while aligning it with the
cleaner v2 selector model.

## Diagnostics

Required for the first slice:

- duplicate `Where.Parameters` selectors diagnose
- malformed selector syntax diagnoses
- malformed `Where.Parameters` entries diagnose
- unmatched `Where.Parameters` selectors diagnose when Clang can determine the
  enclosing method candidate set
- `Where.Parameters` selectors do not silently degrade to broad matching

Desired for the first slice:

- diagnostics show the requested parameter list and nearby candidate overloads

Future v2 diagnostics:

- future cardinality failure for ambiguous matches
- future cardinality failure for missing matches
- invalid `Object` constraints on static or non-member functions
- ambiguous namespace or type-name matches
- conflicting effects from overlapping partial constraints

## Tests

First-slice parser and storage tests:

- parser accepts method-entry `Name` plus optional `Where.Parameters` for C++
  methods under `Tags`
- parser accepts `Where.Parameters` with zero or more type spellings
- two same-name method notes with different `Where.Parameters` selectors roundtrip
- duplicate same-name `Where.Parameters` selectors diagnose
- omitted `Where.Parameters` and `Where.Parameters: []` remain distinct
- legacy `Parameters[].Type` behavior remains unchanged
- legacy broad notes and `Where.Parameters` notes for the same method name both survive
  serialization without one overwriting the other

First-slice Sema/API Notes tests:

- `setValue(int)` and `setValue(double)` receive different effects through
  `Where.Parameters`
- a name-only method note still applies broadly to all overloads
- a method note with `Name` and no `Where.Parameters` still applies broadly to
  all overloads
- exact zero-parameter `Where.Parameters` note does not match one-parameter overload
- `Where.Parameters` note plus broad note can coexist
- a `Where.Parameters` selector with no matching overload does not fall back to the broad
  name-only path
- alias equivalence is not implied
- harmless whitespace differences in supported type spellings behave according
  to the chosen normalization policy
- unsupported type spelling differences do not accidentally match
- by-value top-level `const` does not create a separate overload identity, and
  `int` matches the declaration shape regardless of whether the source spelling
  includes top-level `const`
- default arguments do not reduce the declared parameter count
- static methods can match by exact explicit parameters
- `Position: -1` does not distinguish `status()` from `status() const`

Future v2 tests:

- `Object.Const` distinguishes const and non-const methods
- `Object.Volatile` distinguishes volatile and non-volatile methods
- `Object.Ref` distinguishes no ref, `&`, and `&&`
- `Object` constraints diagnose on global functions and static methods
- `ParameterConstraints` behave conjunctively
- separate partial-constraint entries that overlap either merge compatible
  effects or diagnose conflicting effects according to policy
- future cardinality syntax diagnoses zero or multiple matches according to policy
- future cardinality syntax preserves broad legacy-style matching where requested
- future cardinality syntax diagnoses ambiguous short-name or namespace matches
- language constraints handle mixed C/C++ headers predictably
- structured type matching covers the accepted normalization cases without
  implying alias or template equivalence

## Meeting Feedback Coverage

Addressed directly:

- selector fields and effect fields are separated through `Where`
- existing broad matching is preserved through omitted constraints
- explicit overload information narrows candidates instead of replacing the
  whole API Notes model
- exact signatures and partial filters are separate concepts
- multiple partial filters are conjunctive
- object qualifiers get an explicit `Object` model instead of relying on
  `Position: -1`
- first-slice `Where.Parameters` selectors have required diagnostics for malformed
  or unmatched selectors where Clang can determine the candidate set
- future missing and ambiguous explicit matches have a possible path to
  diagnostics through cardinality controls
- type matching has a documented first-slice boundary
- C/C++ mixed-header behavior has a future `Language` model
- templates and aliases are acknowledged as future work rather than hidden

Still open:

- exact v2 type canonicalization policy beyond first-slice normalization
- whether v2 should parse structured type objects or mostly use type spellings
- whether explicit cardinality controls are needed at all
- exact source-compatibility transition plan for public `Where`
- how much namespace qualification should be required
- how template matching could possibly be integrated

## Review Follow-Ups

Open review items from Gábor's 2026-06-02 feedback:

- Decisions 5 and 6 still need John and Egor's input before future partial
  `ParameterConstraints` matching should be treated as final public syntax.
- Decisions 7 and 8 also need John and Egor's input before locking down public
  receiver/object syntax, especially around C++23 explicit object parameters.
- Fully qualified names may become important relatively soon for APIs that use
  common short names across multiple libraries. Inline namespaces are a known
  complication for any future namespace-aware selector.
- Concrete examples of hardcoded C/C++ macro behavior should be collected if the
  future `Language` model becomes a serious mechanism for moving such logic out
  of the compiler.

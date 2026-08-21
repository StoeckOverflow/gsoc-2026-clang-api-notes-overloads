# GSoC 2026 Final Report - Extending Clang API Notes for C++: Overload-Specific Annotations for Functions and Methods

**Contributor:** Dominic Stöcker

**GitHub:** https://github.com/StoeckOverflow

**Organization:** LLVM Compiler Infrastructure

**Role:** GSoC 2026 contributor, LLVM Compiler Infrastructure

**Project:**  Extending Clang API Notes for C++: Overload-Specific Annotations for Functions and Methods

**Project size:** This project was originally accepted as a small, 8-week project. As the implementation scope grew, it was expanded to a large, 12-week project.

**Mentors:** [Gábor Horváth](https://github.com/Xazax-hun), [John Hui](https://github.com/j-hui), and [Egor Zhdan](https://github.com/egorzhdan)

## Introduction

During Google Summer of Code 2026, I worked on extending Clang API Notes so that they can identify and annotate individual C++ function and method overloads.

API Notes allow additional information to be attached to declarations without modifying the original source headers. They are particularly useful when importing C and C++ APIs into Swift. Before this project, API Notes primarily identified declarations by name, which was insufficient for overloaded C++ functions.

For example:

```cpp
struct Widget {
  void setValue(int);
  void setValue(double);
};
```

A name-only API Notes entry cannot distinguish `setValue(int)` from `setValue(double)`. The goal of this project was to design and implement a compatible overload-matching mechanism while preserving the behavior of existing API Notes files.

The project covered the public YAML design, Clang’s API Notes model, binary serialization, declaration lookup, Sema integration, type normalization, diagnostics, testing, and support for C++ member-function object qualifiers.

## Project Goals

The main goal was to allow API Notes entries to distinguish C++ function and method overloads using their explicit parameter lists and, where necessary, the qualifiers of the implicit C++ object parameter.

Compatibility was a central requirement. Existing name-only entries may intentionally apply to an entire overload set. The new format therefore had to introduce optional narrowing constraints without changing the meaning of existing API Notes files.

The project also needed a practical policy for matching C++ parameter types. Raw textual comparison is often too strict because declarations can use aliases, nullability annotations, qualifiers, and other source-level type sugar. At the same time, fully canonicalizing every type can remove distinctions that an API Notes author may intentionally want to preserve.

## Deliverables

The submitted work consists of a series of upstream LLVM pull requests extending Clang API Notes with overload-aware C++ function and method matching.

Merged upstream:

- [PR1: Parser and API Notes model support for `Where.Parameters`](https://github.com/llvm/llvm-project/pull/203227)
- [PR2: Binary serialization support for overload parameter selectors](https://github.com/llvm/llvm-project/pull/204147)
- [PR3: Sema integration for overload-aware API Notes lookup](https://github.com/llvm/llvm-project/pull/205307)
- [PR4: Diagnostics and integration coverage](https://github.com/llvm/llvm-project/pull/209408)

Still in review:

- [PR5: Type normalization for selector lookup](https://github.com/llvm/llvm-project/pull/213043)
- [PR6: C++ member-function object qualifier matching](https://github.com/llvm/llvm-project/pull/216148)
- [Follow-up: API Notes warning visibility in system headers](https://github.com/llvm/llvm-project/pull/216272)

Future work:

Template specialization selector support remains future design work and was not implemented during this project.

## Design

The design keeps the existing `Functions`, `Tags`, and `Methods` API Notes hierarchy. The declaration name remains the primary lookup key, while an optional `Where` block narrows the initial candidate set.

Fields inside `Where` identify declarations. Fields such as `SwiftName`, `Availability`, and nullability annotations remain outside `Where` and are applied after a declaration has matched.

```yaml
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

This API Notes file uses the `Where.Parameters` selector to apply separate `SwiftName` annotations to the `int`- and `double`-parameter overloads of `setValue`.

The same `Where.Parameters` selector is also available for C++ global functions:

```yaml
Functions:
  - Name: makeWidget
    Where:
      Parameters:
        - int
    SwiftName: makeWidgetFromInt(_:)

  - Name: makeWidget
    Where:
      Parameters:
        - double
    SwiftName: makeWidgetFromDouble(_:)
```

Name-only entries preserve their current behavior:

```yaml
Tags:
  - Name: Widget
    Methods:
      - Name: setValue
        Availability: nonswift
```

This entry still applies to every method named `setValue` in `Widget`. Omitting `Where.Parameters` leaves the parameter list unconstrained. Adding `Where.Parameters` narrows the entry to a specific explicit parameter list.

Omitted `Where` properties act as wildcards, while present properties narrow the candidate set. `Where.Parameters` describes the complete explicit parameter list, so parameter arity and order are significant.

An empty parameter list has an explicit meaning:

```yaml
Methods:
  - Name: build
    Where:
      Parameters: []
    SwiftName: buildWithoutArguments()
```

This selects a method with no explicit source parameters. It is different from omitting `Where.Parameters`, which retains broad name-based matching.

For member functions, the explicit parameter list does not always provide enough information. Methods can have the same name and parameters while differing in qualifiers applied to the implicit object:

```cpp
struct Builder {
  Result build() &;
  Result build() &&;
  Result build() const &;
};
```

To distinguish these declarations, the selector model includes an `Object` constraint:

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

      - Name: build
        Where:
          Parameters: []
          Object:
            Const: true
            Ref: lvalue
        SwiftName: buildFromConstLValue()
```

This directly represents properties of the implicit C++ object rather than encoding the receiver through a special position in the ordinary parameter list. `Object` constraints only apply to non-static C++ methods, because global functions and static methods do not have an implicit object parameter.

## Pull Requests

Implementing overload-aware matching required carrying selector information through the complete API Notes pipeline:

```text
YAML
  -> API Notes YAML compiler builds the in-memory API Notes model
  -> binary writer serializes that model
  -> binary reader loads lookup tables
  -> Sema asks the reader for notes by declaration context and name
  -> API Notes and Sema selector logic filters overload-specific entries
  -> Sema applies the selected annotations to the declaration
```

### PR1: Parser and API Notes model

The first patch introduced the parser and data-model foundation for overload-aware API Notes entries.

It allowed the API Notes model to represent multiple functions or methods with the same name but different parsed `Where.Parameters` selectors. This was necessary because a representation based only on declaration names could not preserve separate entries for individual overloads.

**Testing:** Parser tests verify the new syntax while retaining compatibility with existing API Notes entries.

**Status:** Merged

**Pull request:** https://github.com/llvm/llvm-project/pull/203227

**Branch:** https://github.com/StoeckOverflow/llvm-project/tree/clang-apinotes-parse-cxx-where-parameters

### PR2: Binary serialization

The second patch extended the binary API Notes representation.

Same-name entries with different parameter selectors must remain distinct after serialization and deserialization. This patch added the required storage representation.

It also preserved the difference between an omitted parameter constraint and an explicitly empty parameter list.

**Testing:** Round-trip tests verify that same-name entries with different selectors remain distinct after serialization and deserialization, including the distinction between an omitted parameter constraint and an explicitly empty parameter list.

**Status:** Merged

**Pull request:** https://github.com/llvm/llvm-project/pull/204147

**Branch:** https://github.com/StoeckOverflow/llvm-project/tree/apinotes-where-parameters-binary

### PR3: Lookup and Sema integration

The third patch implemented the first end-to-end overload-matching path.

Clang can form a selector from a declaration’s explicit parameter types, locate the corresponding API Notes entry, and apply its effects through Sema. Name and declaration context remain the primary lookup keys. `Where.Parameters` only narrows the same-name candidate set.

Legacy name-only entries continue to use broad matching. Entries with explicit parameter information use overload-specific lookup.

A failed explicit selector does not silently fall back to an unrelated name-only match. This prevents annotations intended for one overload from accidentally being applied to another declaration with the same name.

PR3 also added the first alias-aware matching behavior. The lookup first tries the declaration’s written or sugared parameter type. If no exact entry is found, it can fall back to an appropriately desugared representation. This allows an underlying-type selector such as `int` to match a declaration written with an alias while still giving the alias-specific selector precedence when both forms exist.

**Testing:** Sema tests verify overload-specific annotation application, legacy name-only behavior, written-type lookup, desugared fallback, alias precedence, and the rule that unmatched explicit selectors do not silently degrade into broad matching.

**Status:** Merged

**Pull request:** https://github.com/llvm/llvm-project/pull/205307

**Branch:** https://github.com/StoeckOverflow/llvm-project/tree/apinotes-where-parameters-sema

### PR4: Diagnostics and integration tests

The fourth patch focuses on diagnostics, edge cases, and broader integration coverage.

Relevant cases include malformed selectors, duplicate entries, selectors with no matching overload, zero-parameter methods, default arguments, static methods, and the interaction between broad and overload-specific entries.

Diagnostics are important because an incorrect selector should not be silently ignored. When Clang can determine the relevant overload set, it should provide useful information about malformed, duplicate, or unmatched selectors.

A small follow-up patch addresses this by making the generic `-Wapinotes` diagnostic visible for system headers. This matters because API Notes are commonly attached to SDK headers, and warnings about invalid API Notes entries should not disappear solely because the referenced declaration comes from a system header.

**Testing:** Integration tests cover malformed selectors, duplicate entries, unmatched selectors, zero-parameter methods, default arguments, static methods, and interactions between broad and overload-specific entries.

**Status:** Merged

**Pull request:** https://github.com/llvm/llvm-project/pull/209408

**Branch:** https://github.com/StoeckOverflow/llvm-project/tree/apinotes-where-parameters-diagnostics

### PR5: Type normalization

The fifth part of the project refines how parameter type spellings in API Notes selectors are normalized before comparison with Clang declarations.

A purely textual comparison is too strict because selectors may differ from Clang’s printed declaration types in whitespace, pointer and reference spacing, template punctuation, nullability annotations, and qualifiers that do not create distinct C++ overloads.

The implementation also normalizes spelling differences that should not create separate overload identities. It collapses insignificant whitespace, normalizes spacing around punctuation such as `*`, `&`, `&&`, template commas, and template closers, strips selector-only nullability including nested nullability in pointer, array, and function-pointer type components, and removes top-level `const` and `volatile` where those qualifiers do not distinguish C++ overloads.

The normalization patch also moves declaration-side selector construction into a small Sema helper, so the production extraction logic can be tested directly. A small prequel patch prepares this by tightening selector qualifier and nullability handling before the broader string-normalization work.

The selector normalizer remains intentionally lexical. It handles supported spelling differences such as `unsigned` and `unsigned int`, but it does not broaden matching into full canonical type equivalence. For example, `char` and `unsigned char` remain distinct.

**Testing:** Tests cover nullability-insensitive matching, spelling normalization, `unsigned` spelling normalization, top-level `const` and `volatile`, direct declaration-side selector extraction, normalized duplicate diagnostics, and negative cases where source-level distinctions should be preserved.

**Status:** In review

**Pull request:** https://github.com/llvm/llvm-project/pull/213043

**Branch:** https://github.com/StoeckOverflow/llvm-project/tree/apinotes-where-parameters-normalization

### PR6: C++ object qualifiers

The sixth patch implements the `Object` selector introduced above. It extends parsing, storage, and declaration matching so that API Notes can distinguish unqualified, `const`, `volatile`, lvalue-qualified, and rvalue-qualified member functions with otherwise identical signatures.

The work also covers invalid uses. Static member functions and non-member functions do not have an implicit C++ object parameter, so applying an `Object` constraint to them should result in a diagnostic.

`Where.Object` can be used by itself or together with `Where.Parameters`. Less-constrained object selectors are applied before more-constrained selectors so that specific notes can refine broader object-qualified entries.

**Testing:** Tests cover `const`, `volatile`, lvalue-ref, rvalue-ref, combined object qualifiers, object-only selectors, parameter-plus-object selectors, malformed object selectors, duplicate object selectors, unmatched object selectors, and static method behavior without implicit object matching.

**Status:** In review

**Pull request:** https://github.com/llvm/llvm-project/pull/216148

**Branch:** https://github.com/StoeckOverflow/llvm-project/tree/apinotes-where-parameters-object-qualifiers

## Future Template Design

The future template design work explored how the overload selector model might eventually extend from concrete functions and methods to C++ templates. Template matching was not implemented in the current patches, but the design work helped identify which parts of the selector model should remain extensible.

Ordinary `Where.Parameters` matching works for concrete function parameter types. Templates add several harder questions. One selector might need to identify a function template or one of its specializations, such as `f<int>`. Another selector might need to match an ordinary function overload whose parameter type contains a class template specialization or dependent template parameter. These cases require more structure than the concrete parameter spelling handled by the current implementation.

For example, dependent types can refer to template parameters directly, through nested dependent names, or as part of a larger type:

```cpp
template <typename T> void f(const T &);
template <typename Container> void f(const typename Container::value_type &);
template <typename Key, typename Value>
void f(const std::pair<Key, Value> &);
```

Relying only on source names such as `T`, `Container`, `Key`, or `Value` would be fragile because template parameter names can differ across redeclarations and library implementations. The design notes therefore explored structural template-parameter identity using depth and index, matching concrete template arguments for specializations such as `f<int>`, and a possible mapping from stable template-parameter identities to readable names so dependent types such as `const T &` can still be written clearly.

One possible direction was to assign stable identities to template parameters in API Notes, using depth and index, and then map those identities to readable local names used in dependent parameter spellings such as `const T &`. This would avoid relying on redeclaration-local source names while still keeping API Notes readable.

The [template design document](https://github.com/StoeckOverflow/gsoc-2026-clang-api-notes-overloads/blob/main/ideas/format_v2_templates.md) contains illustrative future syntax for these ideas. That syntax is only a design sketch and is rejected by the current API Notes implementation.

## Conclusions

The project extends Clang API Notes from broad name-based C++ function and method lookup to overload-aware matching based on explicit parameter types and member-function object qualifiers. The work preserves existing API Notes behavior, carries selector information through parsing and binary serialization, integrates matching with Sema, and defines practical normalization behavior for aliases, nullability, and top-level qualifiers.

The remaining upstream work is to complete review and merge the normalization, object-qualifier, and system-header warning visibility patches. Template specialization selector support remains future work based on the design questions outlined above.

### Challenges

- Preserving compatibility with existing name-only API Notes entries while making overload-specific matching available as an optional narrowing mechanism.
- Carrying overload identity through several compiler layers, including parsing, the in-memory API Notes model, binary serialization, lookup, and Sema application.
- Defining a practical type-matching policy that avoids fragile raw string comparison without erasing useful source-level distinctions through full canonicalization.
- Modeling member-function object qualifiers separately from the explicit parameter list, so the public syntax follows the C++ language model.
- Splitting related parser, serialization, lookup, diagnostics, normalization, and object-qualifier work into reviewable upstream patches.

### What I Learned

- Working across Clang’s YAML parsing, API Notes data structures, serialization, declaration lookup, type representation, and Sema requires keeping data-model and compatibility decisions visible throughout the full pipeline.
- Public syntax and compatibility rules are as important as implementation details. Once a format is used by external projects, omission, exact matching, normalization, fallback, and diagnostics must be defined clearly.
- C++ overload identity involves more than explicit parameters. Aliases, source-level type sugar, top-level qualifiers, nullability, and member-function object qualifiers all affect how overload selection should be modeled.
- Reviewer feedback is most useful when the implementation is split into focused patches that can be discussed, tested, and revised independently.

## Links

**Contributor:**
Dominic Stöcker

**GitHub profile:**
https://github.com/StoeckOverflow

**Development repository:**
https://github.com/StoeckOverflow/llvm-project

**Design document:**
https://github.com/StoeckOverflow/gsoc-2026-clang-api-notes-overloads/blob/main/ideas/format_v2.md

**Development diary:**
https://github.com/StoeckOverflow/gsoc-2026-clang-api-notes-overloads/blob/main/blog/diary.md

**Pull requests:**

- [Parse `Where.Parameters` for C++ functions and methods](https://github.com/llvm/llvm-project/pull/203227)

- [Serialize overload parameter selectors](https://github.com/llvm/llvm-project/pull/204147)

- [Apply overload-aware API Notes in Sema](https://github.com/llvm/llvm-project/pull/205307)

- [Add diagnostics and integration coverage](https://github.com/llvm/llvm-project/pull/209408)

- [Normalize parameter types during selector lookup](https://github.com/llvm/llvm-project/pull/213043)

- [Match C++ member-function object qualifiers](https://github.com/llvm/llvm-project/pull/216148)

- [Show API Notes warnings in system headers](https://github.com/llvm/llvm-project/pull/216272)

**Additional future-design material:**
https://github.com/StoeckOverflow/gsoc-2026-clang-api-notes-overloads/blob/main/ideas/format_v2_templates.md

**Template specialization selector design branch:**
https://github.com/StoeckOverflow/llvm-project/tree/apinotes-where-parameters-template-design

## Acknowledgements

I would like to thank my mentors, Gábor Horváth, John Hui, and Egor Zhdan, for their thoughtful reviews, engaging design discussions, and support throughout Google Summer of Code.

Their feedback helped keep the implementation compatible and reviewable while extending it from basic parameter matching to robust type normalization and C++ object-qualifier support.

# Prototype Notes v1

This document records the first prototype approach. It is kept in `ideas/` because it describes experimental implementation work rather than the stable API Notes knowledge base.

Status: deprecated v1 idea, retained for design history.

## Architecture Summary

The prototype touches the normal API Notes pipeline:

- `Types.h`: shared API Notes data structures.
- `APINotesYAMLCompiler.cpp`: parses YAML and converts it into API Notes model
  objects.
- `APINotesWriter` / `APINotesFormat.h`: binary format constants, table keys,
  block IDs, and layout definitions.
- `APINotesReader`: reads the binary format and performs lookups.
- `APINotesManager`: finds API Notes files for headers, frameworks, and modules.
- `SemaAPINotes.cpp`: applies looked-up notes to Clang declarations during
  semantic analysis.

## Prototype Pipeline

```text
.apinotes YAML
  -> APINotesYAMLCompiler
     classifies method notes as legacy/name-only or typed/signature-specific
  -> APINotesWriter
     serializes tables using improved method keys
  -> Binary API Notes
     stores legacy entries and typed entries
  -> APINotesReader
     looks up notes by method key
  -> SemaAPINotes
     sees a CXXMethodDecl, computes the same key, and asks the reader for a note
  -> Clang AST declaration
     receives SwiftName, availability, nullability, and other annotations
  -> Swift importer / tooling
     sees the precise annotation
```

## Core Problem

Before the prototype, C++ methods were keyed by:

```text
parent C++ tag context + method name
```

For example:

```c++
struct S {
  void frob(int);
  void frob(double);
};
```

Both methods are `S::frob`, so `frob(int)` and `frob(double)` could not be
distinguished.

## Prototype Key

The prototype adds an overload-aware key in `APINotesFormat.h`:

```text
parent C++ tag context
+ method name
+ normalized explicit parameter types
```

Example YAML:

```yaml
- Name: frob
  Parameters:
    - Position: 0
      Type: int
```

This can target `S::frob(int)`.

Implementation sketch:

```c++
enum class CXXMethodKeyKind : uint8_t {
  Typed = 0,
  LegacyNameOnly = 1,
};

struct CXXMethodTableKey {
  uint32_t parentContextID;
  uint32_t nameID;
  CXXMethodKeyKind kind;
  llvm::SmallVector<uint32_t, 2> paramTypeIDs;
};
```

Conceptually:

```text
S::mix       -> legacy fallback
S::mix(int)  -> precise overload note
```

## Type Normalization

Type normalization lives in `Types.cpp`:

```c++
std::string normalizeCXXMethodParamType(llvm::StringRef TypeSpelling) {
  llvm::SmallVector<llvm::StringRef, 8> Parts;
  TypeSpelling.trim().split(Parts, ' ', -1, false);
  return llvm::join(Parts, " ");
}
```

These spellings match:

- `Type: "  int  "`
- `Type: "int"`

These spellings do not match:

- `unsigned int`
- `unsigned`

## YAML Classification

In `APINotesYAMLCompiler.cpp`, a method note becomes typed when:

- typed signatures are enabled for that method name.
- parameters exist.
- at least one explicit parameter position is `>= 0`.
- every explicit position from `0...N` has a type.

The compatibility heuristic is that typed matching only activates when a tag
contains multiple API Notes entries for the same method name.

## Writing

`APINotesWriter.cpp` interns method names and parameter type spellings as
identifiers, then writes either a legacy key or a typed key.

```c++
if (IsLegacyNameOnly)
  return CXXMethodTableKey::legacy(CtxID.Value, NameID);

for (llvm::StringRef ParamType : ParamTypes)
  ParamTypeIDs.push_back(getIdentifier(ParamType));

return CXXMethodTableKey::typed(CtxID.Value, NameID, ParamTypeIDs);
```

Serialized key:

```text
parent context id
method name id
key kind
number of parameter types
parameter type ids...
```

## Reading And Lookup

The reader keeps a compatibility path for the legacy C++ method table:

```c++
class LegacyCXXMethodTableInfo
    : public VersionedTableInfo<..., SingleDeclTableKey, CXXMethodInfo> {
  ...
};
```

Lookup is typed-first, then legacy fallback:

```c++
auto TypedKnown = CXXMethodTable->find(
    CXXMethodTableKey::typed(CtxID.Value, *NameID, ParamTypeIDs));
if (TypedKnown != CXXMethodTable->end())
  return {SwiftVersion, *TypedKnown};

auto LegacyKnown = CXXMethodTable->find(
    CXXMethodTableKey::legacy(CtxID.Value, *NameID));
if (LegacyKnown != CXXMethodTable->end())
  return {SwiftVersion, *LegacyKnown};
```

Example:

```yaml
- Name: mix
  AvailabilityMsg: "legacy mix unavailable"

- Name: mix
  AvailabilityMsg: "int mix unavailable"
  Parameters:
    - Position: 0
      Type: int
```

Behavior:

```text
mix(int)     -> typed note: "int mix unavailable"
mix(double)  -> fallback legacy note: "legacy mix unavailable"
```

## Sema Integration

`SemaAPINotes.cpp` computes the method name and normalized parameter types, then
asks the reader for a C++ method note.

```c++
if (CXXMethod->isOverloadedOperator())
  MethodName = std::string("operator") +
      getOperatorSpelling(CXXMethod->getOverloadedOperator());
else
  MethodName = CXXMethod->getName();

for (const ParmVarDecl *Param : CXXMethod->parameters()) {
  ParamTypeStorage.push_back(
      getNormalizedCXXMethodParamType(Policy, Param->getType()));
  ParamTypes.push_back(ParamTypeStorage.back());
}

auto Info = Reader->lookupCXXMethod(Context->id, MethodName, ParamTypes);
```

Constructors, destructors, and conversion operators are skipped:

```c++
!isa<CXXConstructorDecl>(CXXMethod) &&
!isa<CXXDestructorDecl>(CXXMethod) &&
!isa<CXXConversionDecl>(CXXMethod)
```

## Duplicate Diagnostics

The prototype detects duplicate effective method notes in
`APINotesYAMLCompiler.cpp`. It tracks typed and legacy notes separately.

```c++
SeenLegacyMethodNotes;
SeenTypedMethodNotes;

auto &SeenMethodNotes =
    Signature.IsLegacyNameOnly ? SeenLegacyMethodNotes
                              : SeenTypedMethodNotes;

if (!SeenMethodNotes.insert(DiagnosticKey).second) {
  emitError("duplicate definition of C++ method note ...");
}
```

This is a conflict:

```yaml
- Name: typed
  Parameters:
    - Position: 0
      Type: int

- Name: typed
  Parameters:
    - Position: 0
      Type: int
```

This works:

```yaml
- Name: mixed

- Name: mixed
  Parameters:
    - Position: 0
      Type: int
```

## Reading The YAML Tests

API Notes tests usually have three pieces:

- A `.apinotes` YAML file under `clang/test/APINotes/Inputs/...`.
- A header declaring the API being annotated.
- A `.cpp` or `.m` lit test that imports/includes the header and checks Clang
  output.

## Commits

```text
a921944eee7a Add tests for C++ method overload matching semantics
84827cc55f23 Introduce overload-aware C++ method keys
655e6bce8924 Implement typed-first lookup for C++ method notes
997e49206aa4 Classify C++ method notes as typed or legacy
2e8e8a98997a Diagnose duplicate effective C++ method notes
9c9e05b00f05 Clarify C++ method type normalization boundaries
870d5b5acd92 Clarify legacy C++ method table compatibility path
```

## Testing Commands

Build Clang:

```sh
ninja -C build clang
```

Run focused API Notes tests:

```sh
build/bin/llvm-lit -sv clang/test/APINotes
build/bin/llvm-lit -sv clang/test/ClangScanDeps/modules-include-tree-api-notes.c
```

Build API Notes library/parser/writer code:

```sh
ninja -C build libclangAPINotes.a clang
```

Run broader Clang validation:

```sh
ninja -C build check-clang
```

## Test Results

Focused API Notes testing:

- `ninja -C build clang` succeeded.
- `llvm-lit -sv clang/test/APINotes` succeeded with 42 passed, 1 unsupported,
  and 1 expected failure.
- `clang/test/ClangScanDeps/modules-include-tree-api-notes.c` passed.

Broader validation:

- `check-clang` was run as broader validation.
- It required a build-directory-only workaround for `libIndexStore.so`.
- It completed with 75 failures.
- None of the failures appeared to be API Notes failures.

Observed unrelated failure classes:

- Many failures came from a local GCC installation warning:
  `-Wgcc-install-dir-libstdcxx`.
- Some tests used `-Werror`, so that warning became fatal.
- Some tests expected exact diagnostic counts or output, and the extra warning
  changed them.
- Several CodeGen tests failed because expected LLVM IR used typed GEPs, while
  the compiler emitted byte-wise `i8` GEPs with equivalent offsets.
- `CodeGen/unsigned-trapv.c` failed because `CHECK-NOT: overflow` matched the
  repository URL embedded in `llvm.ident`: `stoeckoverflow-llvm-swift`.
- The `cir-opt` message was informational only; CIR was not built/enabled and
  API Notes tests did not require it.

Conclusion: focused API Notes tests pass; broad `check-clang` failures look
environmental or unrelated to the API Notes work.

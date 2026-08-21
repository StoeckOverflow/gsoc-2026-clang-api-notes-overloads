# Swift API Notes Knowledge Base

This note is the general knowledge base for how Swift uses Clang API Notes. It
should describe the current architecture and stable concepts. Prototype-specific
experiments belong in `ideas/format_v1.md`, and proposed future formats belong
in `ideas/format_v2.md`.

## What API Notes Are

API Notes are sidecar metadata files for C, Objective-C, and C++ APIs. They let
Clang and Swift attach additional information to declarations without editing the
original headers.

API Notes source files are written in YAML and usually use the `.apinotes`
extension.

They are useful when:

- The original headers are owned by a system SDK or third-party library.
- Swift needs better names, nullability, availability, ownership, or import
  behavior than the headers provide directly.
- Metadata should vary by Swift version without changing the underlying C,
  Objective-C, or C++ declarations.

Typical metadata includes:

- Swift names.
- Availability.
- Nullability.
- Ownership conventions.
- Replacement or bridged types.
- Swift privacy/import behavior.
- Bounds-safety notes.

## Who Consumes API Notes

API Notes are parsed by Clang and can be consumed by multiple Clang-based
clients.

Important consumers include:

- Swift's C, Objective-C, and C++ importer.
- Clang semantic analysis.
- Clang static analyzer.
- Other tooling built on top of Clang declarations.

For Swift, API Notes are one of the bridges between C-family declarations and the
Swift-facing imported API surface.

## Mental Model

At a high level, API Notes answer two questions:

1. Which declaration is being described?
2. Which extra metadata should be applied to that declaration?

The current format is organized around declaration categories such as modules,
Objective-C classes and protocols, C++ tags, namespaces, methods, functions,
globals, and Swift-versioned overrides.

```text
Module
  Namespaces
    Tags
      Methods
      Fields
      Tags
  Classes
    Methods
    Properties
  Protocols
    Methods
    Properties
  Tags
    Methods
    Fields
    Tags
  Functions
  Globals
  SwiftVersions
```

In the current Clang API Notes schema, Objective-C methods live under
`Classes` and `Protocols` and are identified by selector-related fields. C++
methods live under `Tags`, which represent C++ classes, structs, enums, and
unions, and those methods are identified by `Name`.

The source file is human-readable YAML, but Clang does not use that YAML directly
for every lookup. It compiles the YAML into a compact binary representation and
then performs declaration lookups against that representation.

## Processing Pipeline

```text
.apinotes YAML
  -> APINotesYAMLCompiler
  -> APINotesWriter
  -> compact binary API Notes buffer
  -> APINotesReader
  -> SemaAPINotes
  -> Clang declaration attributes and type metadata
  -> Swift importer / Clang tooling behavior
```

This means API Notes have two related forms:

- **Source form:** YAML, optimized for authors and SDK tooling.
- **Binary form:** compact lookup tables, optimized for Clang's importer and
  semantic analysis.

## Main Architecture Pieces

### `Types.h`

`Types.h` contains the shared in-memory data structures for API Notes payloads.

Examples include:

- `CommonEntityInfo`.
- `CommonTypeInfo`.
- `VariableInfo`.
- `ParamInfo`.
- `FunctionInfo`.
- `ObjCMethodInfo`.
- `CXXMethodInfo`.
- `TagInfo`.
- `ContextID` and `Context`.

These structures represent the metadata that can be attached to declarations.

### `APINotesYAMLCompiler.cpp`

The YAML compiler parses `.apinotes` source files and lowers them into the API
Notes model.

It is responsible for:

- Defining YAML-facing structs.
- Validating source notes.
- Converting YAML fields into `*Info` payload objects.
- Deciding how declarations are keyed for serialization.
- Reporting source-level diagnostics.

### `APINotesWriter`

The writer serializes API Notes model objects into the compact binary format.

It is responsible for:

- Interning strings and identifiers.
- Assigning context IDs.
- Building declaration lookup tables.
- Emitting LLVM bitstream records.

### `APINotesReader`

The reader loads the binary API Notes representation and exposes lookup APIs.

It is responsible for:

- Reading bitstream blocks.
- Loading on-disk hash tables.
- Looking up notes by declaration identity.
- Handling Swift-versioned information.
- Preserving compatibility with older binary API Notes formats.

### `APINotesManager`

The manager discovers and caches API Notes files.

It is responsible for:

- Finding API Notes next to module maps, frameworks, and headers.
- Compiling YAML source notes into binary buffers.
- Opening those buffers through `APINotesReader`.
- Mapping source locations and modules to the correct loaded notes.

### `SemaAPINotes.cpp`

`SemaAPINotes.cpp` is where API Notes are applied to Clang declarations.

It is responsible for:

- Dispatching by declaration kind.
- Computing lookup keys for declarations.
- Fetching matching API Notes from the reader.
- Materializing notes as attributes, type metadata, availability, naming, and
  other semantic effects.

## Why The Binary Format Exists

The YAML format is good for humans, but it is not ideal for repeated compiler
lookups.

The binary format exists to provide:

- Fast lookup by declaration identity.
- Lower memory overhead.
- Stable cacheable representation.
- Efficient reuse across module builds.
- Versioned compatibility.

The binary representation uses LLVM bitstream blocks and on-disk hash tables.

Conceptually, it contains:

```text
Control block:
  version, module name, options

Identifier block:
  string table for names and type spellings

Context block:
  namespaces, classes, tags, protocols

Entity tables:
  Objective-C classes
  Objective-C methods
  C++ methods
  fields
  globals
  typedefs
  enum constants
```

## Current Lookup Flow

The normal flow is:

1. Clang discovers a relevant `.apinotes` file through `APINotesManager`.
2. `APINotesManager` compiles the YAML into an in-memory binary buffer.
3. `APINotesReader` opens that binary representation.
4. During semantic analysis, Clang calls into `SemaAPINotes` for declarations.
5. `SemaAPINotes` computes the lookup identity for the declaration.
6. `APINotesReader` returns matching metadata, if any.
7. Clang applies that metadata to the declaration.
8. Swift's importer sees the adjusted declaration surface.

In short:

```text
YAML -> binary tables -> declaration lookup -> AST metadata -> Swift import
```

## Resources

- [Clang API Notes documentation](https://clang.llvm.org/docs/APINotes.html)
- [Egors Talk Euro LLVM Talk about Swift/C++ Interop](https://www.youtube.com/watch?v=2g-fQd_OYUU)

# Clang API Notes C++ Overload Support

This repository tracks design notes, meeting notes, diary entries, and proposal
material for a GSoC 2026 project on extending Clang API Notes for C++ overloads.

## Project Summary

This project extends Clang API Notes to support overload-specific annotations for
C++ functions and methods. Previously, API Notes could primarily identify C++
declarations by name, so annotations for overloaded declarations such as
`foo(int)` and `foo(double)` could apply to the whole overload set rather than to
a specific function.

The project introduces overload-aware matching through existing `Name` entries
plus explicit `Where.Parameters` selectors, allowing API Notes to target
individual overloads by parameter type while preserving compatibility with
existing notes. The work also covers global functions, diagnostics, type
normalization, and C++ member-function object qualifiers.

## Motivation

API Notes are sidecar YAML files that Clang reads while building or importing
modules. They attach extra metadata to declarations without modifying the
original headers. This is especially useful for interoperability work, where
upstream C, Objective-C, or C++ headers often cannot be edited directly.

For Swift interoperability, API Notes can describe metadata such as:

- Swift-facing names.
- Availability.
- Nullability.
- Ownership conventions.
- Import behavior.

The core limitation was that C++ notes were effectively matched by their
enclosing context and declaration name. Existing parameter annotation fields are
not an overload-selection syntax, so the implemented direction adds an explicit
`Where.Parameters` selector instead of changing the meaning of existing
`Parameters` entries.

This project makes overload identity more precise while keeping existing
name-based API Notes behavior valid.

## Repository Layout

- `notes/`: current knowledge base for Swift API Notes and Clang architecture.
- `ideas/`: proposed, experimental, or deprecated designs.
- `meetings/`: raw or lightly cleaned meeting notes.
- `blog/`: final report, LLVM blogpost draft, diary entries, and weekly progress
  logs.
- `proposal/`: GSoC proposal source and generated PDF.

Important files:

- `notes/swift_api_notes.md`: general API Notes knowledge base and resource list.
- `ideas/format_v1.md`: deprecated v1 prototype notes.
- `ideas/format_v2.md`: proposed v2 selector-format idea.
- `ideas/format_v2_templates.md`: future C++ template-matching notes for the v2
  selector model.
- `blog/report.md`: final GSoC project report.
- `blog/llvm_blogpost.md`: LLVM blogpost draft.
- `blog/diary.md`: running project diary.
- `meetings/meeting_12052026.md`: first design meeting notes.

## Current Design Questions

The main remaining design topics are:

- How future template support should fit into the same selector model.
- How to represent dependent template parameter types without relying on unstable
  source names.
- How far future type matching should go beyond the implemented normalization
  behavior.

## Status

The repository currently contains project organization notes, meeting notes,
proposal material, design documents, a development diary, the final report, and
an LLVM blogpost draft.

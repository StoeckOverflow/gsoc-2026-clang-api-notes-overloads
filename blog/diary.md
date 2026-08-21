# Diary

## 2026-05-12: First Design Meeting And Project Setup

The first meeting focused on how API Notes should evolve to support C++
overload matching without breaking existing behavior.

This week I also did the initial project setup work:

- Created a Git repository for the project.
- Organized my current notes into a clearer structure.
- Split stable knowledge-base notes from experimental ideas.
- Tidied up my LLVM / Swift fork LLVM workspace so I am ready to start properly.
- Investigated the prototype design for overload-aware C++ method API Notes.

Main takeaways:

- Keep a clear separation between current architecture notes and future ideas.
- Treat the first overload-aware implementation as `format_v1.md`.
- Explore a cleaner v2 selector model where matching is explicit and structured.
- Preserve compatibility with existing API Notes semantics wherever possible.
- Be careful about how strict matching, omitted qualifiers, diagnostics, and type
  spelling normalization should behave.

## 2026-05-18: Format Draft Preparation

This week was quieter on the visible side, but useful for turning the first
meeting notes into a more coherent design draft.

I focused on shaping the open questions from the initial discussion into a
written proposal: how overload-specific API Notes should be represented in YAML,
which parts of an entry should select declarations, and which parts should
describe the annotations applied to those declarations. I also spent time
separating the first implementation slice from larger future ideas such as
templates, receiver qualifiers, and richer diagnostics.

The main outcome was preparation for the first `format_v2.md` draft. Even though
there was not much external progress to report, this helped make the next design
iteration more concrete and easier to review.

## 2026-05-25: First Format V2 Draft Sent For Review

Based on the first design meeting, I created a first draft of the cleaner
`format_v2.md` proposal and sent it out by mail to the project reviewers.

The draft tried to turn the meeting discussion into a concrete YAML direction:

- Keep the existing `Tags` / `Methods` hierarchy instead of proposing a broad
  top-level API Notes redesign.
- Introduce explicit selector-style syntax for overload-aware method matching.
- Separate declaration-selection constraints from annotation effects.
- Preserve broad name-based matching for existing API Notes entries.
- Document the difference between exact signatures and future partial parameter
  filters.
- Keep receiver/object qualifiers, diagnostics, templates, and language gates as
  explicit design questions instead of silently folding them into the first
  implementation.

The goal of sending the draft by mail was to get agreement on the public shape
before spending too much time on implementation details.

## 2026-06-01: Review Waiting And Codebase Orientation

This week was mostly a transition week after sending out the first format draft.
I waited for reviewer feedback while using the time to get more comfortable with
the API Notes implementation and the surrounding Clang code.

The practical work was about reducing friction before implementation: reading
through the relevant parser, data model, serialization, and lookup paths and
checking how the proposed selector shape could fit into the existing code. I prepared
the first commits so the implementation work can be split into small, reviewable pieces.

## 2026-06-08: Feedback Integration And First Prototype Work

This week was mostly about turning the first `format_v2.md` draft into a sharper
first upstreamable slice.

On 2026-06-02, Gábor sent detailed feedback on the proposed v2 format. The most
important compatibility point was that moving `Name` fully inside `Where` would
force API Notes to support both the old and new placement anyway. Based on that,
I revised the proposed first-wave syntax so method-entry `Name` remains the
primary lookup key, while `Where` contains only residual narrowing constraints
such as `Signature`.

The updated first-wave shape is now:

```yaml
Tags:
  - Name: Widget
    Methods:
      - Name: setValue
        Where:
          Signature:
            Parameters:
              - int
        SwiftName: setIntValue(_:)
```

I also folded Gábor's future-facing feedback into the format notes:

- `Expect` remains future syntax because it can improve diagnostics but may also
  make notes brittle when libraries add new overloads.
- Explicit object parameters should probably fit into the same future `Object`
  selector vocabulary as cv/ref-qualified member functions.
- Template matching should leave room for template parameter structure and
  template arguments instead of relying on raw parameter names.
- Qualified names and inline namespaces need to stay visible as future design
  concerns.
- Language restrictions remain lower priority but may be useful if some
  hardcoded C/C++ macro handling can eventually move into API Notes.

Some design questions remain intentionally open pending John and Egor's input,
especially the exact-signature versus partial-constraint split and the future
receiver/object model. The practical goal is to converge on the first-wave shape
soon enough that the prototype can be split into small LLVM upstream PRs.

I separated the future C++ template-matching sketches into
`format_v2_templates.md`, so `format_v2.md` can stay focused on first-slice
rationale and design tradeoffs.

In parallel, I started local prototype work. The current prototype direction is
to split the implementation into smaller pieces: parser/model support for
`Name + Where.Signature`, binary serialization and round-trip tests, lookup/Sema
support for exact explicit-parameter matching, diagnostics for malformed or
duplicate selectors, and focused behavior tests for edge cases such as empty
parameter lists, default arguments, static methods, and supported type-spelling
normalization.

## 2026-06-15: Design Review Before Implementation - PR1 and PR2

This week's meeting focused on finalizing the first implementation slice and
planning the transition from design work to coding. By this point, the first PR
was in review and close to being merged, so the discussion also helped clarify
how the following PRs should be scoped.

The main direction stayed intentionally narrow: land the exact
parameter-signature matching path first, keep the YAML shape compatible with
existing API Notes behavior, and leave the more expressive selector ideas for
later patches. I also started preparing PR2 after the Monday meeting, with the
goal of keeping the implementation incremental and easy to review.

Main takeaways:

- Keep the first implementation focused on exact parameter-based overload
  matching.
- Defer template matching, receiver/object constraints, richer parameter
  filters, and advanced diagnostics to future work.
- Revisit the selector syntax and explore reducing unnecessary nesting in the
  `Where` structure.
- Continue keeping future extensibility in mind without letting it complicate
  the initial implementation.
- Split the implementation into a series of small, reviewable LLVM PRs.
- Submit PR2 after the Monday meeting once the next slice is ready.

## 2026-06-22: PR2 Follow-up and PR3 Start

This week focused on continuing the second implementation slice and moving the
next one forward. PR2 remained under review, with additional reviewer feedback
being integrated, including the template-related preparation needed for the patch
to fit cleanly into the surrounding work.

After that, I started working on PR3 and pushed the first version for review.
With PR3, we now have a first running end-to-end example for overloads, just in
time for the midterms. The rest of the week was spent integrating reviewer
feedback and preparing the PR for finalization.

## 2026-06-29: PR2 Follow Up and PR3 Push

This week focused on finishing the remaining PR2 follow-up and pushing PR3 into
a more complete shape. The main design discussion was about how much type sugar
should participate in overload selector matching.

Main takeaways:

- Nullability should not be part of selector identity. API Notes can add or
  overwrite nullability separately, so matching should ignore outer nullability.
- Typedefs and aliases should be handled permissively: first try the
  declaration's written spelling, then fall back to the desugared type.
- Matching should stay tied to the declaration site, not the call site.
- Partial matching should become a later parameter-constraints follow-up.
- Add template examples for the case 5 design notes.

I also pushed an update to PR3:

- Sema now tries the written parameter type spelling first and uses a desugared
  fallback only if no exact entry exists.
- Alias selectors therefore win over underlying-type selectors when both are
  available.
- Selector formation strips outer nullability and by-value top-level `const`
  before lookup.
- Added tests for alias fallback, alias-vs-underlying precedence, and
  nullability-insensitive matching for both global functions and C++ methods.

## 2026-07-06: Normalization, Object Qualifiers, And Future Scope

- Planned the remaining implementation sequence after the initial end-to-end
  overload matching work.
- Decided to keep normalization and edge-case behavior in the next focused patch,
  followed by object qualifier support for C++ member functions.
- Kept parameter constraints out of the immediate upstream path. They remain
  useful future work, but the current project should stay focused on exact
  explicit-parameter selectors, practical type normalization, diagnostics, and
  member-function object qualifiers.
- Discussed keeping the template design notes visible as future design material,
  possibly through a small documentation-oriented follow-up that links or folds
  the template-matching discussion into the broader design notes.

## 2026-07-13: PR3 Merge, PR4 Review, And Report Planning

- PR3 was merged, completing the first end-to-end Sema integration for
  `Where.Parameters` overload matching.
- Pushed PR4 for diagnostics and broader integration coverage.
- Confirmed that parameter constraints are no longer part of the immediate
  project scope.
- Planned the remaining implementation work around diagnostics review,
  normalization behavior, and object qualifier matching.
- Started shaping the final report, with PR5 and PR6 kept as placeholders until
  those pull requests are opened.
- Kept future ideas such as templates and partial parameter constraints clearly
  separated from the completed implementation.

## 2026-07-20

- Focused on the remaining diagnostics design and on preparing the final patch
  sequence.
- Discussed how to avoid quadratic behavior when checking for unused or
  unmatched API Notes entries.
- Noted that this matters especially for larger API Notes files, such as
  possible C++ standard library annotations, where different libraries or
  implementations may need separate entries and many entries may not apply to a
  particular header parse.
- Considered a callback-based approach: while parsing the header, Clang can visit
  each `FunctionDecl`, form the API Notes selector, try to match the declaration,
  and record which API Notes entries were used.
- Discussed whether a later callback or finalization step could diagnose entries
  that were never selected, without repeatedly scanning the full declaration
  set.
- Talked about test coverage for the diagnostics behavior.
- Agreed to extend the deadline so the remaining patches can be prepared
  carefully.
- Decided that the immediate next step is finishing the diagnostics PR by
  addressing the complexity issue.
- Resolved the selector-header question: the diagnostics PR can keep its helper
  local, while the later string-normalization PR introduces the reusable selector
  extraction API needed for unit testing.

## 2026-07-27

- PR4 was merged, completing the diagnostics and integration-test patch.
- Opened PR5 and spent most of the week focusing on the normalization work.
- Continued refining the report draft and updated it to reflect that the project
  had grown from the original small, 8-week scope into a large, 12-week
  project.
- Planned to use the following vacation week as a break, so no meeting was
  scheduled.
- Agreed that the next design discussion after the break should focus on the PR6
  object-qualifier patch.

## 2026-08-10

- Discussed the PR6 object-qualifier design and confirmed that the overall
  direction looks good.
- Planned to work on PR5 and PR6 in parallel because the normalization and
  object-qualifier changes are independent enough to progress separately.
- Decided to first get the PR5 prequel merged, then address the review comments
  on PR5 while preparing and pushing PR6.
- Identified the remaining cleanup work after PR5 and PR6: follow-up updates to
  the diagnostics PR and the ASTContext `const` cleanup from the PR5 prequel.
- Planned to push the documentation updates and the template-design PR.
- Planned to fold the latest project changes into the final report and prepare
  the GitHub Gist submission.
- If time allows, the remaining exploratory work is to compare the current
  implementation against the future template design and identify possible pain
  points before template support is implemented.

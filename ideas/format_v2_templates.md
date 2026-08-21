# API Notes Format V2: Future C++ Template Matching

Status: future design notes for C++ template matching. This is not part of the
first upstreamable overload-matching slice.

The first slice stays focused on non-template C++ functions and methods, using
declaration-entry `Name` plus optional `Where.Parameters` for exact explicit
parameter matching. This file only sketches how the same selector model could
leave room for C++ templates later.

## Scope Boundary

Template matching should stay separate from ordinary overload matching. A future
design should keep three concerns distinct:

- `Where.Parameters` describes ordinary explicit function parameters.
- `Template.Parameters` could describe the template parameter list structurally.
- `TemplateArguments` could describe a concrete specialization or instantiation.

In the first upstreamable slice, a template type that appears as an ordinary
parameter type is still just a source spelling inside
`Where.Parameters`. Alias equivalence, default template arguments, and
semantic template equivalence remain out of scope.

## Why Templates Need A Separate Design

Ordinary `Where.Parameters` matching works for concrete function parameter
types. Templates add at least two separate matching problems:

- A selector might need to identify a function template or one of its
  specializations, such as `f<int>`.
- A selector might need to match an ordinary function overload whose parameter
  type contains a class template specialization or dependent template parameter.

The second case can involve dependent types that refer to template parameters
directly, through nested dependent names, or as part of a larger type:

```cpp
template <typename T> void f(const T &);
template <typename Container> void f(const typename Container::value_type &);
template <typename Key, typename Value>
void f(const std::pair<Key, Value> &);
```

These cases require more structure than the concrete parameter spelling handled
by the current implementation. Source names such as `T`, `Container`, `Key`, and
`Value` are not stable enough by themselves because they can differ across
redeclarations and library implementations.

## Template Parameter Structure

Gábor's redeclaration example motivates matching template structure without
relying on template parameter names:

```cpp
template <typename T>
void f(T t);

template <typename P>
void f(P p);

int main() {
  f<int>(5);
}
```

If a future API Notes selector refers to the source name of a template parameter,
renaming `T` to `P` could break notes even though the C++ meaning did not
change. Dependent forms such as `T::value_type` or `T *` make this more
complicated than ordinary spelling normalization.

One future direction is to refer to template parameter uses structurally, such as
by template-parameter depth and index rather than by redeclaration-local names.
Illustrative future syntax:

```yaml
Functions:
  - Name: f
    Where:
      Template:
        Parameters:
          - Kind: type
      Parameters:
        - TemplateParameter:
            Depth: 0
            Index: 0
```

Here, `Template.Parameters` describes the template parameter list, while
`Where.Parameters` describes the ordinary function parameter list. The
`TemplateParameter` spelling is only a placeholder for the future structural
idea.

Another possible direction is to let API Notes assign local names to structural
template-parameter identities, then use those API Notes-local names in ordinary
parameter spellings. This keeps the selector readable without depending on the
template parameter names written in any particular C++ redeclaration:

```yaml
Functions:
  - Name: f
    Where:
      TemplateParameters:
        - Name: T
          Depth: 0
          Index: 0
      Parameters:
        - const T &
```

In this sketch, `T` is not looked up as the source-level template parameter
name. It is introduced by the API Notes entry itself and mapped to the
structural template parameter at depth 0, index 0. The same declaration could
therefore still match if one C++ redeclaration names the parameter `T` and
another names it `P`.

This approach raises additional syntax questions. For example, the design would
need to decide whether the local-name binding belongs under `Where`, under a
future `Template` block, or in a separate selector namespace. It would also need
clear rules for nested templates, parameter packs, non-type template parameters,
and template-template parameters.

## Template Arguments

A concrete specialization or instantiation such as `f<int>` may need a separate
template-argument selector:

```yaml
Functions:
  - Name: f
    Where:
      TemplateArguments:
        - int
      Parameters:
        - int
```

This keeps template argument matching separate from the ordinary function
parameter list. Whether this selector should match explicit specializations,
implicit instantiations, or both is a future semantic question.

Another possible future design is to avoid matching primary templates at all and
match only concrete, monomorphized instantiations. That would avoid depending on
template parameter names in API Notes syntax, but it would also make primary
template annotations a separate unresolved problem.

## Mixed Template And Non-Template Overloads

A likely future case is a name that refers to both a function template and a
non-template overload:

```cpp
template <typename T>
void setValue(T);

void setValue(int);
```

The first slice can already target ordinary non-template overloads by explicit
parameter spelling. Future template support would need an additional declaration
identity layer so an API Notes entry can say whether it targets the primary
template, a concrete specialization, or the non-template overload.

## Open Questions

- matching primary templates versus explicit specializations
- matching explicit template arguments such as `f<int>`
- representing non-type template parameters, template-template parameters, and
  parameter packs
- deciding whether default template arguments participate in matching
- deciding whether aliases or alternative template spellings are equivalent
- referring to template parameter uses in ordinary parameter types without
  depending on redeclaration-local names such as `T` or `P`
- deciding whether primary templates are matched directly or only concrete,
  monomorphized instantiations are matched

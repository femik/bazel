---
layout: documentation
title: Extensions - Overview
---
# Overview


## Loading an extension

Extensions are files with the `.bzl` extension. Use the `load` statement to
import a symbol from an extension.

```python
load("//build_tools/rules:maprule.bzl", "maprule")
```

This code will load the file `build_tools/rules/maprule.bzl` and add the
`maprule` symbol to the environment. This can be used to load new rules,
functions or constants (e.g. a string, a list, etc.). Multiple symbols can be
imported by using additional arguments to the call to `load`. Arguments must
be string literals (no variable) and `load` statements must appear at
top-level, i.e. they cannot be in a function body.

`load` also supports aliases, i.e. you can assign different names to the
imported symbols.

```python
load("//build_tools/rules:maprule.bzl", maprule_alias = "maprule")
```

You can define multiple aliases within one `load` statement. Moreover, the
argument list can contain both aliases and regular symbol names. The following
example is perfectly legal (please note when to use quotation marks).

```python
load(":my_rules.bzl", "some_rule", nice_alias = "some_other_rule")
```

In a `.bzl` file, symbols starting with `_` are private and cannot be loaded
from another file. Visibility doesn't affect loading (yet): you don't need to
use `exports_files` to make a `.bzl` file visible.

## Macros and rules

A [macro](macros.md) is a function that instantiates rules. It is useful when a
BUILD file is getting too repetitive or too complex, as it allows you to reuse
some code. The function is evaluated as soon as the BUILD file is read. After
the evaluation of the BUILD file, Bazel has little information about macros: if
your macro generates a `genrule`, Bazel will behave as if you wrote the
`genrule`. As a result, `bazel query` will only list the generated `genrule`.

A [rule](rules.md) is more powerful than a macro. It can access Bazel internals
and have full control over what is going on. It may for example pass information
to other rules.

If you want to reuse simple logic, start with a macro. If a macro becomes
complex, it is often a good idea to make it a rule. Support for a new language
is typically done with a rule. Rules are for advanced users: we expect that most
people will never have to write one, they will only load and call existing
rules.

## Evaluation model

A build consists of three phases.

* **Loading phase**. First, we load and evaluate all extensions and all BUILD
  files that are needed for the build. The execution of the BUILD files simply
  instantiates rules (each time a rule is called, it gets added to a graph).
  This is where macros are evaluated.

* **Analysis phase**. The code of the rules is executed (their `implementation`
  function), and actions are instantiated. An action describes how to generate
  a set of outputs from a set of inputs, e.g. "run gcc on hello.c and get
  hello.o". It is important to note that we have to list explicitly which
  files will be generated before executing the actual commands. In other words,
  the analysis phase takes the graph generated by the loading phase and
  generates an action graph.

* **Execution phase**. Actions are executed, when at least one of their outputs is
  required. If a file is missing or if a command fails to generate one output,
  the build fails. Tests are also run during this phase.

Bazel uses parallelism to read, parse and evaluate the `.bzl` files and `BUILD`
files. A file is read at most once per build and the result of the evaluation is
cached and reused. A file is evaluated only once all its dependencies (`load()`
statements) have been resolved. By design, loading a `.bzl` file has no visible
side-effect, it only defines values and functions.

Bazel tries to be clever: it uses dependency analysis to know which files must
be loaded, which rules must be analyzed, and which actions must be executed. For
example, if a rule generates actions that we don't need for the current build,
they will not be executed.


## Upcoming changes

The following items are upcoming changes.

*   Comprehensions currently "leak" the values of their loop variables into the
    surrounding scope (Python 2 semantics). This will be changed so that
    comprehension variables are local (Python 3 semantics).

*   Previously dictionaries were guaranteed to use sorted order for their keys.
    Going forward, there is no guarantee on order besides that it is
    deterministic. As an implementation matter, some kinds of dictionaries may
    continue to use sorted order while others may use insertion order.

*   The `+=` operator and similar operators are currently syntactic sugar; `x +=
    y` is the same as `x = x + y`. This will change to follow Python semantics,
    so that for mutable collection datatypes, `x += y` will be a mutation to the
    value of `x` rather than a rebinding of the variable `x` itself to a new
    value. E.g. for lists, `x += y` will be the same as `x.extend(y)`.

*   The "set" datatype is being renamed to "depset" in order to avoid confusion
    with Python's sets, which behave very differently.

*   The `+` operator is defined for dictionaries, returning an immutable
    concatenated dictionary created from the entries of the original
    dictionaries. This will be going away. The same result can be achieved using
    `dict(a.items() + b.items())`.

*   The `|` operator is defined for depsets as a synonym for `+`. This will be
    going away; use `+` instead.

*   The structure of the set that you get back from using the `+` or `|`
    operator is changing. Previously `a + b`, where `a` is a set, would include
    as its direct items all of `a`'s direct items. Under the upcoming way, the
    result will only include `a` as a single transitive entity. This will alter
    the visible iteration order of the returned set. Most notably, `set([1,
    2]) + set([3, 4] + set([5, 6])` will return elements in the order `1 2 3 4 5
    6` instead of `3 4 5 6 1 2`. This change is associated with a fix that
    improves set union to be O(1) time.

These changes concern the `load()` syntax in particular.

* Currently a `load()` statement can appear anywhere in a file so long as it is
  at the top-level (not in an indented block of code). In the future they will
  be required to appear at the beginning of the file, i.e., before any
  non-`load()` statement.

* In BUILD files, `load()` can overwrite an existing variable with the loaded
  symbol. This will be disallowed in order to improve consistency with .bzl
  files. Use load aliases to avoid name clashes.

* The .bzl file can be specified as either a path or a label. In the future only
  the label form will be allowed.

* Cross-package visibility restrictions do not yet apply to loaded .bzl files.
  At some point this will change. In order to load a .bzl from another package
  it will need to be exported, such as by using an `exports_files` declaration.
  The exact syntax has not yet been decided.


## Profiling the code

To profile your code and analyze the performance, use the `--profile` flag:

```shell
$ bazel build --nobuild --profile=/tmp/prof //path/to:target
$ bazel analyze-profile /tmp/prof --html --html_details
```

Then, open the generated HTML file (`/tmp/prof.html` in the example).


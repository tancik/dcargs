# Goals and alternatives

## Design goals

The core functionality of `dcargs` — generating argument parsers from type
annotations — overlaps significantly with features offered by other libraries.
Usage distinctions are the result of two API goals:

- **One uninvasive function.** For all core functionality, learning to use
  `dcargs` should reduce to learning to write (type-annotated) Python. For
  example, types are specified using standard annotations, helptext using
  docstrings, choices using the standard `typing.Literal` type, subcommands with
  `typing.Union` of nested types, and positional arguments with `/`.
  - In contrast, similar libraries have more expansive APIs , and require more
    library-specific structures, decorators, or metadata formats for configuring
    parsing behavior.
- **Strict typing.** Any type that can be annotated and unambiguously parsed
  with an `argparse`-style CLI interface should work out-of-the-box; any public
  API that isn't statically analyzable should be avoided.
  - In contrast, many similar libraries implement features that depend on
    dynamic argparse-style namespaces, or string-based accessors that can't be
    statically checked.

More concretely, we can also compare specific features. A noncomprehensive set:

|                                              | Dataclasses | Functions | Literals             | Docstrings as helptext | Nested structures | Unions over primitives | Unions over nested types  | Lists, tuples        | Dictionaries | Generics |
| -------------------------------------------- | ----------- | --------- | -------------------- | ---------------------- | ----------------- | ---------------------- | ------------------------- | -------------------- | ------------ | -------- |
| [argparse-dataclass][argparse-dataclass]     | ✓           |           |                      |                        |                   |                        |                           |                      |              |          |
| [argparse-dataclasses][argparse-dataclasses] | ✓           |           |                      |                        |                   |                        |                           |                      |              |          |
| [datargs][datargs]                           | ✓           |           | ✓[^datargs_literals] |                        |                   |                        | ✓[^datargs_unions_nested] | ✓                    |              |          |
| [tap][tap]                                   |             |           | ✓                    | ✓                      |                   | ✓                      | ~[^tap_unions_nested]     | ✓                    |              |          |
| [simple-parsing][simple-parsing]             | ✓           |           | ✓[^simp_literals]    | ✓                      | ✓                 | ✓                      | ✓[^simp_unions_nested]    | ✓                    | ✓            |          |
| [dataclass-cli][dataclass-cli]               | ✓           |           |                      |                        |                   |                        |                           |                      |              |          |
| [clout][clout]                               | ✓           |           |                      |                        | ✓                 |                        |                           |                      |              |          |
| [hf_argparser][hf_argparser]                 | ✓           |           |                      |                        |                   |                        |                           | ✓                    | ✓            |          |
| [typer][typer]                               |             | ✓         |                      |                        |                   |                        | ~[^typer_unions_nested]   | ~[^typer_containers] |              |          |
| [pyrallis][pyrallis]                         | ✓           |           |                      | ✓                      | ✓                 |                        |                           | ✓                    |              |          |
| [yahp][yahp]                                 | ✓           |           |                      | ~[^yahp_docstrings]    | ✓                 | ✓                      | ~[^yahp_unions_nested]    | ✓                    |              |          |
| [omegaconf][omegaconf]                       | ✓           |           |                      |                        | ✓                 |                        |                           | ✓                    | ✓            |          |
| **dcargs**                                   | ✓           | ✓         | ✓                    | ✓                      | ✓                 | ✓                      | ✓                         | ✓                    | ✓            | ✓        |

<!-- prettier-ignore-start -->

[datargs]: https://github.com/roee30/datargs
[tap]: https://github.com/swansonk14/typed-argument-parser
[simple-parsing]: https://github.com/lebrice/SimpleParsing
[argparse-dataclass]: https://pypi.org/project/argparse-dataclass/
[argparse-dataclasses]: https://pypi.org/project/argparse-dataclasses/
[dataclass-cli]: https://github.com/malte-soe/dataclass-cli
[clout]: https://pypi.org/project/clout/
[hf_argparser]: https://github.com/huggingface/transformers/blob/master/src/transformers/hf_argparser.py
[pyrallis]: https://github.com/eladrich/pyrallis/
[typer]: https://typer.tiangolo.com/
[yahp]: https://github.com/mosaicml/yahp
[omegaconf]: https://omegaconf.readthedocs.io/en/2.1_branch/structured_config.html

[^datargs_unions_nested]: One allowed per class.
[^tap_unions_nested]: Not supported, but API exists for creating subcommands that accomplish a similar goal.
[^simp_unions_nested]: One allowed per class.
[^yahp_unions_nested]: Not supported, but similar functionality available via ["registries"](https://docs.mosaicml.com/projects/yahp/en/stable/examples/registry.html).
[^typer_unions_nested]: Not supported, but API exists for creating subcommands that accomplish a similar goal.
[^simp_literals]: Not supported for mixed (eg `Literal[5, "five"]`) or in container (eg `List[Literal[1, 2]]`) types.
[^datargs_literals]: Not supported for mixed types (eg `Literal[5, "five"]`).
[^typer_containers]: `typer` uses positional arguments for all required fields, which means that only one variable-length argument (such as `List[int]`) without a default is supported per argument parser.
[^yahp_docstrings]: Via the `hp.auto()` function, which can parse docstrings from external classes. Usage is different from the more direct parsing that `dcargs`, `tap`, and `simple-parsing`/`pyrallis` support.

<!-- prettier-ignore-end -->

Note that most of these other libraries are generally aimed specifically at only
one of dataclasses (`datargs`, `simple-parsing`, `argparse-dataclass`,
`argparse-dataclasses`, `dataclass-cli`, `clout`, `hf_argparser`, `pyrallis`,
`yahp`), custom structures (`tap`), or functions (`typer`) rather than general
typed callables, but offer other features that you might find useful, such as
registration for custom types (`pyrallis`), built-in approaches for
serialization and config files (`tap`, `pyrallis`, `yahp`) and, opportunities
for integration with fields generated using standard argparse definitions
(`simple-parsing`).

## Note on configuration systems

Our API is designed to be used to configure general applications and
computational experiments written in Python, but intentionally tries to avoid
building a full configuration framework (for example, `hydra`). These frameworks
can typically be broken into three components with varying levels of
integration, which include syntax and logic for **(1)** defining configurable
fields, **(2)** saving and choosing between a set of base configurations, and
**(3)** overriding configurable values at the command-line.

`dcargs` is meant to take advantage of modern Python features for **(1)** and
focus completely on **(3)** in a way that's agnostic to a project's preferred
approach for **(2)**. **(2)** is left as an exercise to the user; it tends to be
the most open-ended and varied in terms of project and personal preferences, but
is typically straightforward to implement given **(1)**.

In contrast, popular libraries oriented toward configuration often strive to be
more "batteries-included", which is convenient but requires prescribing things
like specific formats, processes, or directory structures for working with saved
configurations. This requires more moving parts, which can be limiting if any
one of them is insufficient for a particular use case. (or if you just don't
like working with YAML formats)

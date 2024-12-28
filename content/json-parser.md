+++
title = "Coding Challenges Solutions: JSON Parser"
date = 2024-12-28

[taxonomies]
categories = ["Guides"]
tags = ["Coding Challenges", "TDD"]
+++

Here I show how I solved the second of the codding challenges from [Coding Challenges](https://codingchallenges.fyi), which is about building a JSON parser from scratch. Please read the challenge description to learn more about what we will be solving here: https://codingchallenges.fyi/challenges/challenge-json-parser

<!-- more -->

## Step Zero

This step is just about the setup of the project. I have decided to use [Python](https://www.python.org/) for this tutorial (version `3.11` to be precise), and the [Poetry packaging and dependency manager](https://python-poetry.org/). Please go through poetry's documentation and start a new project; something like `poetry new pyccjp`, you should endup with a project structure like:

```
.
└── pyccjp
    ├── pyccjp
    ├── tests
    ├── README.md
    └── pyproject.toml
```

I called the project is called `pyccjp` because it is a json parser written in Python following the Coding Challenges guide. Since we are building the solultion from scratch, we will only need development dependencies, so go to the root of the project and do `poetry add --group dev mypy ruff pytest`. Before commiting any changes we should be able to run `poetry run ruff --fix .`, `poetry run ruff format .`, `poetry run mypy .` and `poetry run pytest tests/unit` without getting any error messages.

## Step one

In this step we are required to write a command line program that returns `0` when given a valid JSON file and `1` otherwise; moreover, we are defining a JSON file as valid if it has `{}` as content and invalid otherwise. Basically, we need to write some boilerplate and basic logic for this. Let's create a `pyccjp/main.py` file with the following content:


```python
import pathlib

def main(file: pathlib.Path) -> int:
    raise NotImplementedError


def _cli() -> None:
    import argparse
    import sys

    parser = argparse.ArgumentParser()
    parser.add_argument("file", type=pathlib.Path)

    args = parser.parse_args()

    code = main(args.file)
    sys.exit(code)


if __name__ == "__main__":
    _cli()

```

The `_cli` function doesn't have any logic, just boilerplate: it is one of the simplest ways to handle command line arguments in Python and returning status codes. I don't like to write test cases for such simple code, but I isolate it as much as possible. What we care about here is what we will later write inside the `main` function and we will develop the logic needed using TDD. Let's start with the piece of logic that handle invalid JSON files, create `tests/test_main.py` file and add the following content:

```python
import pathlib

import pytest

from pyccjp import main


@pytest.mark.parametrize("payload", [""])
def test_1_for_invalid_json(payload: str, tmp_path: pathlib.Path):
    filepath = tmp_path / "invalid.json"
    filepath.write_text(payload)

    code = main.main(filepath)

    assert code == 1

```

If you go to your terminal and run `poetry run pytest tests/ -vv` you will get one error message `FAILED`, which is what we want for now. In case you are unfamiliar with how the test was written, I strongly suggest you to go through my previous post about implementing the wc linux tool from scratch. We now need to write the implementation, but we now know that whatever we write must pass the test. One possible implementation is the following:

```python
def main(file: pathlib.Path) -> int:
    return 1
```

You may be surprised to see such implementation, but with TDD we try to do the minimal thing to make tests pass every time. Let me add an additional requirement that is not from the Coding Callenges side now: we are also required to return a status code of `2` if the given file doesn't exist. Let's start with the test again and add the following to `tests/test_main.py`:

```python
def test_2_for_file_doesnot_exist(tmp_path: pathlib.Path):
    filepath = tmp_path / "doesnot_exist.json"

    code = main.main(filepath)

    assert code == 2
```

That will help us reinforce the idea of adding minimial functionality every time, here is one possible solution:

```python
def main(file: pathlib.Path) -> int:
    if not file.exists():
        print("File doesn't exist")
        return 2
    return 1
```

Let's now move forward to the required valid JSON example at this step, by adding the following test to `tests/test_main.py`:

```python
@pytest.mark.parametrize("payload", [r"{}"])
def test_0_for_valid_json(payload: str, tmp_path: pathlib.Path):
    filepath = tmp_path / "valid.json"
    filepath.write_text(payload)

    code = main.main(filepath)

    assert code == 0
```

Again, we are back at having a failing test, but we are getting into the interesting work now. The simplest change to `main` would turn it into:

```python
def main(file: pathlib.Path) -> int:
    if not file.exists():
        print("File doesn't exist")
        return 2
    content = file.read_text()
    if content != r"{}":
        print("file has invalid content")
        return 1

    return 0
```

As far as TDD is concerned, we are good to go to the next step. Before we go, however, let's do some clean up. The Coding Challenges guide asks us to write a simple lexer and parser for this step. If you are unfamiliar with these two concepts, do not worry, the idea is getting familiar through the development of the challenges. It turns out that there are two very important steps to write a parsing tool, which even compilers use: Lexical Analysis and Syntactical Analysis. After you read about them on your own, you probably will get to the conclussion that lexical analysis is about processing relevant characters into understandable tokens and syntactical analysis about putting those tokens together in a way that is understandable by the language (JSON in this case). So let's refactor our code to look like it.

create `pyccjp/core.py` to be:

```python
import enum

class JsonSyntax(enum.Enum):
    LEFT_BRACE = 0
    RIGHT_BRACE = 1

type Token = JsonSyntax


class InvalidJson(Exception): ...

```

`tests/test_lexer.py` to be:

```python
import pytest

from pyccjp import lexer, core

def test_yields_braces():
    tokens = list(lexer.lex(r"{}"))
    assert tokens == [core.JsonSyntax.LEFT_BRACE, core.JsonSyntax.RIGHT_BRACE]


@pytest.mark.parametrize("payload", ["", "  "])
def test_empty_yields_nothing(payload: str):
    tokens = list(lexer.lex(payload))
    assert tokens == []
```

`pyccjp/lexer.py` to be

```python
from collections.abc import Generator

from .core import Token, JsonSyntax


def lex(payload: str) -> Generator[Token]:
    for c in payload:
        if c == "{":
            yield JsonSyntax.LEFT_BRACE
        elif c == "}":
            yield JsonSyntax.RIGHT_BRACE
```

`tests/test_parser.py` to be

```python
import pytest

from pyccjp import core, parser


def test_InvalidJson_when_no_tokens():
    with pytest.raises(core.InvalidJson):
        parser.parse([])


def test_parses_empty_object():
    instance = parser.parse([core.JsonSyntax.LEFT_BRACE, core.JsonSyntax.RIGHT_BRACE])
    assert instance == {}
```

`pyccjp/parser.py` to be 

```python
from typing import Any, Iterable

from .core import Token, InvalidJson, JsonSyntax


def parse(tokens: Iterable[Token]) -> dict[str, Any]:
    for t in tokens:
        if t is JsonSyntax.LEFT_BRACE:
            return _parse_object(tokens)
    raise InvalidJson

def _parse_object(tokens: Iterable[Token]) -> dict[str, Any]:
    for t in tokens:
        if t is JsonSyntax.RIGHT_BRACE:
            return {}

```

And `pyccjp/main.py` to have

```python
# [...]

from . import lexer, parser, core


def main(file: pathlib.Path) -> int:
    if not file.exists():
        print("File doesn't exist")
        return 2
    content = file.read_text()
    try:
        parser.parse(lexer.lex(content))
    except core.InvalidJson:
        print("file has invalid content")
        return 1

    return 0

# [...]
```

I used the comment `[...]` to highlight that there is more code in the file, but you should be able to guess what it is. The new implementation seems a bit more complicated, but we will see later how it helps us to have more testeable and easier to understand code. Finally, add the following to `pyproject.toml` so that you can run the CLI tool using real files found in your laptop (lke the ones suggested for testing each step by the Coding Challenges guide):

```toml
[tool.poetry.scripts]
pyccjp = "pyccjp.main:_cli"
```

To call the tool you can do `poetry run pyccjp [PATH_TO_FILE]`.

## Step two

Now we will start to evolve our tool to support the full JSON file format. Here we are required to handle string keys and string values, something like `{"key": "value"}` for example. Let's add the given test cases to our `tests/test_main.py` file, it should look like:

```python
# [...]

@pytest.mark.parametrize("payload", ["", '{"key": "value",}', '{"key": "value", key2: "value"}'])
def test_1_for_invalid_json(payload: str, tmp_path: pathlib.Path):
    filepath = tmp_path / "invalid.json"
    filepath.write_text(payload)

    code = main.main(filepath)

    assert code == 1

# [...]

@pytest.mark.parametrize("payload", ["{}", '{"key": "value"}', '{"key": "value","key2": "value"}'])
def test_0_for_valid_json(payload: str, tmp_path: pathlib.Path):
    filepath = tmp_path / "valid.json"
    filepath.write_text(payload)

    code = main.main(filepath)

    assert code == 0
```

After executing the tests, you will find that our current implementation passes the valid tests cases but doesn't pass the new invalid tests cases. We need to fix this, but we will do it by modifying the `lex` and `parse` functions, leaving `main.py` intact. Let's start by modifing `parse`, to account for the new set of syntactic elements we need to handle:

```python
# [...]
from typing import Annotated


class JsonSyntax(enum.Enum):
    LEFT_BRACE = 0
    RIGHT_BRACE = 1
    COLON = 2
    COMMA = 3

type Token = JsonSyntax | Annotated[str, "used for keys or string values only"]
# [...]
```
In other words, we now need to support `:`, `,` and string keys and values for objects. With that, we can add the new tests to the `parse` function:

```python
# [...]

def test_ValueError_when_tailing_comma_before_right_brace():
    iterator = iter(
        [
            JsonSyntax.LEFT_BRACE,
            "key",
            JsonSyntax.COLON,
            "value",
            JsonSyntax.COMMA,
            JsonSyntax.RIGHT_BRACE,
        ]
    )
    with pytest.raises(ValueError):
        parse(iterator)


def test_parses_size_1_object_with_string_values():
    iterator = iter(
        [
            JsonSyntax.LEFT_BRACE,
            "key",
            JsonSyntax.COLON,
            "value",
            JsonSyntax.RIGHT_BRACE,
        ]
    )

    assert parse(iterator) == {"key": "value"}


def test_parses_size_2_object_with_string_values():
    iterator = iter(
        [
            JsonSyntax.LEFT_BRACE,
            "key",
            JsonSyntax.COLON,
            "value",
            JsonSyntax.COMMA,
            "key2",
            JsonSyntax.COLON,
            "value2",
            JsonSyntax.RIGHT_BRACE,
        ]
    )

    assert parse(iterator) == {"key": "value", "key2": "value2"}

# [...]
```

To fix our implementation accordingly, we need to upgrade `pyccjp/parser.py` as follows:

```python
def parse(tokens: Iterator[Token]) -> dict[str, Any] | str:
    for t in tokens:
        if t is JsonSyntax.LEFT_BRACE:
            return _parse_object(tokens)
        if isinstance(t, str):
            return t
    raise ValueError


def _parse_object(tokens: Iterator[Token]) -> dict[str, Any]:
    result: dict[str, Any] = {}
    while True:
        key = next(tokens)
        if key is JsonSyntax.RIGHT_BRACE:
            break
        if key is JsonSyntax.COMMA:
            key = next(tokens)
        if not isinstance(key, str):
            raise ValueError

        if next(tokens) is not JsonSyntax.COLON:
            raise ValueError

        value = parse(tokens)
        result[key] = value

    return result
```

You may think of other implementations and that is perfectly fine. The main goal here is to show the process of TDD, not to go deep into implementation details. Note that we didn't need of the new invalid JSON cases for this tests, as we simply don't conceive a quote on its own: it is not one of the *documented* options for the `Token` type. However, we still have the tests for the `main` function to account for this, making sure everything works when those functions are used together.  

We now need to check whether our `lex` function can handle the new type of tokens required, which are `:`, `,` and `"` (and the value they quote). We therefore add the following to `tests/test_lex.py`:

```python
# [...]

from pyccjp.core import JsonSyntax

# [...]

def test_object_with_string_values():
    tokens = lexer.lex('{"key": "value","key2": "value2"}')
    assert list(tokens) == [
        JsonSyntax.LEFT_BRACE,
        "key",
        JsonSyntax.COLON,
        "value",
        JsonSyntax.COMMA,
        "key2",
        JsonSyntax.COLON,
        "value2",
        JsonSyntax.RIGHT_BRACE,
    ]

# [...]
```

And correspondigly update out imlementation of `lex` to:

```python
# [...]

def lex(payload: str) -> Iterator[Token]:
    iterator = iter(payload)
    for c in iterator:
        if c == "{":
            yield JsonSyntax.LEFT_BRACE
        elif c == "}":
            yield JsonSyntax.RIGHT_BRACE
        elif c == ":":
            yield JsonSyntax.COLON
        elif c == ",":
            yield JsonSyntax.COMMA
        elif c == '"':
            yield _lex_string(iterator)


def _lex_string(payload: Iterator[str]) -> str:
    content = ""
    for c in payload:
        if c == '"':
            break
        content += c
    return content
```

Now every test case from step 1 and step 2 added to `tests/test_main.py` works!

## Step Three
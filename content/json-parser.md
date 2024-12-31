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

This step is just about the setup of the project. I have decided to use [Python](https://www.python.org/) for this tutorial (version `3.12` to be precise), and the [Poetry packaging and dependency manager](https://python-poetry.org/). Please go through poetry's documentation and start a new project; running `poetry new pyccjp`, will take you to a project structure like:

```
.
└── pyccjp
    ├── pyccjp
    ├── tests
    ├── README.md
    └── pyproject.toml
```

I called the project `pyccjp` because it is a json parser written in Python following the Coding Challenges guide. Since we are building the solultion from scratch, we will only need development dependencies, so go to the root of the project and do `poetry add --group dev mypy ruff pytest`. Before commiting any changes we should be able to run `poetry run ruff --fix .`, `poetry run ruff format .`, `poetry run mypy .` and `poetry run pytest tests/ -vv` without getting any error messages.

## Step One

In this step we are required to write a command line program that returns `0` when given a valid JSON file and `1` otherwise; moreover, we are defining a JSON file as valid if it has `{}` as content and invalid otherwise. Let's create a `pyccjp/main.py` file with the following content:


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

The `_cli` function doesn't have any meaningful logic, just boilerplate: it is one of the simplest ways to handle command line arguments in Python and returning status codes. I don't like to write test cases for such a simple piece of code, but I like to isolate it as much as possible. What we care about here is what we will later write inside the `main` function and we will develop the logic needed using Test Driven Development (TDD).

Let's start with the piece of logic that handle invalid JSON files, create `tests/test_main.py` file and add the following content:

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

If you go to your terminal and run `poetry run pytest tests/ -vv` you will get one error message, which is what we expect for now. In case you are unfamiliar with how the test was written, I strongly suggest you to go through [my previous post about implementing the wc linux tool from scratch](https://srcolinas.github.io/word-count/). We now need to write the implementation, but whatever we write must pass that test. One possible implementation is the following:

```python
def main(file: pathlib.Path) -> int:
    return 1
```

You may be surprised to see such implementation, but with TDD we try to do the minimal thing to make tests pass every time. Let me add an additional requirement that is not from the Coding Callenges side: return a status code of `2` if the given file doesn't exist. Let's start with the test again and add the following to `tests/test_main.py`:

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

Again, we are back at having a failing test and we want to make it work with the following implementation for `main`:

```python
from collections.abc import Iterator

from . import lexer, parser


def main(file: pathlib.Path) -> int:
    if not file.exists():
        print("File doesn't exist")
        return 2
    
    try:
        parser.parse(lexer.lex(_character_iterator(file)))
    except ValueError:
        print("File has invalid content")
        return 1

    return 0

def _character_iterator(file: pathlib.Path) -> Iterator[str]:
    with open(file, "r") as f:
        for line in f:
            for c in line:
                yield c
# [...]
```

Before you feel like I added too much out of nowhere, let me explain. The Coding Challenges guide asks us to write a simple lexer and parser for this step. If you are unfamiliar with these two concepts, do not worry, the idea is getting familiar through the development of the challenges. It turns out that there are two very important steps to write a parsing tool, which even compilers use: Lexical Analysis and Syntactical Analysis. You can read about them on your own, but, roughly speaking, the first is about processing relevant characters into understandable tokens and the second is about putting those tokens together in a way that is understandable (Python's dictionaries and lists with strings, numbers, booleans and nulls, possibly with nested dictionaries and lists).

I still need to walk you through the implementation of `parse` and `lex`. Let's define the interface needed for parsing (save it in `pyccjp/parser.py`):

```python
import enum
from typing import Any, Iterator


class JsonSyntax(enum.Enum):
    LEFT_BRACE = 0
    RIGHT_BRACE = 1

type Token = JsonSyntax

def parse(tokens: Iterator[Token]) -> dict[str, Any]: ...
```

Now we need some tests to make sure it works as expected at this step. 

```python
import pytest

from pyccjp.parser import JsonSyntax, Token, parse

@pytest.mark.parametrize("tokens", [[], [JsonSyntax.RIGHT_BRACE]])
def test_ValueError_for_invalid_input(tokens: list[Token]):
    tokens = iter(tokens)
    with pytest.raises(ValueError):
        parse(tokens)


def test_parses_empty_object():
    tokens = iter([JsonSyntax.LEFT_BRACE, JsonSyntax.RIGHT_BRACE])
    assert parse(tokens) == {}
```

It is trivial to write a function that passes the tests once one is familiar with iterators, exceptions and enums in Python, so let's add fix our `parse` function:

```python
def parse(
    tokens: Iterator[Token],
) -> dict[str, Any] | list[Any]:
    try:
        lead = next(tokens)
    except StopIteration:
        raise ValueError
    if lead is not JsonSyntax.LEFT_BRACE:
        raise ValueError(f"incorrect leading token")
    
    tail = next(tokens)
    if tail is not JsonSyntax.RIGHT_BRACE:
        raise ValueError(f"incorrect finish token")
    return {}

```

Now, we need to make sure we can process characters as they come by, for which the `lex` function is responsible. The tests we need to make for it at this step are (write it in `tests/test_lex.py`):

```python
import pytest

from pyccjp.lexer import lex, JsonSyntax, Token


@pytest.mark.parametrize(
    "payload,expected",
    [
        ("", []),
        ("  ", []),
        ("\n", []),
        ("{", [JsonSyntax.LEFT_BRACE]),
        ("}", [JsonSyntax.RIGHT_BRACE]),
        ("{}", [JsonSyntax.LEFT_BRACE, JsonSyntax.RIGHT_BRACE]),
        ("{}\n", [JsonSyntax.LEFT_BRACE, JsonSyntax.RIGHT_BRACE]),
        ("\n{}", [JsonSyntax.LEFT_BRACE, JsonSyntax.RIGHT_BRACE]),
        ("{\n}", [JsonSyntax.LEFT_BRACE, JsonSyntax.RIGHT_BRACE]),
    ],
)
def test_handling_of_braces_and_empty_strings(payload: str, expected: list[Token]):
    tokens = lex(iter(payload))
    assert list(tokens) == expected
```

Note that I added a few whitespaces here and there to make the tests more complete. To be able to pass those tests, I came up with the following implementation for `pyccjp/lexer.py`:

```python
from collections.abc import Iterator

from .parser import Token, JsonSyntax


def lex(payload: Iterator[str]) -> Iterator[Token]:
    for c in payload:
        if c == "{":
            yield JsonSyntax.LEFT_BRACE
        elif c == "}":
            yield JsonSyntax.RIGHT_BRACE
```

I used the comment `[...]` to highlight that there is more code in the file, but you should be able to guess what it is. The new implementation seems a bit more complicated, but it helps us to have more testeable and easier to understand code. Finally, add the following to `pyproject.toml` so that you can run the CLI tool using real files found in your laptop (lke the ones suggested for testing each step by the Coding Challenges guide):

```toml
[tool.poetry.scripts]
pyccjp = "pyccjp.main:_cli"
```

To call the tool you can do `poetry run pyccjp [PATH_TO_FILE]`.

## Steps Two and Three

To avoid being repetitive, I decided to merge the steps two and three into one for this post. They are about handling scalar types in JSON, that is, we need to handle objects with string, numeric, booleans and null types. The Coding Challenges guide suggests to run a specific set of tests at every step, so we will add them to `test/test_main.py` but I will not show it here; instead, I will focus on the tests needed for the `lex` and `parse` functions, otherwise this blog becomes more long and boring than needed. 

Let's start by modifing `pyccjp/parser.py`, to account for the new set of syntactic elements we need to handle:

```python
# [...]
class JsonSyntax(enum.Enum):
    LEFT_BRACE = 0
    RIGHT_BRACE = 1
    COLON = 2
    COMMA = 3

type Token = JsonSyntax | str | bool | int | float | None
# [...]
```
Let's add tests to `tests/test_parser.py` so that we check our `parse` function:
1. Can handle scalar types:
2. Raises `ValueError` when new possible tokens are given in wrong order.
3. Handles object with multiple key value pairs.

```python
@pytest.mark.parametrize(
    "tokens",
    [
        [],
        [JsonSyntax.RIGHT_BRACE],
        [JsonSyntax.RIGHT_BRACE, JsonSyntax.LEFT_BRACE],
        [JsonSyntax.LEFT_BRACE, JsonSyntax.COLON],
        [JsonSyntax.LEFT_BRACE, JsonSyntax.COMMA],
        [JsonSyntax.LEFT_BRACE, JsonSyntax.RIGHT_BRACE, JsonSyntax.COMMA],
        [JsonSyntax.LEFT_BRACE, "key", JsonSyntax.RIGHT_BRACE],
    ],
)
def test_ValueError_for_invalid_input(tokens: list[Token]):
    tokens = iter(tokens)
    with pytest.raises(ValueError):
        parse(tokens)


@pytest.mark.parametrize("value", [True, False, 3, 3.14, "pi", None])
def test_parses_object_with_scalar_value_types(value: bool | int | float | str | None):
    iterator = iter(
        [
            JsonSyntax.LEFT_BRACE,
            "key",
            JsonSyntax.COLON,
            value,
            JsonSyntax.RIGHT_BRACE,
        ]
    )

    assert parse(iterator) == {"key": value}

def test_parses_object_with_multiple_scalar_values():
    iterator = iter(
        [
            JsonSyntax.LEFT_BRACE, 
            "true", JsonSyntax.COLON, True, JsonSyntax.COMMA,
            "false", JsonSyntax.COLON, False, JsonSyntax.COMMA,
            "null", JsonSyntax.COLON, None, JsonSyntax.COMMA, 
            "float", JsonSyntax.COLON, 3.14, JsonSyntax.COMMA,
            "int", JsonSyntax.COLON, 3, JsonSyntax.COMMA, 
            "string", JsonSyntax.COLON, "pi",
            JsonSyntax.RIGHT_BRACE,
        ]
    )

    assert parse(iterator) == {
        "true": True, "false": False, "null": None,
        "float": 3.14, "int": 3, "string": "pi",
    }
```

I came up with the following solution:

```python
# [...]
def parse(tokens: Iterator[Token]) -> dict[str, Any]:
    try:
        lead = next(tokens)
    except StopIteration:
        raise ValueError
    if lead is not JsonSyntax.LEFT_BRACE:
        raise ValueError(f"incorrect leading token")
    
    result = _parse_object(tokens)
    _consume_until_end(tokens)
    return result

def _consume_until_end(tokens: Iterator[Token]) -> None:
    i = -1
    for i, t in enumerate(tokens):
        if i > 0:
            raise ValueError(f"incorrect finish token")
        if t is JsonSyntax.RIGHT_BRACE:
            break
    else:
        if i > -1:
            raise ValueError(f"incorrect finish token")
    for t in tokens:
        raise ValueError(f"incorrect finish token")
            

def _parse_object(tokens: Iterator[Token]) -> dict[str, Any]:
    result: dict[str, Any] = {}
    for key in tokens:
        if key is JsonSyntax.RIGHT_BRACE:
            break
        if key is JsonSyntax.COMMA:
            try:
                key = next(tokens)
            except StopIteration:
                raise ValueError
            

        if not isinstance(key, str):
            raise ValueError(f"key {key} should be a string")

        if next(tokens) is not JsonSyntax.COLON:
            raise ValueError

        value = next(tokens)
        if not isinstance(value, (str, int, bool, float)) and value is not None:
            raise ValueError(f"invalid value {value}")
        result[key] = value

    return result
```

You may think of other implementations that work and that is perfectly fine. The main goal here is to show the process of TDD, not to go deep into implementation details. I will leave the updates on `tests/test_lexer.py` and `pyccjp/lexer.py` up to you, but you can check my solution later. 

## Steps Four and Five

Here we need to add support for (possibly nested) arrays and objects. So now, we update the `JsonSyntax` type to look as follows:

```python
class JsonSyntax(enum.Enum):
    LEFT_BRACE = 0
    RIGHT_BRACE = 1
    COLON = 2
    COMMA = 3
    LEFT_BRACKET = 4
    RIGHT_BRACKET = 5
```

Let's add test cases to make sure we can handle arrays with scalar types, similar to what we did for objects in the previous step:

```python
@pytest.mark.parametrize("value", [True, False, 3, 3.14, "pi", None])
def test_parses_array_with_scalar_value_types(value: bool | int | float | str | None):
    iterator = iter(
        [
            JsonSyntax.LEFT_BRACKET,
            value,
            JsonSyntax.RIGHT_BRACKET,
        ]
    )

    assert parse(iterator) == [value]

def test_parses_array_with_multiple_scalar_values():
    iterator = iter(
        [
            JsonSyntax.LEFT_BRACKET, True, JsonSyntax.COMMA,
            False, JsonSyntax.COMMA, None, JsonSyntax.COMMA,
            3.14, JsonSyntax.COMMA, 3, JsonSyntax.COMMA, 
            "pi", JsonSyntax.RIGHT_BRACKET,
        ]
    )

    assert parse(iterator) == [True, False, None, 3.14, 3, "pi"]
```

Note that we can add more cases to raise `ValueError` as we did previously, but I just decided not to show it here. My solution updates the `parse` function as follows:

```python
def parse(
    tokens: Iterator[Token],
) -> dict[str, Any] | list[Any]:
    try:
        lead = next(tokens)
    except StopIteration:
        raise ValueError
    if lead is JsonSyntax.LEFT_BRACE:
        result = _parse_object(tokens)
    elif lead is JsonSyntax.LEFT_BRACKET:
        result = _parse_array(tokens)
    else:
        raise ValueError(f"invalid leading token {lead}")
    
    _consume_until_end(tokens)
    return result

# [...]

def _parse_array(tokens: Iterator[Token]) -> list[Any]:
    result: list[Any] = []
    for t in tokens:
        if t is JsonSyntax.RIGHT_BRACKET:
            break
        if t is JsonSyntax.COMMA:
            continue
        value = t
        result.append(value)
    return result
```

Now comes the trickiest piece. We need to support arrays and objects that can be nested, here are some test cases I thought about to do this:

```python
def test_parses_object_with_empty_object_value():
    iterator = iter(
        [
            JsonSyntax.LEFT_BRACE, "key", JsonSyntax.COLON,
            JsonSyntax.LEFT_BRACE, JsonSyntax.RIGHT_BRACE, 
            JsonSyntax.RIGHT_BRACE,
        ]
    )

    assert parse(iterator) == {"key": {}}


def test_parses_object_with_empty_array_value():
    iterator = iter(
        [
            JsonSyntax.LEFT_BRACE, "key", JsonSyntax.COLON,
            JsonSyntax.LEFT_BRACKET, JsonSyntax.RIGHT_BRACKET,
            JsonSyntax.RIGHT_BRACE,
        ]
    )

    assert parse(iterator) == {"key": []}


def test_parses_object_in_list():
    iterator = iter(
        [
            JsonSyntax.LEFT_BRACKET, JsonSyntax.LEFT_BRACE,
            JsonSyntax.RIGHT_BRACE, JsonSyntax.RIGHT_BRACKET,
        ]
    )

    assert parse(iterator) == [{}]

```

Again, I will leave the implementation as an excercise. Just to avoid a longer and more booring post than necessary. Likewise, the rest of the steps are also left as an excercise. You can see the final solution in `https://github.com/srcolinas/codingchallenges_solutions/tree/main/json_parser/pyccjp`


## Food for Thought

It is possible to come up with a solution that doesn't make use of iterators, we can instead make it with a list and the code may look simpler. However, I wanted to challenge myself making an implementation that only reads each character once and doesn't load the file into memory at once. I am sorry if that brought you unnecessary dificulties for you. 

---

[Suggest Edits](https://github.com/srcolinas/srcolinas.github.io/tree/master/content/json-parser.md)
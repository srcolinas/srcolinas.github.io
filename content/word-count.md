+++
title = "Coding Challenges Solutions: Word Count"
date = 2023-12-16

[taxonomies]
categories = ["Guides"]
tags = ["Coding Challenges", "TDD"]
+++


Here I share how I solved the first of the coding challenges from [Coding Challenges](https://codingchallenges.fyi), please read it yourself first: https://codingchallenges.fyi/challenges/challenge-wc.

<!-- more -->

## Step Zero

The challenge gives us a lot of freedom in how we do things: language used, build process and editor. I have decided to use [Python](https://www.python.org/) for this tutorial (version `3.11` to be precise), and the [Poetry packaging and dependency manager](https://python-poetry.org/). Please go through poetry's documentation and start a new project; something like `poetry new pyccwc`, you should endup with a project structure like

```
.
└── pyccwc
    ├── pyccwc
    ├── tests
    ├── README.md
    └── pyproject.toml
```

I decided to call it `pyccwc` because it is the `wc` tool written in Python and following the Coding Challenges guide, but the name is not really the most important thing in my opinion. Now that you have created the project, add some simple development dependencies, that will help you quickly diagnose the quality of your code, `cd` into the top level `pyccwc` directory and do `poetry add --group dev mypy ruff pytest`. Before commiting code changes, I usually do `poetry run ruff --fix .`, `poetry run ruff format .`, `poetry run mypy .` and `poetry run pytest tests/unit`.

Finally, add the following to `pyproject.toml`

```toml
[tool.poetry.scripts]
pyccwc = "pyccwc.main:_cli"
```

This should allow you to use the tool as `pyccwc {FILE}` from the command line (after we implement a `_cli` function in a `pyccwc/main.py` module).

## Step One

*Note: I misread step 1 with step 2 from the original Coding Challenges post, so I implemented original step 2 here, please note that it doesn't make much of a difference.*

We are initially required to write code to output the number of lines of a file. Here is a first attempt (goes into the file `pyccwc/main.py`):

```python
from pathlib import Path

def _cli():
    import argparse

    parser = argparse.ArgumentParser()
    parser.add_argument("file", type=Path)
    parser.add_argument("-l", "--lines", action="store_true")

    args = parser.parse_args()

    if args.lines:
        num_lines = len(args.file.read_text().splitlines())
        print(f"{num_lines} {args.file}")

if __name__ == "__main__":
    _cli()
```

What you can see from `def _cli` to the end of the file is mostly boilerplate: besides parsing user input in the standard Python way, the code that doesn't contain a lot of meaningful logic to solve our problem. Please go through the [`argparse` module documentation](https://docs.python.org/3/library/argparse.html) if you are feeling lost so far.

The logic is implemented after the `if` statement. The code works, do the following to see it working for yourself:

1. Install your application in the current environment: `poetry install`
2. Start a shell that can execute your application: `poetry shell`
3. Execute the application: `pyccwc -c test.txt`

Note that `test.txt` is referenced in the original Coding Challenges link. Step 3 wil print `7145 test.txt` to the console, which is what we want. However, we do not want to test things manually. We want a quicker and more reliable approach to test this.

What would we need to write an automated test for this? You would need to have an environment in which you have installed `pyccwc`, then automate the process of calling a shell command and capture `STDOUT` to see whether it matches the expected message. You would also have to write and store several files around that will have the necessary contents so that you know your application works well, regardless of the contents of the file (for example, in this case we need to remember that `test.txt` contained exactly 7145 lines). I challenge you to try to do it yourself, I bet it will take you some time to have the setup right. There is something more we can do about it though: let's use TDD to design our solution, so that we automatically end up with something that is easier to test.

If you look beyond the boilerplate, you could say we need a function that takes in a file and produces a string. The fact that we take user input in certain form (command line arguments) and produce user output in another (print values to the terminal) has nothing to do with the logic of our application, but more with how our application is deployed (a command line application in this case). If we are to follow TDD we could say we need a function called `process_file` that takes in a `Path` to a file with certain contents and so far is required to produce a `str` with the number of lines if a flag is provided, something like:

```python
from pathlib import Path

def process_file(file: Path, *, count_lines: bool) -> str:
    ...
```

Automatic tests for this are easier to build, here is one possitiblity (goes into `pyccwc/tests/unit/test_main.py`):

```python
from pathlib import Path

from pyccwc.main import process_file

def test_number_of_lines_is_correct():
    file = Path("test.txt")
    assert process_file(file, count_lines=True) == "7145 test.txt"

```

So far it looks simple simple enough, but think about this:

1. You need to remember the number of lines that `test.txt` has.
2. When we add more functionality to the solution, you will need to remember also how many bytes, words and characters the file has.
3. We most certainly need to cover some edge cases from time to time, so we would have to add more files thus we would need to remember the expected responses for all of those files.

My current approach to build software doesn't only requires me to write tests, it also requires me to write them in a way that is simple to maintain, so let's see how we can remove the maintainability issues I just mentioned:


```python
from pathlib import Path

import pytest

from pyccwc.main import process_file

def test_number_of_lines_is_correct(files_and_results: tuple[Path, str]):
    for file, result in files_and_results:
        assert process_file(file, count_lines=True) == result


@pytest.fixture
def files_and_results(tmp_path: Path):
    values = []

    file = tmp_path / "test1.txt"
    file.write_text("A \n piece \n of text.")
    values.append((file, f"3 {file}"))
    return values
```

If you are unfamiliar with the above, please go through [`pytest` documentation](https://docs.pytest.org/en/7.4.x/), read about the [`tmp_path` fixture](https://docs.pytest.org/en/7.1.x/how-to/tmp_path.html) in particular. With this new approach, we do not need to remember anything about the files and we do not need to keep the files around in our codebase, so we are a bit better than before.  

We are not done though. Now I will tell you two more things that I still don't like about this setup for tests:
1. Writing and reading some files is quite an expensive operation, at least compared to processing data in memory; therefore, running our tests this way can be slow (sure, a few short files may not make a difference and you could run tests in parallel, but there is a simpler and cheaper way).
2. We are actually doing two things inside `process_file`: computing the number of lines and then formating the result (for simple applicationgs it may not be too bad, but we are hoping to build a more scalable habit). 

Here is another version of `pyccwc/main.py`:

```python
from pathlib import Path


def count(content: str, *, count_lines: bool) -> int:
    result = -1
    if count_lines:
        result = len(content.splitlines())
    return result


def format(file: Path, count: int) -> str:
    return f"{count} {file}"


def _cli():
    import argparse

    parser = argparse.ArgumentParser()
    parser.add_argument("file", type=Path)
    parser.add_argument("-l", "--lines", action="store_true")

    args = parser.parse_args()
    print(
        format(
            args.file,
            count(args.file.read_text(), count_lines=args.lines)
        )
    )


if __name__ == "__main__":
    _cli()

```

with `tests/unit/test_main.py` being:

```python
from pathlib import Path

import pytest

from pyccwc.main import count, format


@pytest.mark.parametrize(
    "content,num_lines",
    [("A \n piece \n of text.", 3), ("another piece \n", 1), ("", 0)],
)
def test_number_of_lines(content, num_lines):
    assert count(content, count_lines=True) == num_lines


def test_formating():
    assert format(Path("name.txt"), 3) == "3 name.txt"
```

We work more productively now:
* We don't need to keep track of some files for testing.
* Our tests run as fast as they can, even if we don't have multiple cores.
* We can test different aspects of our code independently. 

By the way, look into `pytest.mark.parametrize`, it is a nice way to define multiple test when just the input and output is changing. We are ready to go to the next step.

## Step Two

*Note: I misread step 1 with step 2 from the original Coding Challenges post, so I implemented orignial step 1 here, please note that it doesn't make much of a difference.*

In this step we will add suport to count the number of bytes in a file. We want to make sure we count bytes and not characters, as there are characters that can use more than one byte. Here is how a test could look like (add to `pyccwc/tests/unit/test_main.py`):


```python
@pytest.mark.parametrize(
    "content,num_bytes",
    [("aa\x01\x02b桜", 3), ("", 0)],
)
def test_number_of_bytes(content, num_bytes):
    assert count(content, count_bytes=True) == num_bytes
```

We could figure out a way to combine it with the `test_number_of_lines` we wrote previously, but I prefer to have the test isolated because it would let me focus on the different options one by one and allow me to fit the code more nicely in the post. We haven't wrote the functionality yet, so running `pytest` will lead to 2 errors. We need to change `pyccwc.main.count` so that:
1. It accepts the `count_bytes` parameter.
2. It returns one or possibly two integers instead of one.

Here is a minimal change to the interface of `pyccwc.main.count` that can support what we want:

```python
class Counts(NamedTuple):
    bytes: int
    lines: int


def count(
    content: str, *, count_lines: bool = False, count_bytes: bool = False
) -> Counts:
    num_lines = -1
    num_bytes = -1
    if count_lines:
        num_lines = len(content.splitlines())
    return Counts(num_lines, num_bytes)
```

Note that we now return an object that contains both the number of lines and the number of bytes (I have arbitraryly chosen `-1` to represent the fact that different counts are optional), so tests have to be changed so that they check for this tuple instead of a single number. 


```python
@pytest.mark.parametrize(
    "content,num_lines",
    [("A \n piece \n of text.", 3), ("another piece \n", 1), ("", 0)],
)
def test_number_of_lines(content, num_lines):
    assert count(content, count_lines=True) == (num_lines, -1)

@pytest.mark.parametrize(
    "content,num_bytes",
    [("aa\x01\x02b桜", 8), ("", 0)],
)
def test_number_of_bytes(content, num_bytes):
    assert count(content, count_bytes=True) == (-1, num_bytes)
```

Moreover, we also need to account for the fact that `count_bytes` and `count_lines` can be both `True`, so we need a test case for that

```python
## Note that we do not need to be very exhaustive with
## the following test, edge cases are to be covered in the
## previous ones.
@pytest.mark.parametrize(
    "content,result",
    [("Dear programmer\n please write tests", (2, 34)), ("", (0, 0))],
)
def test_combined_calls(content, result):
    assert count(content, count_lines=True, count_bytes=True) == result
```

Similarly, our formating function and its test need to adapt to this new interface:

```python
@pytest.mark.parametrize(
    "counts,result_numbers", [
        (Counts(-1, 3), "3"),
        (Counts(5, -1), "5"),
        (Counts(4, 2), "4 2"),
    ]
)
def test_formating(counts, result_numbers):
    assert format(Path("name.txt"), counts) == f"{result_numbers} name.txt"
```

```python
def format(file: Path, counts: Counts) -> str:
    result = ""
    for c in counts:
        if c != -1:
            result += f"{c} "
    result += f"{file}"
    return result
```

After all these changes we are still failing to compute the number of bytes correctly, which can be easily fixed like this:

```python
def count(
    content: str, *, count_lines: bool = False, count_bytes: bool = False
) -> Counts:
    num_bytes = 0 if count_bytes else -1
    i = -1
    for i, line in enumerate(content.splitlines()):
        if count_bytes:
            num_bytes += len(line.encode())
    
    num_lines = -1 if not count_lines else i + 1
    return Counts(num_lines, num_bytes)
```

with this new additions, we just need to change our boilerplate (`pyccwc.main._cli`) so that it calls the nicely typed and tested functions properly:

```python
def _cli():
    import argparse

    parser = argparse.ArgumentParser()
    parser.add_argument("file", type=Path)
    parser.add_argument("-l", "--lines", action="store_true")
    parser.add_argument("-c", "--bytes", action="store_true")

    args = parser.parse_args()
    print(
        format(
            args.file,
            count(
                args.file.read_text(), count_lines=args.lines, count_bytes=args.bytes
            ),
        )
    )


if __name__ == "__main__":
    _cli()
```

## Steps Three, Four, Five and Final Step

Adding command line options `-w` (count words) and `-m` (count characters), is very similar to the previous step: write the tests for the `pyccwc.main.count` and `pyccwc.main.format` functions and then make the necessary changes to them so that tests pass. Therefore, I won't walk you through it. You will later find that we didn't really need to change `pyccwc.main.format` to work with the new tests, it just worked out of the box because `Counts` is a `NamedTuple`. Adding the default option (step five) is just another conditional and test that you can add yourself.


For the final step, we want to read from standard input if no file is specified, so there are a few key changes we want to do here:

1. Parse the file argument with `parser.add_argument("file", nargs="?", type=argparse.FileType("r"), default=sys.stdin)`.
2. Modify `count` so that it takes as `typing.TextIO` instead of `str`.

You can see the final solution in `https://github.com/srcolinas/coding-challenges-solutions-python`. Please notice that we may not have covered all of the edge cases in the tests, we would just add some once discovered. Feel free to point it out in the repo.

## Food for Thought

I wrote the code so that it is easy to debug, change and, very importantly, easy to understand by the reader; however, I didn't write it so that performance of the function calls is optimized as much as possible. In particular, there are a few things to notice:
1. Our current solution is $O(n \times m \times l)$ (for number of lines, characters and whitespaces respectively). It could be made so that it is $O(n)$, whit $n$ being the number of characters.
2. We throw a bunch of conditionals inside of the main `for` loop, which could probably avoided by defining some functions to call depending on the flags given; whether that is faster than the current solution is out of the scope of this post.

Finally, the solution doesn't really match the output of `wc` for the number of characters, I just didn't want to spend more time on that; if you want to fix it, you can deep dive into locales, following the Coding Challenges guide on step three.

---

[Suggest Edits](https://github.com/srcolinas/srcolinas.github.io/tree/master/_posts/2023-12-09-word-count.md)

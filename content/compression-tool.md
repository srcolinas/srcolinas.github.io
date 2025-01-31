+++
title = "Coding Challenges Solutions: Compression Tool"
date = 2024-12-28

[taxonomies]
categories = ["Guides"]
tags = ["Coding Challenges", "TDD"]
+++

Here I show how I solved the third of the codding challenges from [Coding Challenges](https://codingchallenges.fyi), which is about building a JSON parser from scratch using Test Driven Development. Please read the challenge description to learn more about what we will be solving here: https://codingchallenges.fyi/challenges/challenge-huffman .

If you just want to see the end code, you can jump to the final section of the post, but that is not the goal of this post :) . The goal is to show the process of development using Test Driven Development. 

For this post, I will skip a lot of boilerplate and focus on some critical aspects of this challenge, so make sure you review my previous two posts if you feel lost:
* [Word Count](https://srcolinas.github.io/word-count/)
* [JSON Parser](https://srcolinas.github.io/json-parser/)

<!-- more -->

## Step Two

Yes, you read that right. I am skipping the walkthrough through steps one and two from the guide. They are very straight forward and non-special for this challenge, specialy because I assume you have already read my posts referenced above. 

In this step we build a [Huffman Tree](https://opendsa-server.cs.vt.edu/ODSA/Books/CS3/html/Huffman.html#building-huffman-coding-trees) from character frequencies (i.e. how many times a character is repeated). We should be able to keep track of the weight of the tree and traverse it, so we should have a class similar to:

```python
import dataclasses
from typing import Hashable


@dataclasses.dataclass(kw_only=True)
class HuffmanTree:
    weight: int
    children: tuple["HuffmanTree", "HuffmanTree"] | None = None

    @classmethod
    def from_frequenceies(
        cls, frequencies: dict[Hashable, int]
    ) -> "HuffmanTree":
        raise NotImplementedError
```

Let's go step by step into implementing this class. The first thing that comes to mind is that, if there is a single character that is repeated any number of times, then it should have the number of times it is repeated as weight and `children` should be `None`; in other words, it should satisfy the following tests:

```python
from src.huffman_tree import HuffmanTree


def test_build_with_single_element_has_no_children():
    tree = HuffmanTree.from_frequenceies({"a": 3})
    assert tree.children is None


def test_build_with_single_element_has_right_weight():
    tree = HuffmanTree.from_frequenceies({"a": 3})
    assert tree.weight == 3
```

One possible implementation is

```python
[...]
@dataclasses.dataclass(kw_only=True)
class HuffmanTree:
    weight: int
    children: tuple["HuffmanTree", "HuffmanTree"] | None = None

    @classmethod
    def from_frequenceies(
        cls, frequencies: dict[Hashable, int]
    ) -> "HuffmanTree":
        _, value = frequencies.popitem()
        return cls(weight=value)
```

It is true, it only supports one element, so let's add support for at least two elements; in other words, let's add support for the following test cases:

```python
def test_build_from_two_elements_has_children():
    tree = HuffmanTree.from_frequenceies({"a": 3, "b": 4})
    assert isinstance(tree.left, HuffmanTree)
    assert isinstance(tree.right, HuffmanTree)


def test_build_from_two_elements_has_right_weight():
    tree = HuffmanTree.from_frequenceies({"a": 3, "b": 4})
    assert tree.weight == 7
```

If you run the tests now, they should fail. We will solve this by first building a list of simple trees (i.e. trees that don't have children) and then merging them together, so we change the implementation of `HuffmanTree.from_frequencies` to:

```python
[...]
class HuffmanTree:
    [...]

    @classmethod
    def from_frequenceies(
        cls, frequencies: dict[Hashable, int]
    ) -> "HuffmanTree":
        trees : list[HuffmanTree] = []
        for v in frequencies.values():
            trees.append(cls(weight=v))

        while len(trees) > 1:
            left = trees.pop()
            right = trees.pop()
            new = cls(weight=left.weight + right.weight, children=(left, right))
            trees.append(new)
        return trees.pop()
```

Now, if we have three or more elements, the implementation becomes more interesting: we should make sure that the children were added in the correct order and all nodes have the right weight:

```python
def test_build_from_three_elements():
    tree = HuffmanTree.from_frequenceies({"z": 2, "k": 7, "m": 24})
    subtree, first = cast(tuple[HuffmanTree, HuffmanTree], tree.children)
    second, thrid = cast(tuple[HuffmanTree, HuffmanTree], subtree.children)

    assert tree.weight == 33
    assert tree.key is None
    assert subtree.key is None
    assert subtree.weight == 9
    assert subtree.children is not None
    assert first.key == "m"
    assert first.children is None
    assert first.weight == 24
    assert second.key == "z"
    assert second.children is None
    assert second.weight == 2
    assert thrid.key == "k"
    assert thrid.children is None
    assert thrid.weight == 7

```

Here is my implementation that works with all tests so far:

```python
import dataclasses
import heapq
from collections.abc import Hashable


@dataclasses.dataclass(kw_only=True)
class HuffmanTree:
    weight: int
    key: Hashable = None
    children: tuple["HuffmanTree", "HuffmanTree"] | None = None

    def __lt__(self, other: "HuffmanTree") -> bool:
        return self.weight < other.weight

    @classmethod
    def from_frequenceies(
        cls, frequencies: dict[Hashable, int]
    ) -> "HuffmanTree":
        trees: list[HuffmanTree] = []
        for k, v in frequencies.items():
            heapq.heappush(trees, cls(weight=v, key=k))

        while len(trees) > 1:
            left = heapq.heappop(trees)
            right = heapq.heappop(trees)
            heapq.heappush(
                trees,
                cls(weight=left.weight + right.weight, children=(left, right)),
            )
        return trees.pop()
```


You may be wondering about the fact that we are not storing the characters anywhere in the tree, only the weights. It is because we don't need that for now, remember that we add bits of complexity only when it is needed.

## Step three

Here we are expected to create a table to help us know how each character maps to a code derived from a Huffman tree.

I added a `key` attribute to our `HuffmanTree` class, so that we can store the characters, but I won't show that step here, it is a relatively straight forward step from the implementation above. This should let us focus on the implementation that correctly goes through the tree and retrieves the code for each character, wich should have the following interface:

```python
def create_prefix_code_table(tree: HuffmanTree) -> dict[Hashable, str]:
    raise NotImplementedErrpr
```

I won't go step by step this time, maybe it is time for you to try to and create the simplest tests and implemenetations on your own. If you feel stuck, here is one possible test:

```python
def test_handles_three_elements():
    tree = HuffmanTree(
        weight=33,
        children=(
            HuffmanTree(
                weight=9,
                children=(
                    HuffmanTree(weight=2, key="z"),
                    HuffmanTree(weight=7, key="k"),
                ),
            ),
            HuffmanTree(weight=24, key="m"),
        ),
    )
    table = create_prefix_code_table(tree)
    assert table == {"z": "00", "k": "01", "m": "1"}
```

In real life, we never know whether our tests are covering every single edge case out there, so whatever I write and whatever you write may have ways to break. Usually that is the way it works: whenever we (or a user :( ) discover new ways to break our system, we should update the test cases to reflect it and fix it.

Since the implementation is not the goal of the post, I will omit it for now.

## Step four

We are required to write the header of the output file. If you are not very much familiar with what programmers put into files and how, you may feel a bit lost. Basically, you can define a file format in any way you like, you define how it looks like internally and its extension (if any). The famous formats out there just happen to solve a common problem so nicely that people use them and they became standard. In this case, we will create one format that works for our purpose and that we don't really expect anyone else to use it, after all, this is an academic excercise, the world of compression is much more complex nowadays.

What is most important for our file format is that it contains the necessary information to decode a compressed file. The haeder is the piece of the file that will allow us to map original characters to associated codes from the Huffman tree. Since we already have an implementation that buils a tree out of frequencies, let's serialize those frequencies as the header. We will do it as follows:

1. Each character shuold be a utf-8 encoded version of the original character, because we need the contents of the file to be written in bytes to take advantage of the compression that will arise from hufman codes. 
2. Its count will be an integer expressed as bytes, as storing the integer literals will take up more space (one byte per digit, while a single byte can hold more values).
3. Each character-count pair is going to be separated by a comma.
4. Once we finish writing the table, we should write `\n**\n` to mark the section of the file where the compressed contents are meant to be written. 

Now, if we had frequencies like `{{"a": 12, "b": 256, "á": 1}`, we would end up with a file looking like:

```
"a-\x0c,b-\x01\x00,\xc3\xa1-\x01"
**
[...]
```
where `[...]` represents the contents of the file, but we will see how to do that in a later step. 

How to test for that? Here is one idea:

```python
from src import serialize

@pytest.mark.parametrize(
    "frequencies,header",
    [
        ({"a": 12}, b"a-\x0c"),
        ({"a": 256}, b"a-\x01\x00"),
        ({"a": 12, "b": 256}, b"a-\x0c,b-\x01\x00"),
        ({"á": 1}, b"\xc3\xa1-\x01"),
    ],
)
def test_header_values(frequencies: dict[str, int], header: bytes):
    result = serialize.create_header(frequencies)
    assert result.startswith(header)
    assert result.endswith(b"\n**\n")


def test_header_doesnot_write_content():
    header = serialize.create_header({"a": 0})
    _, content = header.split(b"\n**\n")
    assert content == b""
```

Go ahead and try to implement it yourself, try to solve one test case at the time! At this step we are also required to improve the tests of our `main` function to check that a file can be written with that header. I will ommit that part from this post.

## Step five

Here we see how everything fits together. We understand how we managed to achieve compression and also the limits of our algorithm. The idea is that, since we have binary codes for each character, we can group them together to form bytes, so that a single character would take less than a byte.

Here is a hypothetical example: suppose we have a text file whose only content is `abc` and somehow our prefix-code table looks like `{"a": "11", "b": "111", "c": "111"}`, then this means we would only need to store `\xff` in the file, instead of the bytes associated with each of the original characters (1 byte instead of 3).

We need a function that takes in the contents of the source file and the prefix code table, we can define it as:

```python
def create_payload(source: str, table: dict[str, str]) -> bytes:
    ...
```

which should at least pass the following tests:

```python
def test_single_character_payload_leads_to_single_byte():
    result = serialize.create_payload("a", {"a": "0"})
    assert result == b"\x00"

def test_small_payload_codes_are_grouped_in_bytes():
    result = serialize.create_payload(
        "abc", {"a": "11", "b": "111", "c": "111"}
    )
    assert result == b"\xff"

def test_big_payload_codes_are_grouped_in_bytes():
    result = serialize.create_payload(
        "ab", {"a": "1111111100000000", "b": "1111111100000000"}
    )
    assert result == b"\xff\x00\xff\x00"

def test_last_bits_are_padded_into_a_byte():
    result = serialize.create_payload(
        "ab", {"a": "0000000011111", "b": "111"}
    )
    assert result == b"\x00\xff"
```

Again, try to make those test pass one at the time. 

## Food for Thought

I ommited steps six and seven entirely to avoid a huge post, but you can check my full implementation at https://github.com/srcolinas/codingchallenges_solutions/tree/main/compression-tool

Here are some things to think about:

* Remember that the output file will also contain a hedear with the frequencies for each character, so the final amount of bytes is the bytes in the payload + the bytes in the header. If we have a large document, we will still achieve some compression, so that is fine.
* I originally thought I didn't need to write the frequencies to the output file and then build the tree from that. I thought I could just write the prefix-code table and I would be able to restore a file. Think about why if you encounter the same issue. 


---

[Suggest Edits](https://github.com/srcolinas/srcolinas.github.io/tree/master/content/compression-tool.md)

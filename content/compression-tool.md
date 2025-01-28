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
    assert subtree.children is not None
    assert first.children is None
    assert first.weight == 2
    assert second.children is None
    assert second.weight == 24
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
        return self.weight > other.weight

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
                weight=31,
                children=(
                    HuffmanTree(weight=24, key="m"),
                    HuffmanTree(weight=7, key="k"),
                ),
            ),
            HuffmanTree(weight=2, key="z"),
        ),
    )
    table = create_prefix_code_table(tree)
    assert table == {"m": "00", "k": "01", "z": "1"}
```

In real life, we never know whether our tests are covering every single edge case out there, so whatever I write and whatever you write may have ways to break. Usually that is the way it works: whenever we (or a user :( ) discover new ways to break our code, we should just update the test cases to reflect it and fix it.

Since the implementation is not the goal of the post, I will omit it for now.

## Food for Thought


---

[Suggest Edits](https://github.com/srcolinas/srcolinas.github.io/tree/master/content/compression-tool.md)

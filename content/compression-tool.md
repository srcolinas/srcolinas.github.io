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
from typing import Union, Hashable


@dataclasses.dataclass(slots=True, frozen=True, kw_only=True)
class HuffmanTree:
    weight: int
    left: Union["HuffmanTree", None]
    right: Union["HuffmanTree", None]

    @classmethod
    def from_frequenceies(
        cls, frequencies: dict[Hashable, int]
    ) -> "HuffmanTree":
        raise NotImplementedError
```

Let's go step by step into implementing this class. The first thing that comes to mind is that, if there is a single character that is repeated any number of times, then it should have the number of times it is repeated as weight and then the left and right nodes should be `None`; in other words, it should satisfy the following tests:

```python
from src.huffman_tree import HuffmanTree


def test_build_with_single_element_has_no_children():
    tree = HuffmanTree.from_frequenceies({"a": 3})
    assert tree.left is None
    assert tree.right is None


def test_build_with_single_element_has_right_weight():
    tree = HuffmanTree.from_frequenceies({"a": 3})
    assert tree.weight == 3
```

One possible implementation is

```python
[...]
@dataclasses.dataclass(slots=True, frozen=True, kw_only=True)
class HuffmanTree:
    weight: int
    left: Union["HuffmanTree", None] = dataclasses.field(default=None)
    right: Union["HuffmanTree", None] = dataclasses.field(default=None)

    @classmethod
    def from_frequenceies(
        cls, frequencies: dict[Hashable, int]
    ) -> "HuffmanTree":
        _, value = frequencies.popitem()
        return cls(weight=value)
```

It is true, it only supports one element, so let's add support for at least two elements, in other words, let's add the following tests that our implementation needs to satisfy:

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
            new = cls(weight=left.weight + right.weight, left=left, right=right)
            trees.append(new)
        return trees.pop()
```

Now, if we have three or more elements, the implementation becomes more interesting: we should make sure that the children were added in the correct order, which is could be concluded by its weight. I decided to code a tree given in one of the recommended readings from the Coding Challenges' guide, so that you could visualize it better, here I show you a piece of it:

```python
def test_build_from_three_elements_has_right_weight():
    tree = HuffmanTree.from_frequenceies(
        {"z": 2, "k": 7, "m": 24, "c": 32, "u": 37, "d": 42, "l": 42, "e": 120}
    )
    assert tree.weight == 306
    
    assert tree.left is not None
    assert tree.left.weight == 120
    assert tree.right is not None
    assert tree.right.weight == 186

    [...]
```

Make sure you go to the final section if you want to see the full test. Here is my implementation that works with all tests so far:

```python
import dataclasses
import heapq
from typing import Union, Hashable


@dataclasses.dataclass(slots=True, frozen=True, kw_only=True, order=True)
class HuffmanTree:
    weight: int
    left: Union["HuffmanTree", None] = dataclasses.field(default=None, compare=False)
    right: Union["HuffmanTree", None] = dataclasses.field(default=None, compare=False)

    @classmethod
    def from_frequenceies(
        cls, frequencies: dict[Hashable, int]
    ) -> "HuffmanTree":
        trees : list[HuffmanTree] = []
        for v in frequencies.values():
            heapq.heappush(trees, cls(weight=v))

        while len(trees) > 1:
            left = heapq.heappop(trees)
            right = heapq.heappop(trees)
            new = cls(weight=left.weight + right.weight, left=left, right=right)
            heapq.heappush(trees, new)
        return trees.pop()
```


You may be wondering about the fact that we are not storing the characters anywhere in the tree, only the weights. It is because we don't need that for now up to step two, remember that we add bits of complexity only when it is needed.


## Food for Thought


---

[Suggest Edits](https://github.com/srcolinas/srcolinas.github.io/tree/master/content/json-parser.md)

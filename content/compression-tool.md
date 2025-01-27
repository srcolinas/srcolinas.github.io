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

## Step One

From the Challenge description:

>In this step your goal is to build your tool to accept a filename as input. It should return an error if the file is not valid, otherwise your program should open it, read the text and determine the frequency of each character occurring within the text.

I will skip all the boilerplate related to making a tool that takes a file as input, it is extremely similar to what I did for the json parser. Let's focus on 


## Food for Thought

My main goal was to guide your through the process of software development with TDD, but you can see the final solution in `https://github.com/srcolinas/codingchallenges_solutions/tree/main/json_parser/pyccjp` if you want to compare against yours.

A few things to keep in mind:
* I don't know whether the test suite suggested by the Coding Challenges guide is complete or not, so you may still encounter issues with the end implementation from time to time. However, that's how it works: whenever you find a new piece of the spec that doesn't fit the programm, you update the tests and then update the code accordingly.
* Keep in mind that this is an academic exercise on TDD, use standard JSON libraries in real life. 
* It is possible to come up with a solution that doesn't make use of iterators, we can instead make it with a `list`. Here I wanted to challenge myself making an implementation that only reads each character once and doesn't load the whole file into memory. I am sorry if that brought you unnecessary dificulties for you. 

---

[Suggest Edits](https://github.com/srcolinas/srcolinas.github.io/tree/master/content/json-parser.md)

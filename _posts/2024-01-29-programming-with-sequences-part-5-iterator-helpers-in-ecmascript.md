---
layout: post
title: Programming with sequences part 4 - iterator helpers in ECMAScript
date: 2024-01-29
tags: js powerseq
published: false
series: programming-with-sequences
---

{%- include /_posts/programming-with-sequences-series-toc.md -%}

## Introduction

At this point we should have a good understanding of iterators and generators in JS. Those fundamental building blocks allows us to write very elegant functional code using operators like `filter`, `map`, `reduce`, ... . [powerseq](https://github.com/marcinnajder/powerseq) library provides more than 70 general purpose operators. One of the key design decision behind this library is to allow for introducing a completely new operators and glue them together with existing ones. Operators are just a simple functions with a specified signatures (they take sequences and return a new sequences or values), then `pipe` function compose them into single query expression. To be honest, personally I cannot write any code without those operators. It does not matter which programming language I am currently programming in: JavaScript, TypeScript, C#, F#, Koltin, Clojure, ... This is just the way I look at programing problems.

Nowadays most of the technologies provide builtin implementations of such a operators, so we could ask ourself whether the community behind the JavaScript standard is thinking about this too. At the be

[proposal-iterator-helpers](https://github.com/tc39/proposal-iterator-helpers)

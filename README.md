# jute.clj

A Clojure implementation of the [JUTE template
language](https://github.com/HealthSamurai/jute.js).

[![Build Status](https://travis-ci.org/HealthSamurai/jute.clj.svg?branch=master)](https://travis-ci.org/HealthSamurai/jute.clj)

# Reference

## Introduction

JUTE stands for JSON Uniform Templates and it's a small language to
describe JSON documents transformations. JUTE templates are JSON
documents itself. It's safe to evaluate user-provided JUTE templates,
there is no way for a template to currupt a runtime environment [if
you use a safe YAML
parser](https://arp242.net/yaml-config.html#insecure-by-default).

## Few words about YAML

JSON format wasn't designed for ease of use by human beings, it's
relatively hard to write JSON by hands. That's why JUTE's primary
format is [YAML](https://yaml.org/), which is much easier to read and
write, thanks to its clean syntax and indentation-based nesting. Don't
be confused with it, YAML and JSON are interchangeable, and there are
even [online conversion tools](https://www.json2yaml.com/) beetween them.

## An example

Let's say we have a document describing a book:

```yaml
book:
  author:
    name: M. Soloviev
    title: PHD
    gender: m
  title: Approach to Cockroach
  chapters:
  - type: preface
    content: A preface chapter
  - type: content
    content: Chapter 1
  - type: content
    content: Chapter 2
  - type: content
    content: Chapter 3
  - type: afterwords
    content: Afterwords
```

And for some case we need to convert it into a slightly different
format:

```yaml
type: book
author: M. Soloviev
title: Approach to Cockroach
content:
- Chapter 1
- Chapter 2
- Chapter 3
```

Attentive reader will notice that we're going to discard preface and
afterwords as well as minor author information keepeing only his
name. And we want a book's content to be an array of strings, not an
array of objects with a `content` key. Let's write a JUTE template
which will perform this transformation.

We'll start our template with a `type: book` flag:

<table>
<thead>
<tr><th>Template</th><th>Result</th></tr>
</thead>
<tbody>
<tr><td>

```yml
type: "book"
```
</td><td>

```yml
type: "book"
```
</td></tr>
</tbody>
</table>

This tiny document is a valid JUTE template which will always procude
a `{"type": "book"}` result regardless the input data. Actually,
everything in a JUTE is treated as a constant value unless it doesn't
contain a special flag - a dollar sign. A dollar sign can appear
either in a object keys or as the first character of a string. Numbers
and boolean values (`true`/`false`) are always constants in JUTE
templates.

Let's move to the `author` field. Obviously we're gonna take an
author's name from an incoming data:

<table>
<thead>
<tr><th>Template</th><th>Result</th></tr>
</thead>
<tbody>
<tr><td>

```yml
type: "book"
author: "$ book.author.name"
```
</td><td>

```yml
type: "book"
author: "M. Soloviev"
```
</td></tr>
</tbody>
</table>

To tell JUTE that an `author` field will be dynamic we put a dollar
sign at the beginning of a value's string. The rest of the string is a
path for the data we need. Such strings starting with a dollar signs
are called JUTE **expressions** and they have pretty rich syntax to
describe various operations on an incoming data or a **scope**.

One of expression's abilities is an extract data by **path**. Every
path consists of one or several **path components** separated by
dot. In simpliest case a path component is a field name where JUTE
interpreter will dig to get value. In our case it fill take the `book`
field from the scope root, then `author`, then `name`. You can use
digits as path component as well to get N-th value from an
array. Array indices are starting with 0.

Please note that it's ok to omit double-quotes (`"`) for strings in
YAML, so instead of writing `"$ foo.bar"` we can just write `$
foo.bar`.

We can fill the `title` field using similar path expression and omit
double-quotes for readability:

<table>
<thead>
<tr><th>Template</th><th>Result</th></tr>
</thead>
<tbody>
<tr><td>

```yml
type: book
author: $ book.author.name
title: $ book.title
```
</td><td>

```yml
type: book
author: M. Soloviev
title: Approach to Cockroach
```
</td></tr>
</tbody>
</table>

Let's proceed to the `content` part. We need to filter out chapters
where `type` doesn't equal to `"content"`. There is a special type of
a path element to do this called **predicate search**:

<table>
<thead>
<tr><th>Template</th><th>Result</th></tr>
</thead>
<tbody>
<tr><td>

```yml
type: book
author: $ book.author.name
title: $ book.title
content: $ book.chapters.*(this.type = "content")
```
</td><td>

```yml
type: book
author: M. Soloviev
title: Approach to Cockroach
content:
- type: content
  content: Chapter 1
- type: content
  content: Chapter 2
- type: content
  content: Chapter 3
```
</td></tr>
</tbody>
</table>

Instead of telling an exact path, we describe a condition which an
array element should met to be selected for the next step of path
evaluation. Use `this` keyword to reference current element in an
array. A result of a predicate search is always an array, even if
there is only one element matching criteria.

The final step is to extract `content` property from every element in
the `content` array. In most programming languages nowadays it's done
using a [map
function](https://en.wikipedia.org/wiki/Map_(higher-order_function))
which executes same code on every element in an array and returns
results an array with preserved order. In JUTE we have `map` as well,
but it's not a function, it's a **directive**:

<table>
<thead>
<tr><th>Template</th><th>Result</th></tr>
</thead>
<tbody>
<tr><td>

```yml
type: book
author: $ book.author.name
title: $ book.title
content: 
  $map: $ book.chapters.*(this.type = "content")
  $as: i
  $body: $ i.content
```
</td><td>

```yml
type: book
author: M. Soloviev
title: Approach to Cockroach
content:
- Chapter 1
- Chapter 2
- Chapter 3
```
</td></tr>
</tbody>
</table>

## License

Copyright © 2019 Health Samurai Team

Distributed under the MIT License.

---
title: "New Go tools: docgo and litebrite"
layout: post
timestamp: February 20, 2012
---

I'm loving [Go](http://golang.org).  It has anonymous functions, syntax for
hash maps, language-level concurrency support, a unified build and package
management tool, a lightweight approach to object-oriented programming--nearly
everything I expect from a new language.  Goroutines and channels are fantastic.
I think these two features are the most important part of Go--they support so
many different idioms!  It's pretty surprising after writing a lot of Lisp last
year that I've adopted Go, but it feels like a really well-engineered solution.

One thing I like about the dynamic languages is their readability.  Python,
Ruby, CoffeeScript, and others occasionally read like real human language.  Go
is closer to this ideal than other statically-typed languages I've
used--especially compared to, for instance, C++.  To support this claim, I wrote
a clone of [docco](http://jashkenas.github.com/docco/) in Go.  docco is a
literate-programming-style documentation generator that turns source code into
beautiful HTML pages.  My program, [docgo](http://dhconnelly.github.com/docgo/),
is on GitHub.  Use it to publish your beautiful code!

The original docco uses [Pygments](http://pygments.org) for syntax highlighting.
Since I wanted my program to be 100% Go, and there isn't yet a general syntax
highlighting library written in Go, I wrote a small one, also available on
GitHub: [litebrite](http://dhconnelly.github.com/litebrite/).  This wasn't very
difficult, actually, since nearly all of the pieces of a Go compiler exist in
the standard library as separate packages.  I was able to import the scanner
and token packages and iterate over source code tokens with almost zero effort.

Go's expressiveness feels somewhere between Python and C.  The type system is
interesting: statically typed, like C, but the strong-typing is much closer to
Python.  Even arithmetic doesn't promote ints to floats automatically--explicit
conversions are required.  This sort of approach seems irritating at first, but
on the other hand, you don't need to remember as many rules as when programming
in C.

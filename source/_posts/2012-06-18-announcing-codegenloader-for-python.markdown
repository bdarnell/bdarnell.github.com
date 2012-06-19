---
layout: post
title: "Announcing CodeGenLoader for Python"
date: 2012-06-18 23:40
comments: true
categories:
---

[Protocol Buffers][protobuf] and [Thrift][thrift] are good formats for data
serialization, but they are less popular in dynamic languages like
Python than schemaless alternatives like JSON because of the need to
run a code generator on the schema file to create the necessary
classes.  [CodeGenLoader][] is a Python import hook that
automatically and transparently runs the necessary code generator at
import time.  It includes support for both Thrift and Protocol Buffers,
with an extensible base class that can be used to add more code generators.

All you need is two lines of magic in an `__init__.py`:

    import codegenloader.protobuf
    __path__ = codegenloader.protobuf.make_path(__name__, ".")

Get it with `pip install codegenloader` or check out the source
[on github][codegenloader].

[protobuf]: http://code.google.com/p/protobuf/
[thrift]: http://thrift.apache.org
[codegenloader]: https://github.com/bdarnell/codegenloader
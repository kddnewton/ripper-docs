## Ripper

Ripper is a Ruby standard library that acts as an event-based Ruby script parser. It can be used to gather information about Ruby scripts without running them. It can also generate [syntax trees](https://en.wikipedia.org/wiki/Abstract_syntax_tree) from source code.

Ripper is generated from Ruby's own parsing code, and is distributed with Ruby. That means the Ripper syntax is completely up-to-date with its corresponding Ruby.

When Ruby parses source code, it first [scans](https://en.wikipedia.org/wiki/Lexical_analysis) the source code into a series of tokens. When a scanner token is recognized, Ripper dispatches a scanner event. The scanner events are then grouped into parser events by applying the Ruby grammar. When a set of scanner events is grouped (i.e., the production rule in the grammar is reduced), Ripper dispatches a parser event.

Each event will create a node in the syntax tree. For instance, an assignment node ([assign](events.md#assign)) for `value = 7` would come from scanner nodes for `value` ([ident](events.md#ident)), `=` ([op](events.md#op)) and `7` ([int](events.md#int)). A parser event can group parser events, scanner events, or both.

Ruby's parser generator (in `parse.y`) contains [comments for Ripper to use](https://github.com/ruby/ruby/blob/79f9f8326a34e499bb2d84d8282943188b1131bd/parse.y#L1519). When Ripper is compiled, it modifies Ruby's compiler code in `parse.y` based on these comments to make its own compiler code, very similar to Ruby's.

Ripper allows you to define handlers for scanner and parser events, which can let you analyze Ruby programs, or recognize control structures (e.g. an assignment inside an `if` statement, which is often an error).

This document contains notes on various subjects pertaining to Ripper, as well as references for all of the various events and methods that Ripper uses internally. Here is a list of the pages that you can find in this document:

* [Usage](usage.md) will give you a general overview of how to use Ripper for your own purposes
* [Naming](naming.md) will give you an idea of some of the event types that Ripper contains and how they are named
* [Lists](lists.md) will help you understand how Ripper represents and parses lists of values
* [Location](location.md) explains how to determine source location information inside event handlers
* [Events](events.md) contains a reference for every scanner and parser event that Ripper dispatches

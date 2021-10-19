# Location

For a lot of tools that interact with a syntax tree, you will need information about where every node comes from in the source code. For that, `Ripper` provides two methods: `lineno` and `column`. These methods have a lot of nuance, however, so it's important to know what they're actually referring to.

## `lineno`

`lineno` refers to line number that is stored in the internal parser state as it is parsing. It has different behavior depending on if you're handling a scanner event or a parser event. Note that line-numbers are 1-indexed (the first line in the file is line 1, not 0).

If you're handling a scanner event, the `lineno` is always going to be the line number at the start of the token that triggered that scanner event. For example if you have an [ident](events#ident) scanner event `variable`, then the value will be whatever line the `v` is on.

If you're handling a parser event, then `lineno` will be the line number that the parser is looking at when the production rule that you're referencing is reduced. Note that this is not necessarily the line number of the final token in the rule.

## `column`

`column` refers to the byte offset that is stored in the internal parser state as it is parsing an individual line. Similar to how it works for `lineno`, it has different behavior depending on the kind of event you're handling.

If you're handling a scanner event, then `column` will always be the byte offset in the line that the token appears on _before_ the token itself. So for example, if you're parsing `1 + 2`, then in the [int](events#int) scanner event handler for the `2` in that statement, `column` will have a value of `4`.

If you're handling a parser event, the `column` will be the byte offset that the parser is looking at when the production rule that you're referencing is reduced. This can be especially confusing when you factor in comments. Let's say you're parsing `1 + 2 # 3`. Both of the [int](events#int) scanner events will have the expected column values (`0` and `4`), whereas the [binary](events#binary) node that represents that addition will have a `column` value of `9` (because that is the point at which the parser reduced the [binary](events#binary)).

It's important here to note the difference between offset and byte offset. If you're parsing a string of source like `party = "🎉"`, the overall [assign](events#assign) node that represents this assignment will have a `column` value of `14` (5 for the variable, 3 for the spaces and operator, 2 for quotes, and 4 bytes to represent the emoji). Note that this is also encoding-specific, as different encodings will represents codepoints with differing numbers of bytes.

For these reasons, it's easiest to rely on location information from the scanner events for determining the bounds of parser events. For example, if you're handling a [while](events#while) event, you can look for the `while` and `end` keywords to know the bounds of your node.

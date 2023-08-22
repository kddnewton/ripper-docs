## Events

Below is a reference for the different scanner and parser events defined in Ripper. Each event specifies:

* Whether it is a "parser" event or a "scanner" event. Things like identifiers, constants, operators, and the like are scanner events, while higher-level nodes like assignment, conditionals, and loops are parser events.
* An example of the syntax that would trigger the event.
* How you would write a handler method for this event, and the types of objects that would be passed in as arguments to that handler.

### `BEGIN`

`BEGIN` is a parser event that represents the use of the `BEGIN` keyword, which hooks into the lifecycle of the interpreter. Whatever is inside the block will get executed when the program starts. The syntax looks like the following:

```ruby
BEGIN {
}
```

Interestingly, BEGIN doesn't allow the `do` and `end` keywords for the block. Only braces are permitted.

The handler for this event accepts one parameter that is always a `stmts_add` node:

```ruby
def on_BEGIN(stmts_add); end
```

### `CHAR`

`CHAR` is a scanner event that represents a single codepoint in the script encoding. For example:

```ruby
?a
```

is a representation of the string literal `"a"`. You can use control characters with this as well, as in `?\C-a`. The handler for this event accepts one parameter that is always a string:

```ruby
def on_CHAR(value); end
```

### `END`

`END` is a parser event that represents the use of the `END` keyword, which hooks into the lifecycle of the interpreter. Whatever is inside the block will get executed when the program ends. The syntax looks like the following:

```ruby
END {
}
```

Interestingly, you can't use the `do` and `end` keywords for the block. The handler for this event accepts one parameter that is always a `stmts_add` node:

```ruby
def on_END(stmts_add); end
```

### `__end__`

`__end__` is a scanner event that represents `__END__` syntax, which allows individual scripts to keep content after the main ruby code that can be read through the `DATA` constant. It looks like:

```ruby
puts DATA.read

__END__
some other content that is not executed by the program
```

The handler for this event accepts one parameter that is always a string containing `"__END__\n"` (provided you're using `\n` as the newline). Notably it does not contain the content appearing after the keyword, so you need to use the line number information to access that.

```ruby
def on___end__(value); end
```

### `alias`

`alias` is a parser event that represents the use of the `alias` keyword with regular arguments (not global variables). The `alias` keyword is used to make a method respond to another name as well as the current one. For example, to get the method `name` to also respond to `aliased_name`, you would:

```ruby
alias aliased_name name
```

Now, in the current context you can call `aliased_name` and it will execute the `name` method. When you're aliasing two methods, you can either provide bare words (like the example above) or you can provide symbols (note that this includes dynamic symbols like `:"left-#{middle}-right"`).

The handler for this event accepts two parameters, that correspond to the first and second arguments to the keyword. So, for the above example the left would be the symbol literal `aliased_name` and the right could be the symbol literal `name`. Either argument can be a [dyna_symbol](#dyna_symbol) node or a [symbol_literal](#symbol_literal) node.

```ruby
def on_alias(left, right); end
```

### `aref`

`aref` is a parser event when you're pulling a value out of a collection at a specific index. Put another way, it's any time you're calling the method `#[]`. As an example:

```ruby
collection[index]
```

The nodes usually contains two children, the collection and the index. In some cases, you don't necessarily have the second child node, because you can call procs with a pretty esoteric syntax. In the following example, you wouldn't have a second child node:

```ruby
collection[]
```

Because the left-hand side of this expression can be any primary Ruby expression, its type is not known. The right-hand side (if provided) will be either an [args_add](#args_add) node or an [args_add_block](#args_add_block) node.

```ruby
def on_aref(collection, index); end
```

### `aref_field`

`aref_field` nodes are for assigning values into collections at specific indices. Put another way, it's any time you're calling the method `#[]=`. The `aref_field` node itself is just the left side of the assignment, and they're always wrapped in [assign](#assign) nodes. As an example:

```ruby
collection[index] = value
```

The nodes always contain two children, the expression that corresponds to the collection being indexed and the index itself. The collection is any primary Ruby expression. The index can optionally be omitted (in the very rare case that someone defines a `#[]=` method that accepts no arguments).

```ruby
def on_aref_field(collection, index); end
```

### `arg_ambiguous`

`arg_ambiguous` is a parser event that represents when the parser sees an argument as ambiguous. For example, in the following snippet:

```ruby
value //
```

the question becomes if the forward slash is being used as a division operation or if it's the start of a regular expression.

Unlike most other parser events, the output of the handler method for this event is not passed up to any parent nodes, as it is not a node in the resulting tree. As such, it does not matter what value is returned from the handler method. The handler accepts one parameter that is a string representing the ambiguous argument. In the above example, it would be `"/"`.

```ruby
def on_arg_ambiguous(value); end
```

### `arg_paren`

`arg_paren` is a parser event that represents wrapping arguments to a method inside a set of parentheses. For example, in the follow snippet:

```ruby
method(argument)
```

there would be an `arg_paren` node around the [args_add_block](#args_add_block) node that represents the set of arguments being sent to the `method` method. The argument child node can be `nil` if no arguments were passed, as in:

```ruby
method()
```

The handler for this event accepts one parameter, which is either an [args_add](#args_add), [args_add_block](#args_add_block), or [args_forward](#args_forward) node. It can also optionally be `nil`.

```ruby
def on_arg_paren(args); end
```

### `args_add`

`args_add` is a parser event that represents a single argument inside a list of arguments to any method call or an array. It accepts as arguments the previous `args_add` or [args_new](#args_new) node as well as a value which can be anything that could be passed as an argument.

For example, in the following snippet:

```ruby
method(first, second, third)
```

you would first receive an event for [args_new](#args_new). Then for each subsequent argument, you would receive an event for `args_add`, with the following arguments:

* The result of [args_new](#args_new) and the [vcall](#vcall) node for `first`
* The result of the first `args_add` call and the [vcall](#vcall) node for `second`
* The result of the second `args_add` call and the [vcall](#vcall) node for `third`

Note that because of the nature of the chaining here, it's not possible to know if you're on the last argument to a list without further inspecting the source through the scanner events.

```ruby
def on_args_add(args, arg); end
```

### `args_add_block`

`args_add_block` is a parser event that represents a list of arguments and potentially a block argument. If no block is passed, then the second argument will be the literal `false`. `args_add_block` is commonly seen being passed to any method where you use parentheses (wrapped in an [arg_paren](#arg_paren) node). It's also used to pass arguments to the various control-flow keywords like `return`.

For example, in the following snippet, the `&block` would trigger the `args_add_block` to have a [vcall](#vcall) node as its second argument:

```ruby
method(argument, &block)
```

whereas in the following snippet, you would have an `args_add_block` node with `false` as the block:

```ruby
method(argument)
```

The handler for this event accepts the arguments (as an [args_new](#args_new), [args_add](#args_add), or [args_add_star](#args_add_star) node) and the optional block (which is usually a [vcall](#vcall) node, or `false` if it's not passed).

```ruby
def on_args_add_block(args, block); end
```

### `args_add_star`

`args_add_star` is a parser event that represents adding a splat of values to a list of arguments. For example, in the following snippet, the `*arguments` would trigger an `args_add_star` event:

```ruby
method(prefix, *arguments, suffix)
```

This event is very similar to the [args_add](#args_add) event except that whatever expression is being used as an argument is prefixed with the splat operator. The handlers for this event accepts two arguments: the parent arguments node as well as the expression that is being splatted.

```ruby
def on_args_add_star(args, arg); end
```

### `args_forward`

`args_forward` is a parser event that represents forwarding all kinds of arguments onto another method call. For example, in the following snippet:

```ruby
def request(method, path, **headers, &block); end

def get(...)
  request(:GET, ...)
end

def post(...)
  request(:POST, ...)
end
```

both the `get` and `post` methods are forwarding all of their arguments (positional, keyword, and block) on to the `request` method. The `args_forward` node appears in both the caller (the `request` method calls) and the callee (the `get` and `post` definitions).

The handler for this event accepts no arguments. It is passed up to the parent argument node (be it an [args_add](#args_add) or [args_add_block](#args_add_block) node).

```ruby
def on_args_forward; end
```

### `args_new`

`args_new` is a parser event that represents the beginning of a list of arguments to any method call or an array. It can be followed by any number of [args_add](#args_add), [args_add_block](#args_add_block), or [args_forward](#args_forward) events, which end up in a chain. For example, in the following snippet:

```ruby
method(argument)
```

there will be one [args_add](#args_add) node that contains as its first child an `args_new` node and its second child a [vcall](#vcall) node for the argument. The handlers for this event accepts no arguments. It is passed up to the parent argument node (be it an [args_add](#args_add) or [args_add_block](#args_add_block) node).

```ruby
def on_args_new; end
```

### `array`

`array` is a parser event that contains myriad child nodes because of the special array literal syntax like `%w` and `%i`. For example, the following lines all produce an `array` event:

```ruby
[]
[one, two, three]
[*one_two_three]
%w[one two three]
%i[one two three]
%W[one two three]
%I[one two three]
```

In order, the child node types coming in would be `nil`, [args_add](#args_add), [args_add_star](#args_add_star), [qwords_add](#qwords_add), [qsymbols_add](#qsymbols_add), [words_add](#words_add), and [symbols_add](#symbols_add). The handler for this event should account for those various types.

```ruby
def on_array(contents); end
```

### `aryptn`

`aryptn` is a parser event that represents matching against an array pattern using the Ruby `2.7`+ pattern matching syntax. It's one of the more complicated events, because the four parameters that it accepts can almost all be `nil`. First, let's look at what kind of code triggers this event:

```ruby
case [1, 2, 3]
in [Integer, Integer]
  "matched"
in Container[Integer, Integer]
  "matched"
in [Integer, *, Integer]
  "matched"
end
```

An `aryptn` event triggers with four parameters: an optional constant wrapper, an array of positional matches, an optional splat with identifier, and an optional array of positional matches that occur after the splat. All of the `in` clauses above would trigger an `aryptn` event.

In the first clause, the first parameter would be `nil` because it's not being wrapped with a constant. The second parameter would be an array of nodes representing the two positional matches. The third parameter would be `nil` because there's no splat operator, and the fourth parameter would be `nil` for the same reason.

In the second clause, the first parameter would be a [const](#const) node containing the `Container` name, and the rest of the parameters would be the same as in the first clause.

The final clause would have a `nil` constant, a single-element array containing the first `Integer` match for the second parameter, a [var_field](#var_field) node containing the splat for the third parameter, and another single-element array containing the final `Integer` match for the fourth parameter.

```ruby
def on_aryptn(const, preargs, splatarg, postargs); end
```

### `assign`

`assign` is a parser event that represents assigning something to a variable or constant. Generally, the left side of the assignment is going to be any node that ends with the name `field` (see [naming](#naming)).

```ruby
variable = value
```

It accepts as arguments the left side of the expression before the equals sign and the right side of the expression. The right side of the expression can be any of most of the nodes in the tree.

```ruby
def on_assign(left, right); end
```

### `assoc_new`

`assoc_new` is a parser event that contains a key-value pair within a hash. It is a child event of either an [assoclist_from_args](#assoclist_from_args) or a [bare_assoc_hash](#bare_assoc_hash).

```ruby
{ key1: value1, key2: value2 }
```

In the above example, the would be two `assoc_new` nodes. Each would contain a key ([label](#label) nodes) and a value ([vcall](#vcall) nodes).

```ruby
def on_assoc_new(key, value); end
```

### `assoc_splat`

`assoc_splat` is a parser event that represents double-splatting a value into a hash (either a hash literal or a bare hash in a method call).

```ruby
{ **pairs }
```

Much like `assoc_new`, these nodes get added into an array that gets sent up to either an [assoclist_from_args](#assoclist_from_args) or a [bare_assoc_hash](#bare_assoc_hash). The handler for this event receives a single parameter the represents the value being double-splatted (which can be any Ruby expression).

```ruby
def on_assoc_splat(contents); end
```

### `assoclist_from_args`

`assoclist_from_args` is a parser event that represents the key-value pairs of a hash literal. Its parent node is always a hash.

```ruby
{ key1: value1, key2: value2 }
```

The handler for this event accepts a single parameter: an array containing either [assoc_new](#assoc_new) or [assoc_splat](#assoc_splat) nodes.

```ruby
def on_assoclist_from_args(assocs); end
```

### `backref`

`backref` is a scanner event that represents a global variable referencing a matched value. It comes in the form of a $ followed by a positive integer.

```ruby
$1
```

The handler accepts a single string parameter containing the value as seen in the source.

```ruby
def on_backref(value); end
```

### `backtick`

`backtick` is a scanner event that represents the use of the \` operator. It's usually found being used for an [xstring_literal](#xstring_literal), but could also be found as the name of a method being defined.

```ruby
`ls`
```

The above example would trigger two `backtick` events. The handler accepts a single string parameter that always contains a single backtick.

```ruby
def on_backtick(value); end
```

### `bare_assoc_hash`

`bare_assoc_hash` is a parser event that represents a hash of contents being passed as a method argument (and therefore has omitted braces). It's very similar to an [assoclist_from_args](#assoclist_from_args) event.

```ruby
method(key1: value1, key2: value2)
```

In the above example, the event would be triggered for the source bound between the `key1` [label](#label) and the `value2` [vcall](#vcall). The handler for this event accepts a single array parameter of assoc events (either [assoc_new](#assoc_new) or [assoc_splat](#assoc_splat)).

```ruby
def on_bare_assoc_hash(assocs); end
```

### `begin`

`begin` is a parser event that represents the beginning of a `begin`..`end` chain. If may also happen as the result of using the "pin" operator `^` in the pattern of an [in](#in) statement.

```ruby
begin
  value
end
case value
  in ^(/a/)
  in {released_at: ^(Time.new(2010)..Time.new(2020))}
end
```

The handler for this event accepts a single event. In the case of a `begin`..`end` chain, it is a [bodystmt](#bodystmt) that has all of the potential clauses (`rescue`/`else`/`ensure`). When pinning, it is the pinned expression.

```ruby
def on_begin(body_or_pinned); end
```

### `binary`

`binary` is a parser event that represents any expression that involves two sub-expressions with an operator in between. This can be something that looks like a mathematical operation:

```ruby
1 + 1
```

but can also be something like pushing a value onto an array:

```ruby
array << value
```

The handler for this event accepts three parameters. The first and last represent the left- and right-hand side of the operation. The second parameter represents the operator being used. On most Ruby implementations, the operator is a symbol (like `:+` or `:<<` for the examples above). However, on JRuby, it's an [op](#op) node which matches some other event handlers.

```ruby
def on_binary(left, operator, right); end
```

### `block_var`

`block_var` is a parser event that represents the parameters being declared for a block. Effectively this event is everything contained within the pipes. This includes all of the various parameter types, as well as block-local variable declarations. For example:

```ruby
method do |positional, optional = value, keyword:, &block|
end
```

In the above example, all of those parameters would be included inside the child [params](#params) node. You can use a relatively esoteric syntax for declaring block-local variables, as in:

```ruby
method do |positional; local|
end
```

With this syntax, `local` becomes a variable local only to the block, and can shadow an outer variable without overwriting or modifying it. The handler for this event accepts two parameters. The first is the [params](#params) node that contains all of the declared parameters. The second is an optional array of [ident](#ident) nodes that represent any declared block-local variables. If none are present, then the second parameter will be the literal `false`.

```ruby
def on_block_var(params, locals); end
```

### `blockarg`

`blockarg` is a parser event that represents declaring a block parameter on a method definition.

```ruby
def method(&block); end
```

In the above example, the `&block` would trigger a `blockarg` event. The handler for this event always accepts a single [ident](#ident) parameter which represents the name of the block parameter.

```ruby
def on_blockarg(ident); end
```

### `bodystmt`

`bodystmt` is a parser event that represents all of the possible combinations of clauses within the body of a method or do block. This means the regular statements and the optionally attached `rescue`, `else`, and `ensure` blocks. It can be passed up to any node that accepts these kinds of bodies like method definitions, lambdas using the `do` keyword, class and module definitions, singleton class scopes, and [begin](#begin) nodes. For example:

```ruby
class Container
rescue
end

class << self
rescue
end

lambda do
rescue
end
```

In all three of the above snippets, the `rescue` keyword is used to indicate that a `bodystmt` node is being passed up to the various parent nodes. The handler for this event accepts four parameters. The first is a [stmts_add](#stmts_add) node representing the first (and only required) set of statements. The second is an optional [rescue](#rescue) node. The third is an optional second set of statements that belong in the `else` clause if one is given. The fourth and final parameter is an optional [ensure](#ensure) node.

```ruby
def on_bodystmt(stmts, rescued, elsed, ensured); end
```

Note that it's difficult to determine the character bounds of this node since it doesn't necessarily know where it started. You can look at the first child node that you encounter, but that might be missing comments that conceptually "belong" to this node. To remedy this, if you need the chracter bounds you need to determine them in each of the parent event handlers.

### `brace_block`

`brace_block` is a parser event that represents passing a block to a method call using the `{` `}` operators. For example:

```ruby
method { variable + 1 }
method { |variable| variable + 1 }
```

The handler for this event accepts as arguments an optional [block_var](#block_var) event that represents any parameters to the block as well as a [stmts_add](#stmts_add) event that represents the statements inside the block. So in the first line of the example above the first parameter would be `nil`, but in the second line it would be present.

```ruby
def on_brace_block(block_var, stmts); end
```

### `break`

`break` is a parser event that represents using the `break` keyword. For example:

```ruby
break
break 1
```

The handler for this event accepts one parameter that is an [args_new](#args_new) event (in the case of no parameters, as in the first line of the example) or an [args_add_block](#args_add_block) event (in the case parameters were passed, as in the second line of the example) that contains all of the arguments being passed to the `break` keyword.

```ruby
def on_break(args); end
```

### `call`

`call` is a parser event representing a method call. This event doesn't contain the arguments being passed (if arguments _are_ passed, this node will get nested under a [method_add_arg](#method_add_arg) node).

```ruby
receiver.message
```

There is one esoteric syntax that comes into play here as well. If the last parameter to the handler for this event is the symbol literal `:call`, then it represents calling a callable object in a very odd looking way, as in:

```ruby
callable.(1, 2, 3)
```

The handler for this event accepts as parameters the receiver of the message (which can be another nested `call` as well), the operator being used to send the message (this can be an [op](#op) node containing `.` or `&.`, or the symbol literal `:"::"`), and the message that is being sent to the receiver. The message will usually be an [ident](#ident) node, but can also be a [backtick](#backtick), [op](#op), or [const](#const) node, depending on the look of the message. It can also be the already mentioned `:call` symbol literal.

```ruby
def on_call(receiver, operator, message); end
```

### `case`

`case` is a parser event that represents the beginning of a `case` chain. For example:

```ruby
case value
when 1
  "one"
when 2
  "two"
else
  "number"
end
```

`case` will also get dispatched when using the `in` keyword on a single line, as in:

```ruby
value in pattern
```

Finally, `case` will get dispatched when using right-hand assignment in Ruby 2.7+, as in:

```ruby
expression => value
```

The handler for this event accepts two parameters: the optional value that is being used as the predicate for the `case` chain, and the consequent clause (a [when](#when) or [in](#in) clause).

```ruby
def on_case(switch, consequent); end
```

### `class`

`class` is a parser event that represents defining a class using the `class` keyword. For example:

```ruby
class Container
end
```

Classes can have path names as their class name in case it's being nested under a namespace, as in:

```ruby
class Namespace::Container
end
```

Classes can also be defined as a top-level path, in the case that it's already in a namespace but you want to define it at the top-level instead, as in:

```ruby
module OtherNamespace
  class ::Namespace::Container
  end
end
```

All of these declarations can also have an optional superclass reference, as in:

```ruby
class Child < Parent
end
```

That superclass can actually be any Ruby expression, it doesn't necessarily need to be a constant, as in:

```ruby
class Child < method
end
```

The handler for this event accepts as parameters the name of the class (a [const_path_ref](#const_path_ref), [const_ref](#const_ref), or [top_const_ref](#top_const_ref) node), the optional value of the superclass (any Ruby expression), and the [bodystmt](#bodystmt) event that represents the statements evaluated within the context of the class.

```ruby
def on_class(const, superclass, bodystmt); end
```

### `comma`

`comma` is a scanner event that represents the use of the `,` operator. In general you don't necessarily need to handle this event, but it's useful for determining if the source is using trailing commas in literals or method calls.

```ruby
method(
  parameter1,
  parameter2,
)
```

The above example would trigger two `comma` events. The handler accepts a single string parameter that always contains a single comma.

```ruby
def on_comma(value); end
```

### `command`

`command` is a parser event representing a method call with arguments and no parentheses. Note that `command` events only happen when there is no explicit receiver for this method, as in the following example:

```ruby
method argument
```

The handler for this event accepts as arguments the name of the method (either a [const](#const) or an [ident](#ident) node) and the arguments being passed to the method (as an [args_add_block](#args_add_block) node).

```ruby
def on_command(message, args); end
```

### `command_call`

`command_call` is a parser event representing a method call on an object with arguments and no parentheses.

```ruby
object.method argument
```

The handler for this event accepts as arguments the receiver of the method (any Ruby expression), the operator being used to send the method (this can be an [op](#op) node containing `.` or `&.`, or the symbol literal `:"::"`), the name of the method (an [op](#op), [ident](#ident), or [const](#const) node, depending on what the method name looks like), and the arguments being passed to the method (an [args_add_block](#args_add_block) node).

```ruby
def on_command_call(receiver, operator, method, args); end
```

### `comment`

`comment` is a scanner event that represents a comment in source.

```ruby
# comment
```

The above example would trigger one `comment` events. The handler for this event accepts a single string parameter that contains the value of the comment (including the `#` sign).

```ruby
def on_comment(value); end
```

Note that this is one of a few scanner events that give you a string value that can potentially extend beyond the range of ASCII. If there's a magic encoding comment at the top of the source that ripper is parsing (as in `# encoding: Shift_JIS`) then the encoding of the string being passed into the handler for this event will match it. If you're planning on serializing the resulting syntax tree in any way, it's important to handle the encoding change in this event handler.

### `const`

`const` is a scanner event that represents a literal value that _looks like_ a constant. This could actually be a reference to a constant:

```ruby
Const
```

It could also be something that looks like a constant in another context, as in a method call to a capitalized method:

```ruby
object.Const
```

or a symbol that starts with a capital letter:

```ruby
:Const
```

The handler for this event accepts a single string parameter that contains the value of the const.

```ruby
def on_const(value); end
```

### `const_path_field`

`const_path_field` is a parser event that always represents the child node of some kind of assignment. It represents when you're assigning to a constant that is being referenced as a child of another variable. For example:

```ruby
object::Const = value
```

The handler for this event accepts two parameters. The first is the value to the left of the `::` operator. It's the parent object of the constant that is being assigned. It can be any Ruby expression. The second is the [const](#const) node that represents the constant being assigned (as in `Const` in the example above).

```ruby
def on_const_path_field(left, const); end
```

### `const_path_ref`

`const_path_ref` is a parser event that is a very similar to [const_path_field](#const_path_field) except that it is not involved in an assignment. It looks like the following example:

```ruby
object::Const
```

The handler for this event is the same as [const_path_field](#const_path_field). The first parameter for the left side of the `::` operator and the second parameter for the right.

```ruby
def on_const_path_ref(left, const); end
```

### `const_ref`

`const_ref` is a parser event that represents the name of the constant being used in a class or module declaration. In the following example it is the [const](#const) scanner event that has the contents of `Container`.

```ruby
class Container; end
```

The handler for this event always accepts a single [const](#const) parameter.

```ruby
def on_const_ref(const); end
```

### `cvar`

`cvar` is a scanner event that represents the use of a class variable.

```ruby
@@variable
```

The handler for this event accepts a single string parameter that contains the name of the variable including the `@@` prefix.
 
 ```ruby
def on_cvar(value); end
```

### `def`

`def` is a parser event that represents defining a regular method on the current self object. It accepts as arguments an identifier naming the method being defined, a [params](#params) or [paren](#paren) node (the parameter declaration for the method), and a [bodystmt](#bodystmt) node which represents the statements inside the method. As an example, here are the parts that go into this:

```ruby
def method(param) do result end
```

In this case `method` would be the [ident](#ident), `param` would be inside a [params](#params) node, itself in a [paren](#paren) node, and [bodystmt](#bodystmt) would contain the single `result` statement.

You can also have single-line methods since Ruby 3.0+, which have slightly different syntax but still flow through this event handler. Those look like:

```ruby
def method = result
```

In this case `method` would be the [ident](#ident), the [params](#params) node would contain only `nil` values since this single-line method doesn't declare any parameters, and the final parameter would be a [bodystmt](#bodystmt) containing the [vcall](#vcall) for `result`.

The handler for this event accepts the identifier (either [ident](#ident), [const](#const), [op](#op) or [kw](#kw)), a [params](#params) node (or a [paren](#paren) node containing the [params](#params) node if parentheses are used in the declaration), and finally a [bodystmt](#bodystmt) node.

```ruby
def on_def(ident, params, body); end
```

### `defined`

`defined` is a parser event that represents the use of the rather unique `defined?` operator. It can be used with and without parentheses.

```ruby
defined?(variable)
```

The handler for this event accepts a single parameter that represents whatever Ruby expression is being passed to the operator.

```ruby
def on_defined(value); end
```

### `defs`

`defs` is a parser event that represents defining a singleton method on an object. For example:

```ruby
def object.method(param) do result end
```

It accepts the same arguments as the [def](#def) event, as well as the target and operator on which this method is being defined. So in the example above, the parameters would be the [vcall](#vcall) for `object` as the target, a [period](#period) node for the `operator`, and then the same parameters as the [def](#def) event above.

```ruby
def on_defs(target, operator, ident, params, body); end
```

### `do_block`

`do_block` is a parser event that represents passing a block to a method call using the `do` and `end` keywords.

```ruby
method do |value|
end
```

The handler for this event accepts as arguments an optional [block_var](#block_var) event that represents any parameters to the block as well as a [bodystmt](#bodystmt) event that represents the statements inside the block.

```ruby
def on_do_block(block_var, bodystmt); end
```

### `dot2`

`dot2` is a parser event that represents using the `..` operator between two expressions. Usually this is to create a range object.

```ruby
1..2
```

Sometimes this operator is used to create a flip-flop.

```ruby
if value == 5 .. value == 10
end
```

The handler for this event accepts two parameters representing the left and right side of the `..` operator. Note that either one of them could be `nil`, but not both.

```ruby
def on_dot2(left, right); end
```

### `dot3`

`dot3` is a parser event that represents using the `...` operator between two expressions. It's effectively the same event as the [dot2](#dot2) event but with this operator you're asking Ruby to omit the final value.

```ruby
1...2
```

Like [dot2](#dot2) it can also be used to create a flip-flop.

```ruby
if value == 5 ... value == 10
end
```

The handler for this event accepts two parameters representing the left and right side of the `...` operator. Note that either one of them could be `nil`, but not both.

```ruby
def on_dot3(left, right); end
```

### `dyna_symbol`

`dyna_symbol` is a parser event that represents a symbol literal that uses quotes, be them double quotes (with interpolation) or single quotes (without interpolation). When not using quotes, the event is a [symbol_literal](#symbol_literal) instead. For example:

```ruby
:'some-string'
:"some-#{variable}"
```

They will also be used as a special kind of dynamic hash key, as in:

```ruby
{ "#{key}": value, 'other-key': other_value }
```

The handler for this event accepts one parameter which is either a [string_content](#string_content) node (representing an empty string, as in `:""` or `:''`) or a [string_add](#string_add) node (representing a non-empty string, as in the examples).

```ruby
def on_dyna_symbol(contents); end
```

### `else`

`else` is a parser event that represents the end of a `if` or `unless` chain.

```ruby
if variable
else
end
```

The handler for this event accepts a single [stmts_add](#stmts_add) node that represents the list of statements inside the `else` clause.

```ruby
def on_else(stmts_add); end
```

### `elsif`

`elsif` is a parser event that represents another clause in an `if` or `unless` chain.

```ruby
if variable
elsif other_variable
end
```

The handler for this event accepts the predicate to the `elsif` operator (any Ruby expression), the statements inside the clause (a [stmts_add](#stmts_add) node) and an optional consequent clause (`elsif` or [else](#else)).

```ruby
def on_elsif(predicate, stmts_add, consequent); end
```

### `embdoc`

`embdoc` is a scanner event that gets dispatched when the parser is inside a multi-line comment and receive a new line of content. For example, in the following multi-line comment, the `embdoc` event would be dispatched twice:

```ruby
=begin
first line
second line
=end
```

The handler for this event accepts a single string parameter containing the value of the line of content. Note that this includes the trailing newline.

```ruby
def on_embdoc(value); end
```

### `embdoc_beg`

`embdoc_beg` is a scanner event that represents the beginning of a multi-line comment.

```ruby
=begin
comment
=end
```

In the above example, it represents the `=begin` string. The handler for this event accepts a single string parameter that always contains `"=begin\n"` (provided you're using `\n` newlines).

```ruby
def on_embdoc_beg(value); end
```

### `embdoc_end`

`embdoc_end` is a scanner event that represents the ending of a multi-line comment.

```ruby
=begin
comment
=end
```

In the above example, it represents the `=end` string. The handler for this event accepts a single string parameter that always contains `"=end\n"` (provided you're using `\n` newlines).

```ruby
def on_embdoc_end(value); end
```

### `embexpr_beg`

`embexpr_beg` is a scanner event that represents the beginning token for using interpolation inside of a parent node that accepts string content (like a string or regular expression).

```ruby
"Hello, #{person}!"
```

The handler for this event accepts a single string parameter that always contains the string literal `"#{"`.

```ruby
def on_embexpr_beg(value); end
```

### `embexpr_end`

`embexpr_end` is a scanner event that represents the ending token for using interpolation inside of a parent node that accepts string content (like a string or regular expression).

```ruby
"Hello, #{person}!"
```

The handler for this event accepts a single string parameter that always contains the string literal `"}"`.

```ruby
def on_embexpr_end(value); end
```

### `embvar`

`embvar` is a scanner event that represents the use of shorthand interpolation for an instance, class, or global variable into a parent node that accepts string content (like a string or regular expression).

```ruby
"#@variable"
```

In the example above, the `#` would be triggering the `embvar` event because it forces `@variable` to be interpolated. The handler for this event accepts a single string parameter that is always the string literal `"#"`.

```ruby
def on_embvar(value); end
```

Note that changing the return value of this method does not impact other event handlers since the result never gets passed up the tree.

### `ensure`

`ensure` is a parser event that represents the use of the `ensure` keyword and its subsequent statements.

```ruby
begin
ensure
end
```

The handler for this event accepts a single [stmts_add](#stmts_add) event that represents the statements inside the `ensure` clause.

```ruby
def on_ensure(stmts_add); end
```

### `excessed_comma`

`excessed_comma` is a parser event that represents a trailing comma in a list of block parameters. It changes the block parameters such that they will destructure.

```ruby
[[1, 2, 3], [2, 3, 4]].each do |first, second,|
end
```

In the above example, an `excessed_comma` node would appear in the third position of the [params](#params) node that is used to declare that block. The third position typically represents a `rest`-type parameter, but in this case is used to indicate that a trailing comma was used. The handler for this event accepts no parameters (though in previous versions of Ruby it accepted a string literal with a value of `","`).

```ruby
def on_excessed_comma; end
```

### `fcall`

`fcall` is a parser event that represents the piece of a method call that comes before any arguments (i.e., just the name of the method). It is used in places where the parser is sure that it _is_ a method call and not potentially a local variable.

```ruby
method(argument)
```

In the above example, it's referring to the `method` segment. The handler for this event accepts a single [ident](#ident) or [const](#const) parameter that represents the name of the message being sent.

```ruby
def on_fcall(message); end
```

### `field`

`field` is a parser event that is always the child of an assignment. It represents assigning to a "field" on an object, as in:

```ruby
object.variable = value
```

The handler for this event accepts three parameters. The first is everything to the left of the operator (the object owning the field, can be any Ruby expression), the second is the operator itself (a [period](#period) for `.`, an [op](#op) for `&.`, or the symbol literal `:"::"` for `::`), and the third is everything to the right of the operator (the name of the field being assigned, either a [const](#const) or [ident](#ident) node).

```ruby
def on_field(left, operator, right); end
```

### `float`

`float` is a scanner event that represents a floating point value literal.

```ruby
1.0
```

The handler for this event accepts a single string parameter that represents the float's value in string form.

```ruby
def on_float(value); end
```

### `fndptn`

`fndptn` is a parser event that represents matching against a pattern where you find a pattern in an array using the Ruby 3.0+ pattern matching syntax.

```ruby
case value
in [*, 7, *]
end
```

In the above example, the syntax means it will match an array that contains a `7` integer literal.

The handler for this event accepts four parameters. The first is an optional constant wrapper like [aryptn](#aryptn). The second is a [var_field](#var_field) that represents the first `*` operator and its optional name. The third is any array of Ruby expressions that represent all of the values in the array between the two `*` operators. The fourth and final parameter is another [var_field](#var_field) that represents the second `*` operator and its optional name.

```ruby
def on_fndptn(const, presplat, values, postsplat); end
```

### `for`

`for` is a parser event that represents using a for loop.

```ruby
for value in list do
end
```

The handler for this event accepts three parameters. The first represents the list of iteration parameters. It can be an [mlhs_add](#mlhs_add), an [mlhs_add_star](#mlhs_add_star), or a [var_field](#var_field) node, depending on the kind of iteration. The second is the Ruby expression that represents the value being iterated. The third and final parameter is a [stmts_add](#stmts_add) node that represents the list of statements inside the for loop.

```ruby
def on_for(iterator, enumerable, stmts_add); end
```

### `gvar`

`gvar` is a scanner event that represents a global variable literal.

```ruby
$variable
```

The handler for this event accepts a single string value that represents the variable including the `$` prefix.

```ruby
def on_gvar(value); end
```

### `hash`

`hash` is a parser event that represents a hash literal.

```ruby
{ key => value }
```

The handler for this event accepts a single optional [assoclist_from_args](#assoclist_from_args) parameter that represents the contents of the hash literal. If the hash literal is empty, this parameter will be `nil`.

```ruby
def on_hash(assoclist_from_args); end
```

### `heredoc_beg`

`heredoc_beg` is a scanner event that represents the beginning of the heredoc.

```ruby
<<~DOC
  contents
DOC
```

In the above example, a `heredoc_beg` would get dispatched containing the `"<<~DOC"` string literal. The handler for this event accepts a single string parameter that represents the beginning declaration of the heredoc.

```ruby
def on_heredoc_beg(value); end
```

### `heredoc_dedent`

`heredoc_dedent` is a parser event that occurs when you're using a heredoc with a tilde (otherwise known as a "squiggly" heredoc).

```ruby
<<~DOC
  contents
DOC
```

This event will get dispatched once when the `heredoc` is closed. The handler for this event accepts two parameters. The first is the content of the heredoc, represented with a [string_add](#string_add) node. The second is an integer that represents the number of characters that should be stripped off from the beginning of each line of the heredoc (because of the semantics of the tilde in the declaration).

```ruby
def on_heredoc_dedent(string_add, width); end
```

### `heredoc_end`

`heredoc_end` is a scanner event that represents the end of the heredoc literal.

```ruby
<<~DOC
  contents
DOC
```

The handler for this event accepts a single string parameter that represents the end of the heredoc declaration. In the above example it would be the string literal `"DOC\n"` (provided you're using `\n` newlines).

```ruby
def on_heredoc_end(value); end
```

### `hshptn`

`hshptn` is a parser event that represents matching against a hash pattern using the Ruby 2.7+ pattern matching syntax.

```ruby
case value
in { key: }
end
```

The handler for this event accepts three parameters. The first is an optional constant wrapper, like [aryptn](#aryptn) and [fndptn](#fndptn). The second is an array of pairs of [label](#label) and an optional Ruby expression node for the value. Those pairs represent the key-value pairs used in the pattern matching. In the example above, it would contain a single pair of the `key` label and `nil` for the value. The third and final parameter is a [var_field](#var_field) that represents the optional double splat for grabbing the remaining keys.

```ruby
def on_hshptn(const, pairs, kwrest); end
```

### `ident`

`ident` is a scanner event that represents an identifier anywhere in code. It can represent a very large number of things, depending on where it is in the syntax tree.

```ruby
value
```

The handler for this event accepts a single string parameter representing the value in the source.

```ruby
def on_ident(value); end
```

### `if`

`if` is a parser event that represents the first clause in an `if` chain.

```ruby
if predicate
end
```

The handler for this event accepts three parameters. The first is any Ruby expression that represents the predicate used in the `if` clause. The second is a [stmts_add](#stmts_add) node that represent the statements inside the clause. The third and final parameter is an optional consequent clause the follows the `if` which can be an [elsif](#elsif) or [else](#else) clause.

```ruby
def on_if(predicate, stmts_add, consequent); end
```

### `ifop`

`ifop` is a parser event that represents a ternary clause.

```ruby
predicate ? truthy : falsy
```

The handler for this event accepts three parameters. The first is the predicate used in the ternary. The second is the value used in the truthy case. The third is the value used in the falsy case. All three parameters can be any Ruby expression.

```ruby
def on_ifop(predicate, truthy, falsy); end
```

### `if_mod`

`if_mod` is a parser event that represents the modifier form of an `if` statement.

```ruby
value if predicate
```

The handler for this event accepts two parameters. The first is the predicate for the `if` clause. The second is the statement being executed if the predicate is truthy. Both of the parameters can be any Ruby expression.

```ruby
def on_if_mod(predicate, statement); end
```

### `ignored_nl`

`ignored_nl` is a scanner event that represents a newline in the middle of a statement where it should be ignored. For example:

```ruby
object.first
      .second
```

The `ignored_nl` event will get triggered on the newline between the first and second method calls. The handler for this event accepts a single parameter that is always `nil`.

```ruby
def on_ignored_nl(value); end
```

### `ignored_sp`

`ignored_sp` is a scanner event that represents the space before the content of each line of a squiggly heredoc that will be removed from the string before it gets transformed into a string literal. For example:

```ruby
<<~DOC
  line1
    line2
DOC
```

In the above snippet, two `ignored_sp` event would be dispatched. The first would be dispatched with two spaces and the second with four.

Notice that this event is not a "native" `Ripper` event: it is produced only by `Ripper::Filter`, though the internal `Ripper` parser `Ripper::Lexer`, which generates them when handling [heredoc_dedent](#heredoc_dedent) events.

```ruby
def on_ignored_sp(value); end
```

### `imaginary`

`imaginary` is a scanner event that represents an imaginary number literal.

```ruby
1i
```

The handler for this event accepts a single string parameter that represents the value as seen in the source.

```ruby
def on_imaginary(value); end
```

### `in`

`in` is a parser event that represents using the `in` keyword within the Ruby 2.7+ pattern matching syntax.

```ruby
case value
in pattern
end
```

Alternatively, in Ruby 3+ it is also used to handle rightward assignment for pattern matching.

```ruby
value in pattern
```

Or, if you're using rightward assignment without pattern matching:

```ruby
expression => value
```

The handler for this event accepts three parameters. The first is the pattern that is being matched, which can be any Ruby expression. The second is a [stmts_add](#stmts_add) node that represents the statements inside the `in` clause. The final is an optional consequent clause that follows this `in` clause, which can be either an `in` or [else](#else).

Note that for a single-line rightward pattern matching like the second example or single-line rightward assignment like the third example, both the second and third parameters to this event handler will be `nil`.

```ruby
def on_in(pattern, stmts_add, consequent); end
```

### `int`

`int` is a scanner event the represents a number literal.

```ruby
1
```

The handler for this event accepts a single string parameter that represents the value as seen in the source.

```ruby
def on_int(value); end
```

### `ivar`

`ivar` is a scanner event the represents an instance variable literal.

```ruby
@variable
```

The handler for this event accepts a single string parameter that represents the value as seen in the source, including the `@` prefix.

```ruby
def on_ivar(value); end
```

### `kw`

`kw` is a scanner event the represents the use of a keyword. It can be almost anywhere in the syntax tree, so you end up seeing it quite a lot.

```ruby
if value
end
```

In the above example, `kw` would be dispatched twice: once for the `if` and once for the `end`. Note that anything that matches the list of keywords in Ruby will dispatch this event, so if you use a keyword in a symbol literal for instance:

```ruby
:if
```

then the contents of the [symbol](#symbol) node will contain a `kw` node. The handler for this event accepts a single string parameter that represents the value as seen in the source.

```ruby
def on_kw(value); end
```

### `kwrest_param`

`kwrest_param` is a parser event that represents defining a parameter in a method definition that accepts all remaining keyword parameters.

```ruby
def method(**kwargs); end
```

The handler for this event accepts a single optional [ident](#ident) node that represents the name of the parameter.

```ruby
def on_kwrest_param(ident); end
```

### `label`

`label` is a scanner event that represents the use of an identifier to associate with an object. You can find it in a hash key, as in:

```ruby
{ key: value }
```

In this case `"key:"` would be the body of the label. You can also find it in pattern matching, as in:

```ruby
case value
in key:
end
```

In this case `"key:"` would be the body of the label.

The handler for this event accepts a single string parameter that represents the value as seen in the source, including the `:` suffix.

```ruby
def on_label(value); end
```

### `label_end`

`label_end` is a scanner event that represents the end of a dynamic symbol. If for example you had the following hash:

```ruby
{ "key": value }
```

then the string `"\":"` would be the value of this `label_end`. The handler for this event accepts a single string parameter that represents the value as seen in the source. It's particularly useful for determining the type of quote being used by the label.

```ruby
def on_label_end(value); end
```

### `lambda`

`lambda` is a parser event that represents using a lambda literal (_not_ the `lambda` method call).

```ruby
->(value) { value * 2 }
```

The handler for this event accepts parameters a [params](#params) node (which can be a [paren](#paren) node if parentheses are used) that represents any parameters to the lambda and a statements node ([stmts_add](#stmts_add) for a lambda using braces and [bodystmt](#bodystmt) for a lambda using the `do` and `end` keywords) that represents the statements inside the lambda.

```ruby
def on_lambda(params, stmts); end
```

### `lbrace`

`lbrace` is a scanner event representing the use of a left brace, i.e., `{`.

The handler for this event accepts a single string parameter that is always `"{"`.

```ruby
def on_lbrace(value); end
```

### `lbracket`

`lbracket` is a scanner event representing the use of a left bracket, i.e., `[`.

The handler for this event accepts a single string parameter that is always `"["`.

```ruby
def on_lbracket(value); end
```

### `lparen`

`lparen` is a scanner event representing the use of a left parenthesis, i.e., `(`.

The handler for this event accepts a single string parameter that is always `"("`.

```ruby
def on_lparen(value); end
```

### `magic_comment`

`magic_comment` is a scanner event that represents the use of a pragma at the beginning of the file. Usually it will include something like
`frozen_string_literal` (the key) with a value of `true` (the value). It can also take multiple other forms though if using the `-*-` emacs-style file pragmas. Note that it's not just known values that dispatch this event, anything that matches a specific pattern will as well.

```ruby
# frozen_string_literal: true
```

The handler for this event accepts two parameters, the key and value. They both come in as string literals.

```ruby
def on_magic_comment(key, value); end
```

Note that the return value of this method will be passed immediately up into the [comment](#comment) event handler. So it is possible to skip this handler definition entirely and just process it in the comments handler.

### `massign`

`massign` is a parser event that is a parent node of any kind of multiple assignment. This includes splitting out variables on the left like:

```ruby
first, second, third = value
```

as well as splitting out variables on the right, as in:

```ruby
value = first, second, third
```

Both sides support splats, as well as variables following them. There's also destructuring behavior that you can achieve with the following:

```ruby
first, = value
```

In this case a would receive only the first value of the `value` enumerable. The handler for this event accepts two parameters representing the left and right side of the `=` operator. The left-hand side will be any of [mlhs_add](#mlhs_add), [mlhs_add_post](#mlhs_add_post), [mlhs_add_star](#mlhs_add_star), or [mlhs_paren](#mlhs_paren) depending on how it is declared. The right-hand side will be any Ruby expression.

```ruby
def on_massign(left, right); end
```

### `method_add_arg`

`method_add_arg` is a parser event that represents a method call with arguments and parentheses.

```ruby
method(argument)
```

The handler for this event accepts two parameters. The first represents the method being called, the second represents the arguments being passed to the method. In the example above, those would be [fcall](#fcall) and [arg_paren](#arg_paren) nodes, respectively.

You can also dispatch a `method_add_arg` event with a method on an object, as in:

```ruby
object.method(argument)
```

In this case the first parameter would be a [call](#call) node. Finally, you can dispatch a `method_add_arg` event when you are calling a method with no receiver that ends in a `?`. In this case, the parser knows it's a method call and not a local variable, so it dispatches a `method_add_arg` event as opposed to a [vcall](#vcall) event, as in:

```ruby
method?
```

In that case, the second parameter would be an [args_new](#args_new) event.

```ruby
def on_method_add_arg(method, args); end
```

### `method_add_block`

`method_add_block` is a parser event that represents a method call with a block argument.

```ruby
method {}
```

The handler for this event accepts two parameters, the method being called and the block being passed. The first can be a lot of different kinds of nodes, depending on how the method is being called. That parameter can be:

* [fcall](#fcall) if there is no explicit receiver (`method {}`)
* [call](#call) if there is an explicit receiver (`object.method {}`)
* [method_add_arg](#method_add_arg) if there are arguments in parentheses (`method(value) {}`)
* [command](#command) if there is no explicit receiver, it's a [do_block](#do_block) for the block, and there are arguments (`method value do end`)
* [command_call](#command_call) if there is an explicit receiver, it's a [do_block](#do_block) for the block, and there are arguments (`object.method value do end`)
* [super](#super) or [zsuper](#zsuper) in case the super method accepts a block (`super(args) {}` or `super do ... end`)
* [break](#break), [next](#next), [return](#return) in case of convoluted statements like:
  ```ruby
  break obj.method :value do |x| x end
  ```

The second parameter will always be a [brace_block](#brace_block) or a [do_block](#do_block) node.

```ruby
def on_method_add_block(method, block); end
```

### `mlhs_add`

`mlhs_add` is a parser event that represents adding another variable onto a list of variables within a multiple assignment.

```ruby
first, second, third = value
```

In the above example, three `mlhs_add` events would be dispatched. The handler for this event accepts two parameters. The first is either the result of a previous call to `mlhs_add` or the result of the [mlhs_new](#mlhs_new) event handler. The second parameter is the variable that is being assigned, which can be one of:

* [aref_field](#aref_field) if assigning into an enumerable (`collection[index]`)
* [field](#field) if assigning to a field on an object (`object.field`)
* [mlhs_paren](#mlhs_paren) if assigning into a destructured object with parentheses (`(first, second)`)
* [var_field](#var_field) if assigning to a local variable (`variable`)

```ruby
def on_mlhs_add(mlhs, part); end
```

### `mlhs_new`

`mlhs_new` is a parser event that represents the beginning of the left side of a multiple assignment. It is followed by any number of [mlhs_add](#mlhs_add) nodes that each represent another variable being assigned.

```ruby
first, second, third = value
```

In the above example, the `mlhs_new` event would be dispatched just before the `first` variable is declared. The handler for this event accepts no parameters as it is effectively the start of a list.

```ruby
def on_mlhs_new; end
```

### `mlhs_add_post`

`mlhs_add_post` is a parser event that represents adding another set of variables onto a list of assignments after a splat variable within a multiple assignment.

```ruby
left, *middle, right = values
```

In the example above, an `mlhs_add_post` event would be dispatched when the parser encountered the `right` token. The handler for this event accepts two parameters. The first is the [mlhs_add_star](#mlhs_add_star) the contains everything leading up to the splatted variable. The second is the [mlhs_add](#mlhs_add) that represents every variable after that splatted variable.

```ruby
def on_mlhs_add_post(mlhs_add_star, mlhs_add); end
```

### `mlhs_add_star`

`mlhs_add_star` is a parser event that represents a splatted variable inside of a multiple assignment on the left hand side.

```ruby
first, *rest = values
```

The handler for this event accepts two parameters. The first is the result of the first call to either [mlhs_new](#mlhs_new) (if the splatted variable is the first in the list) or [mlhs_add](#mlhs_add) (if the splatted variable is not the first in the list). The second is the node that represents the value being splatted, which can be many different nodes (usually a field node, but it depends on the expression type).

```ruby
def on_mlhs_add_star(mlhs, part); end
```

### `mlhs_paren`

`mlhs_paren` is a parser event that represents parentheses being used to destruct values in a multiple assignment on the left hand side.

```ruby
(left, right) = value
```

The handler for this event accepts one parameter which represents the value contained within the parentheses. Depending on the declaration, it can be an [mlhs_add](#mlhs_add), [mlhs_add_post](#mlhs_add_post), [mlhs_add_star](#mlhs_add_star), or a nested `mlhs_paren`.

```ruby
def on_mlhs_paren(contents); end
```

### `module`

`module` is a parser event that represents defining a module using the `module` keyword.

```ruby
module Namespace
end
```

The handler for this event accepts one parameter for the name of the module (a [const](#const) node) and one parameter for the statements inside the module (a [bodystmt](#bodystmt) node).

```ruby
def on_module(const, bodystmt); end
```

### `mrhs_add`

`mrhs_add` is a parser event that represents adding a last, non-splat value onto a list on the right hand side of a multiple assignment, after an [mrhs_new_from_args](#mrhs_new_from_args).

```ruby
values = first, second, third
```

In the example above, one `mrhs_add` event would be dispatched for the identifier `third`. The handler for this event accepts the result of the handler for [mrhs_new_from_args](#mrhs_new_from_args), as well as the part that is being added which can be any Ruby expression.

```ruby
def on_mrhs_add(mrhs_new_from_args, part); end
```

### `mrhs_add_star`

`mrhs_add_star` is a parser event that represents using the splat operator to expand out the value of the last argument on the right hand side of a multiple assignment.

```ruby
values = first, *rest
```

The handler for this event accepts two parameters. The first is either an [mrhs_new](#mrhs_new) (in the case that the splat is the only value being assigned) or an [mrhs_new_from_args](#mrhs_new_from_args) (in the case that multiple values are being assigned, as in the example above). The second is the part that is being splatted into the list, which can be any Ruby expression.

```ruby
def on_mrhs_add_star(mrhs, part); end
```

### `mrhs_new`

`mrhs_new` is a parser event that represents the assignment of a splat to a value. It will be followed by an [mrhs_add_star](#mrhs_add_star) node.

```ruby
values = *argument
```

In the example above, an `mrhs_new` event would be dispatched when the parser hits `*argument`. The next call will be a [mrhs_add_star](#mrhs_add_star) for `argument`. The handler for this event accepts no parameters as it represents the beginning of a list.

```ruby
def on_mrhs_new; end
```

### `mrhs_new_from_args`

`mrhs_new_from_args` is a parser event that represents the shorthand of a multiple assignment that allows you to assign values using just commas as opposed to assigning from an array. For example, in the following segment the right hand side of the assignment would dispatch this event:

```ruby
values = first, second, third
```

In the example above, the sequence of event calls will be:
```
mrhs_add(
  mrhs_new_from_args(
    args_add(
      args_add(
        args_new(),
        vcall(ident("first"))
      ),
      vcall(ident("second"))
    )
  ),
  vcall(ident("third")
)
```

The event is therefore dispatched once, before the last right-hand side argument. The handler for this event accepts a single [args_add](#args_add) or [args_add_star](#args_add_star) node that represents the values collected before the last right-hand side argument, which will be added by the following [mrhs_add](#mrhs_add) or [mrhs_add_star](#mrhs_add_star) event.

```ruby
def on_mrhs_new_from_args(args); end
```

### `next`

`next` is a parser event that represents using the `next` keyword.

```ruby
next
```

The `next` keyword can also optionally be called with an argument:

```ruby
next value
```

`next` can even be called with multiple arguments, but only if parentheses are omitted, as in:

```ruby
next first, second, third
```

If a single value is being given, parentheses can be used, as in:

```ruby
next(value)
```

The handler for this event accepts a single parameter that represents the type of values being passed to the `next` keyword. In the case that no arguments are given, it's an [args_new](#args_new) node. If one or more arguments are given, it's an [args_add_block](#args_add_block) node.

```ruby
def on_next(args); end
```

### `nl`

`nl` is a scanner event representing a newline in the source. As you can imagine, it gets dispatched quite often, so take care if you're defining this method manually. 

The handler for this event accepts a single string parameter that always contains the value of the newline.

```ruby
def on_nl(value); end
```

### `nokw_param`

`nokw_param` is a parser event that represents the use of the Ruby 2.7+ syntax to indicate a method should take no additional keyword arguments. For example in the following snippet:

```ruby
def method(**nil) end
```

This example is saying that the method `method` should not accept any keyword arguments. The handler for this event accepts a single parameter that is always the value `nil`.

```ruby
def on_nokw_param(value); end
```

The result of this event handler will get passed up to the event handler for the [params](#params) node that it is a part of in the `kwrest` position.

### `op`

`op` is a scanner event representing an operator literal in the source. For example, in the following snippet:

```ruby
1 + 2
```

In the example above, the `+` operator would dispatch this event. The handler for this event accepts a single string parameter that represents the operator as seen in the source.

```ruby
def on_op(value); end
```

### `opassign`

`opassign` is a parser event that represents assigning a value to a variable or constant using an operator like `+=` or `||=`.

```ruby
variable += value
```

The handler for this event accepts three parameters. The first is the left side of the operator, which can be any kind of field node. The second is the [op](#op) node that represents the operator being used. The third is the right side of the operator, which can be any Ruby expression.

```ruby
def on_opassign(left, operator, right); end
```

### `operator_ambiguous`

`operator_ambiguous` is a parser event that represents when the parser sees an operator as ambiguous. For example, in the following snippet:

```ruby
method %[]
```

the question becomes if the percent sign is being used as a method call or if it's the start of a string literal. The handler for this event accepts two parameters. The first is a symbol that represents the operator that dispatched this event (in the case of this example it would be `:%`). The second is a string that is used to indicate the kind of ambiguity (these are defined explicitly in the parser, in the case of this example it would be `"string literal"`).

```ruby
def on_operator_ambiguous(operator, ambiguity); end
```

### `params`

`params` is a parser event that represents defining parameters on a method or lambda.

```ruby
def method(param) end
```

In the example above a `params` event would be dispatched when the parser found the `param` identifier. The handler for this event accepts seven parameters, each indicating the presence of a different type of parameter (so they can all be `nil`). They are, in order:

* Positional parameters (`req`) - an array of [ident](#ident) nodes
* Optional parameters (`opt`) - an array of pairs containing [ident](#ident) nodes for the name as well as a node representing whatever expression is being used for the default value
* Rest parameter (`rest`) - either an [args_forward](#args_forward) node (if argument forwarding is being used), an [excessed_comma](#excessed_comma) node (if a trailing comma is present to indicate destructuring), or a [rest_param](#rest_param) node to represent a splat
* Post parameters (`req`) - an array of [ident](#ident) nodes that represent the list of parameters occuring _after_ a rest parameter
* Keyword parameters (`keyreq` and `key`) - an array of pairs containing [label](#label) nodes for the name as well as a node representing whatever expresion is used for the default value (or `false` if none is provided)
* Keyword rest parameter (`keyrest` or `nokey`) - either a [kwrest_param](#kwrest_param) node if a double-splat operator is being used to gather up remaining keyword arguments or the symbol `:nil` if it's using the `**nil` syntax
* Block parameter (`block`) - a [blockarg](#blockarg) node

Note that the shorthands above come from calling the `Method#parameters` method.

```ruby
def on_params(req, opts, rest, post, keys, keyrest, block); end
```

### `paren`

`paren` is a parser event that represents using balanced parentheses in a couple places in a Ruby program. In general parentheses can be used anywhere a Ruby expression can be used.

```ruby
(1 + 2)
```

The handler for this event accepts a single parameter that represents the expression inside the parentheses.

```ruby
def on_paren(contents); end
```

### `period`

`period` is a scanner event that represents the use of the `.` operator. It is usually found in method calls.

The handler for this event accepts a single string parameter that always contains the value `"."`.

```ruby
def on_period(value); end
```

### `program`

`program` is a parser event that represents the overall syntax tree. It will always be called when a Ripper parser is used.

The handler for this event accepts a single [stmts_add](#stmts_add) node that represents the top-level statements of the source.

```ruby
def on_program(stmts_add); end
```

### `qsymbols_beg`

`qsymbols_beg` is a scanner event that represents the beginning of a symbol literal array. For example:

```ruby
%i[one two three]
```

In the snippet above, a `qsymbols_beg` event would be dispatched with the value of `"%i["`. The handler for this event accepts a single string parameter representing the token as seen in the source. Note that these kinds of arrays can start with a lot of different delimiter types (e.g., `%i|` or `%i<`).

```ruby
def on_qsymbols_beg(value); end
```

### `qsymbols_add`

`qsymbols_add` is a parser event that represents adding an element to a symbol literal array.

```ruby
%i[one two three]
```

In the example above, three `qsymbols_add` events would be dispatched. The first would be with the result of the [qsymbols_new](#qsymbols_new) event handler, the second with the result of the first `qsymbols_add` event handler call, and the third with the result of the second call.

The handler for this event accepts the current list of symbols (as a [qsymbols_new](#qsymbols_new) or [qsymbols_add](#qsymbols_add) node) and the next value that should be added (always a [tstring_content](#tstring_content) node).

```ruby
def on_qsymbols_add(qsymbols, tstring_content); end
```

### `qsymbols_new`

`qsymbols_new` is a parser event that represents the beginning of a symbol literal array.

```ruby
%i[one two three]
```

In the axample above, a `qsymbols_new` event would be dispatched when the parser finds the `one` token. It can be followed by any number of [qsymbols_add](#qsymbols_add) events. The handler for this event accepts no parameters, as it's the start of a list.

```ruby
def on_qsymbols_new; end
```

### `qwords_beg`

`qwords_beg` is a scanner event that represents the beginning of a string literal array. For example:

```ruby
%w[one two three]
```

In the snippet above, a `qwords_beg` event would be dispatched with the value of `"%w["`. The handler for this event accepts a single string parameter representing the token as seen in the source. Note that these kinds of arrays can start with a lot of different delimiter types (e.g., `%w|` or `%w<`).

```ruby
def on_qwords_beg(value); end
```

### `qwords_add`

`qwords_add` is a parser event that represents adding an element to a string literal array.

```ruby
%w[one two three]
```

In the example above, three `qwords_add` events would be dispatched. The first would be with the result of the [qwords_new](#qwords_new) event handler, the second with the result of the first `qwords_add` event handler call, and the third with the result of the second call.

The handler for this event accepts the current list of strings (as a [qwords_new](#qwords_new) or [qwords_add](#qwords_add) node) and the next value that should be added (always a [tstring_content](#tstring_content) node).

```ruby
def on_qwords_add(qwords, tstring_content); end
```

### `qwords_new`

`qwords_new` is a parser event that represents the beginning of a string literal array.

```ruby
%w[one two three]
```

In the axample above, a `qwords_new` event would be dispatched when the parser finds the `one` token. It can be followed by any number of [qwords_add](#qwords_add) events. The handler for this event accepts no parameters, as it's the start of a list.

```ruby
def on_qwords_new; end
```

### `rational`

`rational` is a scanner event that represents the use of a rational number literal.

```ruby
1r
```

The handler for this event accepts a single string parameter representing the value as seen in source.

```ruby
def on_rational(value); end
```

### `rbrace`

`rbrace` is a scanner event that represents the use of a right brace, i.e., `}`.

The handler for this event accepts a single string parameter that always contains the value `"}"`.

```ruby
def on_rbrace(value); end
```

### `rbracket`

`rbracket` is a scanner event that represents the use of a right bracket, i.e., `]`.

The handler for this event accepts a single string parameter that always contains the value `"]"`.

```ruby
def on_rbracket(value); end
```

### `redo`

`redo` is a parser event that represents the use of the `redo` keyword.

```ruby
redo
```

The handler for this event accepts no parameters as the `redo` keyword accepts no arguments.

```ruby
def on_redo; end
```

### `regexp_add`

`regexp_add` is a parser event that represents a piece of a regular expression body.

```ruby
/.+ #{pattern} .+/
```

In the example above, three `regexp_add` events would be dispatched. The first would be for the [tstring_content](#tstring_content) node representing everything up to the interpolation. The second would be for the interpolated expression. The final would be for the second [tstring_content](#tstring_content) node representing everything after the interpolation.

The handler for this event accepts two parameters. The first is the result of the call to the previous `regexp_add` (if this is not the first part of the regular expression) or the result of the call to [regexp_new](#regexp_new) (if this is the first part of the regular expression). The second is the piece that is being added to the list of parts, which can be a [tstring_content](#tstring_content) for plain string content, a [string_embexpr](#string_embexpr) for an interpolated expression, or a [string_dvar](#string_dvar) for the shorthand variable interpolation.

```ruby
def on_regexp_add(regexp, part); end
```

### `regexp_beg`

`regexp_beg` is a scanner event that represents the start of a regular expression literal.

```ruby
/.+/
```

In the example above, the `regexp_beg` event would be dispatched when the parser sees the `/` token. Regular expression literals can also be declared using the `%r` syntax, as in:

```ruby
%r{.+}
```

The handler for this event accepts a single string parameter that represents the start of the regular expression literal (either the `"/"` or the `"%r{"` in the examples above).

```ruby
def on_regexp_beg(value); end
```

### `regexp_end`

`regexp_end` is a scanner event that represents the end of a regular expression literal.

```ruby
/.+/m
```

In the example above, the `regexp_end` event represents the `/m` at the end of the regular expression literal. You can also declare regular expression literals using `%r`, as in:

```ruby
%r{.+}m
```

The handler for this event accepts a single string parameter that represents the end of the regular expression literal (either the `"/m"` or the `"}m"` in the examples above).

```ruby
def on_regexp_end(value); end
```

### `regexp_literal`

`regexp_literal` is a parser event that represents a regular expression literal.

```ruby
/.+/
```

The handler for this event accepts two parameters. The first is either a [regexp_new](#regexp_new) node (if the regular expression literal is empty) or [regexp_add](#regexp_add) (if there is content in the regular expression literal). The second is a [regexp_end](#regexp_end) node that represents the end of the declaration of the regular expression literal.

```ruby
def on_regexp_literal(regexp, ending); end
```

### `regexp_new`

`regexp_new` is a parser event that represents the beginning of a regular expression literal.

```ruby
/.+/
```

In the example above, a `regexp_new` event would be dispatched just before the `.+` [tstring_content](#tstring_content) node. The handler for this event accepts no parameters as it's the start of a list.

```ruby
def on_regexp_new; end
```

### `rescue`

`rescue` is a parser event that represents the use of the `rescue` keyword inside of a [bodystmt](#bodystmt) node.

```ruby
begin
rescue
end
```

You can optionally rescue a list of exceptions (either in the form of constants by referencing them directly or in the form of variables that reference exceptions), as in:

```ruby
begin
rescue exception
rescue IOError, SyntaxError
end
```

You can also assign the rescued error to a variable, as in:

```ruby
begin
rescue => error
end
```

Finally, you can rescue a specific error and then assign it to a variable, as in:

```ruby
begin
rescue IOError => error
end
```

The handler for this event accepts four parameters. The first is the list of exceptions that are being rescued. This is either `nil` (if there are no explicit exceptions being rescued), an array containing a single node (if only one exception is being rescued), a [mrhs_add](#mrhs_add) node (if multiple exceptions are being rescued), or an [mrhs_add_star](#mrhs_add_star) node (if multiple exceptions are being rescued and its using a splat). The second is an optional variable being used to capture the exception. The third is a [stmts_add](#stmts_add) node representing the statements inside the `rescue` clause. Finally, the fourth is an optional consequent `rescue` clause if one is present.

```ruby
def on_rescue(exceptions, variable, stmts_add, consequent); end
```

### `rescue_mod`

`rescue_mod` is a parser event that represents the using modifier form of a `rescue` clause.

```ruby
expression rescue value
```

The handler for this event accepts one parameter for the expression that is being rescued and one parameter for the value that should be used if an exception is raised. Both can be any Ruby expression.

```ruby
def on_rescue_mod(expression, rescue_value); end
```

### `rest_param`

`rest_param` is a parser event that represents defining a parameter in a method definition that accepts all remaining positional parameters.

```ruby
def method(*rest) end
```

The handler for this event accepts a single optional [ident](#ident) parameter that represents the name assigned to the rest parameter. If no name is provided (just a bare `*`), then this parameter will be `nil`.

```ruby
def on_rest_param(ident); end
```

### `retry`

`retry` is a parser event that represents the use of the `retry` keyword.

```ruby
retry
```

The handler for this event accepts no parameters as the `retry` keyword accepts no arguments.

```ruby
def on_retry; end
```

### `return`

`return` is a parser event that represents using the return keyword with arguments.

```ruby
return value
```

The handler for this event accepts a single [args_add_block](#args_add_block) event that contains all of the arguments being passed to the keyword.

```ruby
def on_return(args_add_block); end
```

### `return0`

`return0` is a parser event that represents the bare return keyword.

```ruby
return
```

The handler for this event accepts no parameters.

```ruby
def on_return0; end
```

### `rparen`

`rparen` is a scanner event that represents the use of a right parenthesis, i.e., `)`.

The handler for this event accepts a single string parameter that always contains the value `")"`.

```ruby
def on_rparen(value); end
```

### `sclass`

`sclass` is a parser event that represents a block of statements that should be evaluated within the context of the singleton class of an object. It's frequently used to define singleton methods. It looks like the following example:

```ruby
class << self
end
```

The handler for this event accepts two parameters. The first is the object whose singleton class the statements should be evaluated in. In the example above it would be the [var_ref](#var_ref) representing the `self` keyword. The second is the [bodystmt](#bodystmt) node that represents the statements inside the block.

```ruby
def on_sclass(object, bodystmt); end
```

### `semicolon`

`semicolon` is a scanner event that represents the use of a semicolon in the source.

```ruby
;
```

The handler for this event accepts a single string parameter that always contains the value `";"`.

```ruby
def on_semicolon(value); end
```

### `sp`

`sp` is a scanner event that represents the use of a space in the source. As you can imagine, this event gets triggered quite often, so take care when manually defining the event handler for this event.

The handler for this event accepts a single string parameter that always contains the value `" "`.

```ruby
def on_sp(value); end
```

### `stmts_add`

`stmts_add` is a parser event that represents a single statement inside a list of statements within any lexical block.

The handler for this event accepts two parameters. The first is the accumulated list of statements (either a [stmts_new](#stmts_new) if it's the first statement or a `stmts_add` if it's not the first statement). The second is the statement that we're adding to the list which can be any Ruby expression.

```ruby
def on_stmts_add(stmts, stmt); end
```

Lots of different nodes function as wrappers around lists of statements (`stmts_add`/[stmts_new](#stmts_new) nodes). For instance, an `if` or `else` block normally contains a list of statements, or the inside of a `while` loop, `rescue` clause, or `lambda`. Fortunately, they are all handled in the same way for each node type, which makes them simpler to work with if you're going to be iterating through statement lists (for instance in a formatter or a static analyzer). The nodes that wrap statements are:

* [BEGIN](#BEGIN)
* [END](#END)
* [bodystmt](#bodystmt)
* [brace_block](#brace_block)
* [else](#else)
* [elsif](#elsif)
* [ensure](#ensure)
* [for](#for)
* [if](#if)
* [in](#in)
* [lambda](#lambda)
* [program](#program)
* [rescue](#rescue)
* [string_embexpr](#string_embexpr)
* [unless](#unless)
* [until](#until)
* [when](#when)
* [while](#while)

### `stmts_new`

`stmts_new` is a parser event that represents the beginning of a list of statements within any lexical block. It can be followed by any number of `stmts_add` events.

The handler for this event accepts no parameters as it's the start of a list.

```ruby
def on_stmts_new; end
```

### `string_add`

`string_add` is a parser event that represents adding a section onto a string.

```ruby
"left #{middle} right"
```

In the example above, three `string_add` events would be dispatched. The first would be for the [tstring_content](#tstring_content) on the left side of the interpolation. The second would be for the [string_embexpr](#string_embexpr) representing the interpolation. The third would be for the [tstring_content](#tstring_content) on the right side of the interpolation.

The handler for this event accepts two parameters. The first is a [string_content](#string_content) if it's the first part of the string or a `string_add` if it's not the first part. The second is the part that is being added to the string, which can be a [tstring_content](#tstring_content) for plain string content, a [string_embexpr](#string_embexpr) for an interpolated expression, or a [string_dvar](#string_dvar) for a shorthand variable interpolation.

```ruby
def on_string_add(string, part); end
```

### `string_concat`

`string_concat` is a parser event that represents concatenating two strings together using a backward slash, as in the following example:

```ruby
"first" \
  "second"
```

The handler for this event accepts two parameters. The first is either a `string_concat` or a [string_literal](#string_literal). The second is always the trailing [string_literal](#string_literal).

```ruby
def on_string_concat(left, right); end
```

### `string_content`

`string_content` is a parser event that represents the beginning of the contents of a string.

```ruby
"string"
```

In the example above, a `string_content` event would be dispatched when the parser encounters the `string` token. The handler for this event accepts no parameters as its the beginning of a list.

```ruby
def on_string_content; end
```

### `string_dvar`

`string_dvar` is a parser event that represents shorthand interpolation of a variable into a string. It allows you to take an instance variable, class variable, or global variable and omit the braces when interpolating. For example, if you wanted to interpolate the instance variable `@variable` into a string, you would write:

```ruby
"#@variable"
```

The handler for this event accepts a single parameter that represents the variable being interpolated. This can either be a [var_ref](#var_ref) or a [backref](#backref).

```ruby
def on_string_dvar(variable); end
```

### `string_embexpr`

`string_embexpr` is a parser event that represents interpolated content. It can be contained within a couple of different parent nodes, including regular expressions, strings, and dynamic symbols.

```ruby
"string #{expression}"
```

The handler for this event accepts a single [stmts_add](#stmts_add) node representing the statements contained within the interpolation.

```ruby
def on_string_embexpr(stmts_add); end
```

### `string_literal`

`string_literal` is a parser event that represents a string literal.

```ruby
"string"
```

It is also used to represent a heredoc literal.

```ruby
<<~DOC
  string
DOC
```

The handler for this event accepts a single [string_content](#string_content) node (if the string is empty) or a [string_add](#string_add) node (if the string is not empty).

```ruby
def on_string_literal(string); end
```

### `super`

`super` is a parser event that represents using the `super` keyword with arguments. It can optionally use parentheses.

```ruby
super(value)
```

The handler for this event accepts a single parameter that represents the arguments given to the `super` keyword. It can be an [arg_paren](#arg_paren) (in parentheses are used) or an [args_add_block](#args_add_block) (if parentheses are not used).

```ruby
def on_super(args); end
```

### `symbeg`

`symbeg` is a scanner event that represents the beginning of a symbol literal.

```ruby
:symbol
```

`symbeg` will also get dispatched for dynamic symbols, as in:

```ruby
:"symbol"
```

Finally, `symbeg` will get dispatched for symbols using the `%s` syntax, as in:

```ruby
%s[symbol]
```

The handler for this event accepts a single string parameter. In most cases (as in the first example above) it will contain just `":"`. In the case of dynamic symbols it will contain `":'"` or `":\""`. In the case of `%s` symbols, it will contain the start of the symbol including the `%s` and the delimiter.

```ruby
def on_symbeg(value); end
```

### `symbol`

`symbol` is a parser event that immediately descends from a [symbol_literal](#symbol_literal).

```ruby
:symbol
```

The handler for this event accepts a single node that represents the contents of the symbol. In most cases this will be an [ident](#ident) node, but if the symbol matches other patterns it can be other scanner events (like [kw](#kw) for `:if` or [ivar](#ivar) for `:@ivar`).

```ruby
def on_symbol(contents); end
```

### `symbol_literal`

`symbol_literal` is a parser event that represents a symbol not using quotes (as opposed to a [dyna_symbol](#dyna_symbol)).

```ruby
:symbol
```

The handler for this event accepts a single parameter. In most cases it is a [symbol](#symbol) node. In rare cases, it's an [ident](#ident) node (where Ruby accepts bare words like `alias aliased_name name`).

```ruby
def on_symbol_literal(contents); end
```

### `symbols_add`

`symbols_add` is a parser event that represents adding an element to a symbol literal array with interpolation.

```ruby
%I[one two three]
```

In the example above, three `symbols_add` events would be dispatched. The first would be with the result of the [symbols_new](#symbols_new) event handler, the second with the result of the first `symbols_add` event handler call, and the third with the result of the second call.

The handler for this event accepts the current list of symbols (as a [symbols_new](#symbols_new) or [symbols_add](#symbols_add) node) and the next value that should be added (always a [word_add](#word_add) node).

```ruby
def on_symbols_add(qsymbols, word_add); end
```

### `symbols_beg`

`symbols_beg` is a scanner event that represents the start of a symbol literal array with interpolation. For example, in the following snippet:

```ruby
%I[one two three]
```

In the snippet above, a `symbols_beg` event would be dispatched with the value of `"%I["`. The handler for this event accepts a single string parameter representing the token as seen in the source. Note that these kinds of arrays can start with a lot of different delimiter types (e.g., `%I|` or `%I<`).

```ruby
def on_symbols_beg(value)
```

### `symbols_new`

`symbols_new` is a parser event that represents the beginning of a symbol literal array with interpolation.

```ruby
%I[one two three]
```

In the example above, a `symbols_new` event would be dispatched when the parser finds the `one` token. It can be followed by any number of [symbols_add](#symbols_add) events. The handler for this event accepts no parameters, as it's the start of a list.

```ruby
def on_symbols_new; end
```

### `tlambda`

`tlambda` is a scanner event that represents the beginning of a lambda literal.

```ruby
-> { value }
```

In the example above it represents the `->` operator. The handler for this event accepts a single string parameter that always has the value of `"->"`.

```ruby
def on_tlambda(value); end
```

### `tlambeg`

`tlambeg` is a scanner event that represents the beginning of the body of a lambda literal using braces.

```ruby
-> { value }
```

In the example above it represents the `{` operator. The handler for this event accepts a single string parameter that always has the value of `"{"`.

```ruby
def on_tlambeg(value); end
```

### `top_const_field`

`top_const_field` is a parser event that is always the child of some kind of assignment. It represents when you're assigning to a constant that is being referenced at the top level. For example:

```ruby
::Constant = value
```

The handler for this event accepts a single [const](#const) parameter that represents the name of the constant being assigned to.

```ruby
def on_top_const_field(const); end
```

### `top_const_ref`

`top_const_ref` is a parser event that is a very similar to [top_const_field](#top_const_field) except that it is not involved in an assignment. It looks like the following example:

```ruby
::Constant
```

The handler for this event accepts a single [const](#const) parameter that represents the name of the constant being referenced.

```ruby
def on_top_const_ref(const); end
```

### `tstring_beg`

`tstring_beg` is a scanner event that represents the beginning of a string literal.

```ruby
"string"
```

In the example above, a `tstring_beg` event would be dispatched when the parser encounters the `"` token. Strings can also use single quotes. They can also be declared using the `%q` and `%Q` syntax, as in:

```ruby
%q{string}
```

The handler for this event accepts a single string parameter that represents the start of the string.

```ruby
def on_tstring_beg(value); end
```

### `tstring_content`

`tstring_content` is a scanner event that represents plain characters inside of an entity that accepts string content like a string, heredoc, command string, or regular expression.

```ruby
"string"
```

In the example above, a `tstring_content` event would be dispatched for the `string` token contained within the string. The handler for this event accepts a single string parameter containing the value as seen in the source.

```ruby
def on_tstring_content(value); end
```

### `tstring_end`

`tstring_end` is a scanner event that represents the end of a string literal.

```ruby
"string"
```

In the example above, a `tstring_end` event would be dispatched when the parser encountered the second set of quotes. Strings can also use single quotes. They can also be declared using the `%q` and `%Q` syntax, as in:

```ruby
%q{string}
```

The handler for this event accepts a single string parameter representing the end of the string literal.

```ruby
def on_tstring_end(value); end
```

### `unary`

`unary` is a parser event that represents a unary method being called on an expression, as in `!`, `~`, or `not`.

```ruby
!value
```

The handler for this event accepts a parameter for the operator being used (in the form of a symbol) and a parameter for the value which can be almost any Ruby expression.

```ruby
def on_unary(operator, value); end
```

### `undef`

`undef` is a parser event that represents the use of the `undef` keyword.

```ruby
undef method
```

The handler for this event accepts a single array parameter that represents all of the names of the methods to be undefined. The names of the methods can be [symbol_literal](#symbol_literal) nodes (in the case of actual symbol literals or bare words) or they can be [dyna_symbol](#dyna_symbol) nodes (in the case of dynamic symbols).

```ruby
def on_undef(methods); end
```

### `unless`

`unless` is a parser event that represents an `unless` statement.

```ruby
unless predicate
end
```

The handler for this event accepts three parameters. The first is the predicate to the `unless` clause which can be any Ruby expression. The second is the list of statements under `unless` (a [stmts_add](#stmts_add) node), executed when the predicate has a false value. The third is an optional [else](#else) node, executed if the predicate has a true value.

```ruby
def on_unless(predicate, stmts_add, if_true); end
```

### `unless_mod`

`unless_mod` is a parser event that represents the modifier form of an `unless` statement.

```ruby
value unless predicate
```

The handler for this event accepts two parameters. The first is the predicate being used and the second is the statement to be executed. Both parameters can be any Ruby expression.

```ruby
def on_unless_mod(predicate, statement); end
```

### `until`

`until` is a parser event that represents an `until` loop.

```ruby
until predicate
end
```

The handler for this event accepts two parameters. The first is the predicate for the `until` clause which can be any Ruby expression. The second is the list of statements inside the loop which is represented by a [stmts_add](#stmts_add) node.

```ruby
def on_until(predicate, stmts_add); end
```

### `until_mod`

`until_mod` is a parser event that represents the modifier form of an `until` loop.

```ruby
expression until predicate
```

The handler for this event accepts a parameter for the predicate to the `until` clause and a parameter for the statement to be executed. Both can be any Ruby expression.

```ruby
def on_until_mod(predicate, statement); end
```

### `var_alias`

`var_alias` is a parser event that represents when you're using the `alias` keyword with global variable arguments.

```ruby
alias $new $old
```

The handler for this event accepts two parameters. The first is always a [gvar](#gvar) node which represents the first argument to the `alias` keyword. The second is either a [gvar](#gvar) node if a previously-defined global variable is being aliased or a [backref](#backref) node if a back reference is being aliased.

```ruby
def on_var_alias(left, right); end
```

### `var_field`

`var_field` is a parser event that represents a variable that is being assigned a value. As such, it is always a child of an assignment type node. For example:

```ruby
variable = value
```

In the example above, the `var_field` event represents the `variable` token. The handler for this event accepts a single [ident](#ident) node that represents the name of the variable being assigned to.

```ruby
def on_var_field(ident); end
```

Note that there are a few cases where the ident can be omitted, as in the case that you're using a single splat operator without a name.

### `var_ref`

`var_ref` is a parser event that represents a variable reference.

```ruby
variable
```

This can be a plain local variable like the example above. It can also be a constant, class variable, global variable, instance variable, keyword (like `self`, `nil`, `true`, or `false`), or numbered block variable.

The handler for this event accepts a single scanner event parameter representing the value as seen in source.

```ruby
def on_var_ref(contents); end
```

### `vcall`

`vcall` nodes are any plain named object with Ruby that could be either a local variable or a method call.

```ruby
variable
```

The handler for this event accepts a single [ident](#ident) node that represents the contents.

```ruby
def on_vcall(ident); end
```

### `void_stmt`

`void_stmt` is a special kind of parser event that represents an empty lexical block of code.

```ruby
;;
```

In the example above, there is a `void_stmt` between the two semicolons. The handler for this event accepts no parameters.

```ruby
def on_void_stmt; end
```

### `when`

`when` is a parser event that represents another clause in a `case` chain.

```ruby
case value
when predicate
end
```

The handler for this event accepts three parameters. The first is the predicate used in the `when` clause, which can be any Ruby expression. The second is the statements contained within the clause, represented by a [stmts_add](#stmts_add) node. The third is an optional consequent clause that follows this `when` clause, which can be an [else](#else) node or another `when` node.

```ruby
def on_when(predicate, stmts_add, consequent); end
```

### `while`

`while` is a parser event that represents a `while` loop.

```ruby
while predicate
end
```

The handler for this event accepts two parameters. The first is the predicate for the `while` clause which can be any Ruby expression. The second is the list of statements inside the loop which is represented by a [stmts_add](#stmts_add) node.

```ruby
def on_while(predicate, stmts_add); end
```

### `while_mod`

`while_mod` is a parser event that represents the modifier form of an `while` loop.

```ruby
expression while predicate
```

The handler for this event accepts a parameter for the predicate to the `while` clause and a parameter for the statement to be executed. Both can be any Ruby expression.

```ruby
def on_while_mod(predicate, statement); end
```

### `word_add`

`word_add` is a parser event that represents a piece of a word within a special array literal that accepts interpolation.

```ruby
%w[a#{b}c xyz]
```

In the example above, `word_add` would be dispatched four times. Three for the first word (the plain content, the interpolation, and the plain content), and once for the second word.

The handler for this event accepts the accumulated word (either a `word_add` or a [word_new](#word_new) node) and the part that should be appended (anything that can be added into a [string_add](#string_add)).

```ruby
def on_word_add(word, part); end
```

### `word_new`

`word_new` is a parser event that represents the beginning of a word within a special array literal (either strings or symbols) that accepts interpolation. For example, in the following array, there are three word nodes:

```ruby
%W[one a#{two}a three]
```

Each word inside that array is represented as its own node, which is in terms of the parser a tree of [word_new](#word_new) and [word_add](#word_add) nodes. The handler for this event accepts no parameters as it is the start of a list.

```ruby
def on_word_new; end
```

### `words_beg`

`words_beg` is a scanner event that represents the beginning of a string literal array with interpolation. For example:

```ruby
%W[one two three]
```

In the snippet above, a `words_beg` event would be dispatched with the value of `"%W["`. The handler for this event accepts a single string parameter representing the token as seen in the source. Note that these kinds of arrays can start with a lot of different delimiter types (e.g., `%W|` or `%W<`).

```ruby
def on_words_beg(value); end
```

### `words_add`

`words_add` is a parser event that represents adding an element to a string literal array with interpolation.

```ruby
%W[one two three]
```

In the example above, three `words_add` events would be dispatched. The first would be with the result of the [words_new](#words_new) event handler, the second with the result of the first `words_add` event handler call, and the third with the result of the second call.

The handler for this event accepts the current list of strings (as a [words_new](#words_new) or [words_add](#words_add) node) and the next value that should be added (always a [word_add](#word_add) node).

```ruby
def on_words_add(words, word_add); end
```

### `words_new`

`words_new` is a parser event that represents the beginning of a string literal array with interpolation.

```ruby
%W[one two three]
```

In the example above, a `words_new` event would be dispatched when the parser finds the `one` token. It can be followed by any number of [words_add](#words_add) events. The handler for this event accepts no parameters, as it's the start of a list.

```ruby
def on_words_new; end
```

### `words_sep`

`words_sep` is a scanner event that represents the separation between two words inside of a word literal array, or before/after the first/last word, if any. It contains any amount of whitespace characters that are used to delimit the words. For example,

```ruby
%w[
  one
  two
  three
]
```

In the snippet above there would be four `words_sep` events dispatched:
- between `%w[` and `one`
- between `one` and `two`
- between `two` and `three`
- between `three` and `]`

The handler for this event accepts a single string parameter that contains the separation between the two consecutive elements in the array.

```ruby
def on_words_sep(value); end
```

### `xstring_add`

`xstring_add` is a parser event that represents appending a part of a string of commands that gets sent out to the terminal.

```ruby
`ls`
```

The handler for this event accepts two parameters. The first is either an [xstring_new](#xstring_new) node (if this is the first part of the string) or an [xstring_add](#xstring_add) node (if this is not the first part of the string). The second is the part of the string to append (which can be any node that [string_add](#string_add) accepts).

```ruby
def on_xstring_add(xstring, part); end
```

### `xstring_new`

`xstring_new` is a parser event that represents the beginning of a string of commands that gets sent out to the terminal.

```ruby
`ls`
```

The handler for this event accepts no parameters, as it is the start of a list.

```ruby
def on_xstring_new; end
```

### `xstring_literal`

`xstring_literal` is a parser event that represents a string of commands that gets sent to the terminal.

```ruby
`ls`
```

They can also use heredocs to present themselves, as in the example:

```ruby
<<-`CMD`
  ls
CMD
```

The handler for this event accepts a single [xstring_new](#xstring_new) node (if it is empty) or [xstring_add](#xstring_add) node (if it is not empty) which represents its contents.

```ruby
def on_xstring_literal(xstring); end
```

### `yield`

`yield` is a parser event that represents using the `yield` keyword with arguments.

```ruby
yield value
```

The handler for this event accepts a single [args_add_block](#args_add_block) event that contains all of the arguments being passed. It can also be a [paren](#paren) node if parentheses are being used.

```ruby
def on_yield(args); end
```

### `yield0`

`yield0` is a parser event that represents the bare `yield` keyword.

```ruby
yield
```

The handler for this event accepts no parameters. This is as opposed to the `yield` parser event, which is the version where one or more values are being yielded.

```ruby
def on_yield0; end
```

### `zsuper`

`zsuper` is a parser event that represents the bare `super` keyword.

```ruby
super
```

The handler for this event accepts no parameters. This is as opposed to the `super` parser event, which is the version where one or more values are being passed to the `super` keyword.

```ruby
def on_zsuper; end
```

<!--
## Errors

# If we encounter a parse error, just immediately bail out so that our runner
# can catch it.
def on_parse_error(error, *)
  raise ParserError.new(error, lineno, column)
end
alias on_alias_error on_parse_error
alias on_assign_error on_parse_error
alias on_class_name_error on_parse_error
alias on_param_error on_parse_error
-->

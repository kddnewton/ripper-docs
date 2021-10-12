# Ripper

Documentation for the ripper Ruby standard library.

## Events

### `BEGIN` {#BEGIN}

`BEGIN` is a parser event that represents the use of the `BEGIN` keyword, which hooks into the lifecycle of the interpreter. Whatever is inside the block will get executed when the program starts. The syntax looks like the following:

```ruby
BEGIN {
  # execute stuff here
}
```

Interestingly, you can't use the `do...end` keywords for the block. The handler for this event accepts one parameter that is always a `stmts` node:

```ruby
def on_BEGIN(stmts); end
```

### `CHAR`

`CHAR` is a parser event that represents a single codepoint in the script encoding. For example:

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
  # execute stuff here
}
```

Interestingly, you can't use the `do...end` keywords for the block. The handler for this event accepts one parameter that is always a `stmts` node:

```ruby
def on_END(stmts); end
```

### `__end__`

`__end__` is a scanner event that represents `__END__` syntax, which allows individual scripts to keep content after the main ruby code that can be read through the DATA constant. It looks like:

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

The handler for this event accepts two parameters, that correspond to the first and second arguments to the keyword. So, for the above example the left would be the symbol literal `aliased_name` and the right could be the symbol literal `name`. Either argument can be a `dyna_symbol` node or a `symbol_literal` node.

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

Because the left-hand side of this expression can be any primary Ruby expression, its type is not known. The right-hand side (if provided) will be either an `args_add` node or an `args_add_block` node.

```ruby
def on_aref(collection, index); end
```

### `aref_field`

`aref_field` nodes are for assigning values into collections at specific indices. Put another way, it's any time you're calling the method `#[]=`. The `aref_field` node itself is just the left side of the assignment, and they're always wrapped in `assign` nodes. As an example:

```ruby
foo[]
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

there would be an `arg_paren` node around the `args_add_block` node that represents the set of arguments being sent to the `method` method. The argument child node can be `nil` if no arguments were passed, as in:

```ruby
method()
```

The handler for this event accepts one parameter, which is either an `args_add`, `args_add_block`, or `args_forward` node. It can also optionally be `nil`.

```ruby
def on_arg_paren(args); end
```

<!--
export type Args = ParserEvent<"args", { body: AnyNode[] }>;
export type ArgsAddBlock = ParserEvent<"args_add_block", { body: [Args | ArgsAddStar, false | AnyNode] }>;
export type ArgsAddStar = ParserEvent<"args_add_star", { body: [Args | ArgsAddStar, ...AnyNode[]] }>;
export type ArgsForward = ParserEvent0<"args_forward">;
export type Array = ParserEvent<"array", { body: [null | Args | ArgsAddStar | Qsymbols | Qwords | Symbols | Words] }>;
export type Aryptn = ParserEvent<"aryptn", { body: [null | VarRef, AnyNode[], null | VarField, null | AnyNode[]] }>;
export type Assign = ParserEvent<"assign", { body: [Assignable, AnyNode] }>;
export type AssocNew = ParserEvent<"assoc_new", { body: [AnyNode, AnyNode] }>;
export type AssocSplat = ParserEvent<"assoc_splat", { body: [AnyNode] }>;
export type AssoclistFromArgs = ParserEvent<"assoclist_from_args", { body: HashContent[] }>;
export type BareAssocHash = ParserEvent<"bare_assoc_hash", { body: HashContent[] }>;
export type Begin = ParserEvent<"begin", { body: [Bodystmt] }>;
export type Binary = ParserEvent<"binary", { body: [AnyNode, string, AnyNode] }>;
export type BlockVar = ParserEvent<"block_var", { body: [Params, false | Identifier[]] }>;
export type Blockarg = ParserEvent<"blockarg", { body: [Identifier] }>;
export type Bodystmt = ParserEvent<"bodystmt", { body: [Stmts, null | Rescue, null | Stmts, null | Ensure] }>;
export type BraceBlock = ParserEvent<"brace_block", { body: [null | BlockVar, Stmts], beging: Lbrace }>;
export type Break = ParserEvent<"break", { body: [Args | ArgsAddBlock] }>;
export type Call = ParserEvent<"call", { body: [AnyNode, CallOperator, Backtick | Op | Identifier | Const | "call"] }>;
export type CallOperator = Op | Period | "::";
export type Case = ParserEvent<"case", { body: [AnyNode, In | When] }>;
export type Class = ParserEvent<"class", { body: [ConstPathRef | ConstRef | TopConstRef, null | AnyNode, Bodystmt] }>;
export type Command = ParserEvent<"command", { body: [Const | Identifier, Args | ArgsAddBlock] }>;
export type CommandCall = ParserEvent<"command_call", { body: [AnyNode, CallOperator, Op | Identifier | Const, Args | ArgsAddBlock] }>;
export type ConstPathField = ParserEvent<"const_path_field", { body: [ConstPathRef | Paren | TopConstRef | VarRef, Const] }>;
export type ConstPathRef = ParserEvent<"const_path_ref", { body: [AnyNode, Const] }>;
export type ConstRef = ParserEvent<"const_ref", { body: [Const] }>;
export type Def = ParserEvent<"def", { body: [Backtick | Const | Identifier | Keyword | Op, Params | Paren, Bodystmt] }>;
export type Defined = ParserEvent<"defined", { body: [AnyNode] }>;
export type Defs = ParserEvent<"defs", { body: [AnyNode, Op | Period, Const | Op | Identifier | Keyword, Params | Paren, Bodystmt] }>;
export type Defsl = ParserEvent<"defsl", { body: [Identifier, null | ParenAroundParams, AnyNode] }>;
export type DoBlock = ParserEvent<"do_block", { body: [null | BlockVar, Bodystmt], beging: Keyword }>;
export type Dot2 = ParserEvent<"dot2", { body: [AnyNode, null] | [null, AnyNode] | [AnyNode, AnyNode] }>;
export type Dot3 = ParserEvent<"dot3", { body: [AnyNode, null] | [null, AnyNode] | [AnyNode, AnyNode] }>;
export type DynaSymbol = ParserEvent<"dyna_symbol", { body: StringContent[], quote: string }>;
export type END = ParserEvent<"END", { body: [Lbrace, Stmts] }>;
export type Else = ParserEvent<"else", { body: [Stmts] }>;
export type Elsif = ParserEvent<"elsif", { body: [AnyNode, Stmts, null | Elsif | Else] }>;
export type Ensure = ParserEvent<"ensure", { body: [Keyword, Stmts] }>;
export type ExcessedComma = ParserEvent0<"excessed_comma">;
export type Fcall = ParserEvent<"fcall", { body: [Const | Identifier] }>;
export type Field = ParserEvent<"field", { body: [AnyNode, CallOperator, Const | Identifier] }>;
export type FndPtn = ParserEvent<"fndptn", { body: [null | AnyNode, VarField, AnyNode[], VarField] }>;
export type For = ParserEvent<"for", { body: [Mlhs | MlhsAddStar | VarField, AnyNode, Stmts] }>;
export type Hash = ParserEvent<"hash", { body: [null | AssoclistFromArgs] }>;
export type Heredoc = ParserEvent<"heredoc", { beging: HeredocBegin, ending: string, body: StringContent[] }>;
export type Hshptn = ParserEvent<"hshptn", { body: [null | AnyNode, [Label, AnyNode][], null | VarField] }>;
export type If = ParserEvent<"if", { body: [AnyNode, Stmts, null | Elsif | Else] }>;
export type IfModifier = ParserEvent<"if_mod", { body: [AnyNode, AnyNode] }>;
export type In = ParserEvent<"in", { body: [AnyNode, Stmts, null | In | Else] }>;
export type KeywordRestParam = ParserEvent<"kwrest_param", { body: [null | Identifier] }>;
export type Lambda = ParserEvent<"lambda", { body: [Params | ParenAroundParams, Bodystmt | Stmts] }>;
export type Massign = ParserEvent<"massign", { body: [Mlhs | MlhsAddPost | MlhsAddStar | MlhsParen, AnyNode] }>;
export type MethodAddArg = ParserEvent<"method_add_arg", { body: [Call | Fcall, Args | ArgParen | ArgsAddBlock] }>;
export type MethodAddBlock = ParserEvent<"method_add_block", { body: [AnyNode, BraceBlock | DoBlock] }>;
export type Mlhs = ParserEvent<"mlhs", { body: (ArefField | Field | Identifier | MlhsParen | VarField)[], comma: undefined | true }>;
export type MlhsAddPost = ParserEvent<"mlhs_add_post", { body: [MlhsAddStar, Mlhs] }>;
export type MlhsAddStar = ParserEvent<"mlhs_add_star", { body: [Mlhs, null | ArefField | Field | Identifier | VarField] }>;
export type MlhsParen = ParserEvent<"mlhs_paren", { body: [Mlhs | MlhsAddPost | MlhsAddStar | MlhsParen] }>;
export type Module = ParserEvent<"module", { body: [ConstPathRef | ConstRef | TopConstRef, Bodystmt] }>;
export type Mrhs = ParserEvent<"mrhs", { body: [] }>;
export type MrhsAddStar = ParserEvent<"mrhs_add_star", { body: [Mrhs | MrhsNewFromArgs, AnyNode] }>;
export type MrhsNewFromArgs = ParserEvent<"mrhs_new_from_args", { body: [Args | ArgsAddStar, AnyNode], oper: string }>;
export type Next = ParserEvent<"next", { body: [Args | ArgsAddBlock] }>;
export type Opassign = ParserEvent<"opassign", { body: [Assignable, Op, AnyNode] }>;
export type Params = ParserEvent<"params", { body: [Identifier[], null | [Identifier, AnyNode][], null | ArgsForward | ExcessedComma | RestParam, Identifier[], null | [Label, AnyNode][], null | "nil" | KeywordRestParam, null | Blockarg] }>;
export type Paren = ParserEvent<"paren", { body: [AnyNode], lparen: Lparen }>;
export type Program = ParserEvent<"program", { body: [Stmts] }>;
export type Qsymbols = ParserEvent<"qsymbols", { body: TStringContent[] }>;
export type Qwords = ParserEvent<"qwords", { body: TStringContent[] }>;
export type Rassign = ParserEvent<"rassign", { body: [AnyNode, AnyNode], keyword: boolean }>;
export type Redo = ParserEvent0<"redo">;
export type RegexpLiteral = ParserEvent<"regexp_literal", { body: StringContent[], beging: string, ending: string }>;
export type Rescue = ParserEvent<"rescue", { body: [null | RescueEx, Stmts, null | Stmts] }>;
export type RescueEx = ParserEvent<"rescue_ex", { body: [AnyNode, null | Field | VarField] }>;
export type RescueModifier = ParserEvent<"rescue_mod", { body: [AnyNode, AnyNode] }>;
export type RestParam = ParserEvent<"rest_param", { body: [null | Identifier] }>;
export type Retry = ParserEvent0<"retry">;
export type Return = ParserEvent<"return", { body: [Args | ArgsAddBlock] }>;
export type Return0 = ParserEvent0<"return0">;
export type Sclass = ParserEvent<"sclass", { body: [AnyNode, Bodystmt] }>;
export type Stmts = ParserEvent<"stmts", { body: AnyNode[] }>;
export type String = ParserEvent<"string", { body: [TStringContent] }>;
export type StringConcat = ParserEvent<"string_concat", { body: [StringConcat | StringLiteral, StringLiteral] }>;
export type StringContent = StringDVar | StringEmbExpr | TStringContent;
export type StringDVar = ParserEvent<"string_dvar", { body: [Backref | VarRef] }>;
export type StringEmbExpr = ParserEvent<"string_embexpr", { body: [Stmts] }>;
export type StringLiteral = ParserEvent<"string_literal", { body: StringContent[], quote: string }>;
export type Super = ParserEvent<"super", { body: [Args | ArgParen | ArgsAddBlock] }>;
export type SymbolLiteral = ParserEvent<"symbol_literal", { body: [Backtick | Const | CVar | GVar | Identifier | IVar | Keyword | Op] }>;
export type Symbols = ParserEvent<"symbols", { body: Word[] }>;
export type Ternary = ParserEvent<"ifop", { body: [AnyNode, AnyNode, AnyNode] }>;
export type TopConstField = ParserEvent<"top_const_field", { body: [Const] }>;
export type TopConstRef = ParserEvent<"top_const_ref", { body: [Const] }>;
export type Unary = ParserEvent<"unary", { body: [AnyNode], oper: string, paren: boolean | undefined }>;
export type Undef = ParserEvent<"undef", { body: (DynaSymbol | SymbolLiteral)[] }>;
export type Unless = ParserEvent<"unless", { body: [AnyNode, Stmts, null | Elsif | Else] }>;
export type UnlessModifier = ParserEvent<"unless_mod", { body: [AnyNode, AnyNode] }>;
export type Until = ParserEvent<"until", { body: [AnyNode, Stmts] }>;
export type UntilModifier = ParserEvent<"until_mod", { body: [AnyNode, AnyNode] }>;
export type VCall = ParserEvent<"vcall", { body: [Identifier] }>;
export type VarAlias = ParserEvent<"var_alias", { body: [GVar, Backref | GVar] }>;
export type VarField = ParserEvent<"var_field", { body: [null | Const | CVar | GVar | Identifier | IVar] }>;
export type VarRef = ParserEvent<"var_ref", { body: [Const | CVar | GVar | Identifier | IVar | Keyword] }>;
export type VoidStmt = ParserEvent<"void_stmt">;
export type When = ParserEvent<"when", { body: [Args | ArgsAddStar, Stmts, null | Else | When] }>;
export type While = ParserEvent<"while", { body: [AnyNode, Stmts] }>;
export type WhileModifier = ParserEvent<"while_mod", { body: [AnyNode, AnyNode] }>;
export type Word = ParserEvent<"word", { body: StringContent[] }>;
export type Words = ParserEvent<"words", { body: Word[] }>;
export type XStringLiteral = ParserEvent<"xstring_literal", { body: StringContent[] }>;
export type Yield = ParserEvent<"yield", { body: [ArgsAddBlock | Paren] }>;
export type Yield0 = ParserEvent0<"yield0">;
export type Zsuper = ParserEvent0<"zsuper">;

type Assignable = ArefField | ConstPathField | Field | TopConstField | VarField;
type HashContent = AssocNew | AssocSplat;
type ParenAroundParams = Omit<Paren, "body"> & { body: [Params] };
-->

<!--
  # args_add is a parser event that represents a single argument inside a list
  # of arguments to any method call or an array. It accepts as arguments the
  # parent args node as well as an arg which can be anything that could be
  # passed as an argument.
  def on_args_add(args, arg)
    if args[:body].empty?
      # If this is the first argument being passed into the list of arguments,
      # then we're going to use the bounds of the argument to override the
      # parent node's location since this will be more accurate.
      arg.merge(type: :args, body: [arg])
    else
      args.merge!(body: args[:body] << arg, el: arg[:el], ec: arg[:ec])
    end
  end

  # args_add_block is a parser event that represents a list of arguments and
  # potentially a block argument. If no block is passed, then the second
  # argument will be the literal false.
  def on_args_add_block(args, block)
    ending = block || args

    args.merge(
      type: :args_add_block,
      body: [args, block],
      el: ending[:el],
      ec: ending[:ec]
    )
  end

  # args_add_star is a parser event that represents adding a splat of values
  # to a list of arguments. If accepts as arguments the parent args node as
  # well as the part that is being splatted.
  def on_args_add_star(args, part)
    beging = find_scanner_event(:@op, '*')
    ending = part || beging

    {
      type: :args_add_star,
      body: [args, part],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # args_forward is a parser event that represents forwarding all kinds of
  # arguments onto another method call.
  def on_args_forward
    find_scanner_event(:@op, '...').merge!(type: :args_forward)
  end

  # args_new is a parser event that represents the beginning of a list of
  # arguments to any method call or an array. It can be followed by any
  # number of args_add events, which we'll append onto an array body.
  def on_args_new
    {
      type: :args,
      body: [],
      sl: lineno,
      sc: char_pos,
      el: lineno,
      ec: char_pos
    }
  end

  # Array nodes can contain a myriad of subnodes because of the special
  # array literal syntax like %w and %i. As a result, we may be looking for
  # an left bracket, or we may be just looking at the children to get the
  # bounds.
  def on_array(contents)
    if !contents || %i[args args_add_star].include?(contents[:type])
      beging = find_scanner_event(:@lbracket)
      ending = find_scanner_event(:@rbracket)

      {
        type: :array,
        body: [contents],
        sl: beging[:sl],
        sc: beging[:sc],
        el: ending[:el],
        ec: ending[:ec]
      }
    else
      ending = find_scanner_event(:@tstring_end)
      contents[:ec] = ending[:ec]

      ending.merge!(
        type: :array,
        body: [contents],
        sl: contents[:sl],
        sc: contents[:sc]
      )
    end
  end

  # aryptn is a parser event that represents matching against an array pattern
  # using the Ruby 2.7+ pattern matching syntax.
  def on_aryptn(const, preargs, splatarg, postargs)
    pieces = [const, *preargs, splatarg, *postargs].compact

    {
      type: :aryptn,
      body: [const, preargs, splatarg, postargs],
      sl: pieces[0][:sl],
      sc: pieces[0][:sc],
      el: pieces[-1][:el],
      ec: pieces[-1][:ec]
    }
  end

  # assign is a parser event that represents assigning something to a
  # variable or constant. It accepts as arguments the left side of the
  # expression before the equals sign and the right side of the expression.
  def on_assign(left, right)
    left.merge(
      type: :assign,
      body: [left, right],
      el: right[:el],
      ec: right[:ec]
    )
  end

  # assoc_new is a parser event that contains a key-value pair within a
  # hash. It is a child event of either an assoclist_from_args or a
  # bare_assoc_hash.
  def on_assoc_new(key, value)
    {
      type: :assoc_new,
      body: [key, value],
      sl: key[:sl],
      sc: key[:sc],
      el: value[:el],
      ec: value[:ec]
    }
  end

  # assoc_splat is a parser event that represents splatting a value into a
  # hash (either a hash literal or a bare hash in a method call).
  def on_assoc_splat(contents)
    find_scanner_event(:@op, '**').merge!(
      type: :assoc_splat,
      body: [contents],
      el: contents[:el],
      ec: contents[:ec]
    )
  end

  # assoclist_from_args is a parser event that contains a list of all of the
  # associations inside of a hash literal. Its parent node is always a hash.
  # It accepts as an argument an array of assoc events (either assoc_new or
  # assoc_splat).
  def on_assoclist_from_args(assocs)
    {
      type: :assoclist_from_args,
      body: assocs,
      sl: assocs[0][:sl],
      sc: assocs[0][:sc],
      el: assocs[-1][:el],
      ec: assocs[-1][:ec]
    }
  end

  # backref is a scanner event that represents a global variable referencing a
  # matched value. It comes in the form of a $ followed by a positive integer.
  def on_backref(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@backref,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # backtick is a scanner event that represents the use of the ` operator. It's
  # usually found being used for an xstring, but could also be found as the name
  # of a method being defined.
  def on_backtick(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@backtick,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # bare_assoc_hash is a parser event that represents a hash of contents
  # being passed as a method argument (and therefore has omitted braces). It
  # accepts as an argument an array of assoc events (either assoc_new or
  # assoc_splat).
  def on_bare_assoc_hash(assoc_news)
    {
      type: :bare_assoc_hash,
      body: assoc_news,
      sl: assoc_news[0][:sl],
      sc: assoc_news[0][:sc],
      el: assoc_news[-1][:el],
      ec: assoc_news[-1][:ec]
    }
  end

  # begin is a parser event that represents the beginning of a begin..end chain.
  # It includes a bodystmt event that has all of the consequent clauses.
  def on_begin(bodystmt)
    beging = find_scanner_event(:@kw, 'begin')
    ec =
      if bodystmt[:body][1..-1].any?
        bodystmt[:ec]
      else
        find_scanner_event(:@kw, 'end')[:ec]
      end

    bodystmt.bind(beging[:ec], ec)

    beging.merge!(
      type: :begin,
      body: [bodystmt],
      el: bodystmt[:el],
      ec: bodystmt[:ec]
    )
  end

  # binary is a parser event that represents a binary operation between two
  # values.
  def on_binary(left, oper, right)
    # On most Ruby implementations, oper is a Symbol that represents that
    # operation being performed. For instance in the example `1 < 2`, the `oper`
    # object would be `:<`. However, on JRuby, it's an `@op` node, so here we're
    # going to explicitly convert it into the same normalized form.
    oper = scanner_events.delete(oper)[:body] unless oper.is_a?(Symbol)

    {
      type: :binary,
      body: [left, oper, right],
      sl: left[:sl],
      sc: left[:sc],
      el: right[:el],
      ec: right[:ec]
    }
  end

  # block_var is a parser event that represents the parameters being passed to
  # block. Effectively they're everything contained within the pipes.
  def on_block_var(params, locals)
    index =
      scanner_events.rindex do |event|
        event[:type] == :@op && %w[| ||].include?(event[:body]) &&
          event[:sc] < params[:sc]
      end

    beging = scanner_events[index]
    ending = scanner_events[-1]

    {
      type: :block_var,
      body: [params, locals],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # blockarg is a parser event that represents defining a block variable on
  # a method definition.
  def on_blockarg(ident)
    find_scanner_event(:@op, '&').merge!(
      type: :blockarg,
      body: [ident],
      el: ident[:el],
      ec: ident[:ec]
    )
  end

  # bodystmt can't actually determine its bounds appropriately because it
  # doesn't necessarily know where it started. So the parent node needs to
  # report back down into this one where it goes.
  class BodyStmt < Node
    def bind(sc, ec)
      value.merge!(sc: sc, ec: ec)
      parts = value[:body]

      # Here we're going to determine the bounds for the stmts
      consequent = parts[1..-1].compact.first
      value[:body][0].bind(sc, consequent ? consequent[:sc] : ec)

      # Next we're going to determine the rescue clause if there is one
      if parts[1]
        consequent = parts[2..-1].compact.first
        value[:body][1].bind_end(consequent ? consequent[:sc] : ec)
      end
    end
  end

  # bodystmt is a parser event that represents all of the possible combinations
  # of clauses within the body of a method or block.
  def on_bodystmt(stmts, rescued, ensured, elsed)
    BodyStmt.new(
      self,
      type: :bodystmt,
      body: [stmts, rescued, ensured, elsed],
      sl: lineno,
      sc: char_pos,
      el: lineno,
      ec: char_pos
    )
  end

  # brace_block is a parser event that represents passing a block to a
  # method call using the {..} operators. It accepts as arguments an
  # optional block_var event that represents any parameters to the block as
  # well as a stmts event that represents the statements inside the block.
  def on_brace_block(block_var, stmts)
    beging = find_scanner_event(:@lbrace)
    ending = find_scanner_event(:@rbrace)

    stmts.bind(
      find_next_statement_start((block_var || beging)[:ec]),
      ending[:sc]
    )

    {
      type: :brace_block,
      body: [block_var, stmts],
      beging: beging,
      sl: beging[:sl],
      sc: beging[:sc],
      el: [ending[:el], stmts[:el]].max,
      ec: ending[:ec]
    }
  end

  # break is a parser event that represents using the break keyword. It
  # accepts as an argument an args or args_add_block event that contains all
  # of the arguments being passed to the break.
  def on_break(args_add_block)
    beging = find_scanner_event(:@kw, 'break')

    # You can hit this if you are passing no arguments to break but it has a
    # comment right after it. In that case we can just use the location
    # information straight from the keyword.
    if args_add_block[:type] == :args
      return beging.merge!(type: :break, body: [args_add_block])
    end

    beging.merge!(
      type: :break,
      body: [args_add_block],
      el: args_add_block[:el],
      ec: args_add_block[:ec]
    )
  end

  # call is a parser event representing a method call with no arguments. It
  # accepts as arguments the receiver of the method, the operator being used
  # to send the method (., ::, or &.), and the value that is being sent to
  # the receiver (which can be another nested call as well).
  #
  # There is one esoteric syntax that comes into play here as well. If the
  # sending argument to this method is the symbol :call, then it represents
  # calling a lambda in a very odd looking way, as in:
  #
  #     foo.(1, 2, 3)
  #
  def on_call(receiver, oper, sending)
    ending = sending

    if sending == :call
      ending = oper

      # Special handling here for Ruby <= 2.5 because the oper argument to this
      # method wasn't a parser event here it was just a plain symbol.
      ending = receiver if RUBY_MAJOR <= 2 && RUBY_MINOR <= 5
    end

    {
      type: :call,
      body: [receiver, oper, sending],
      sl: receiver[:sl],
      sc: receiver[:sc],
      el: [ending[:el], receiver[:el]].max,
      ec: ending[:ec]
    }
  end

  # case is a parser event that represents the beginning of a case chain.
  # It accepts as arguments the switch of the case and the consequent
  # clause.
  def on_case(switch, consequent)
    beging =
      if event = find_scanner_event(:@kw, 'case', consume: false)
        scanner_events.delete(event).merge!(type: :case)
      else
        keyword = find_scanner_event(:@kw, 'in', consume: false)
        switch.merge(type: :rassign, keyword: keyword)
      end

    beging.merge!(
      body: [switch, consequent],
      el: consequent[:el],
      ec: consequent[:ec]
    )
  end

  # class is a parser event that represents defining a class. It accepts as
  # arguments the name of the class, the optional name of the superclass,
  # and the bodystmt event that represents the statements evaluated within
  # the context of the class.
  def on_class(const, superclass, bodystmt)
    beging = find_scanner_event(:@kw, 'class')
    ending = find_scanner_event(:@kw, 'end')

    bodystmt.bind(
      find_next_statement_start((superclass || const)[:ec]),
      ending[:sc]
    )

    {
      type: :class,
      body: [const, superclass, bodystmt],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # comma is a scanner event that represents the use of the comma operator.
  def on_comma(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@comma,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # command is a parser event representing a method call with arguments and
  # no parentheses. It accepts as arguments the name of the method and the
  # arguments being passed to the method.
  def on_command(ident, args)
    {
      type: :command,
      body: [ident, args],
      sl: ident[:sl],
      sc: ident[:sc],
      el: args[:el],
      ec: args[:ec]
    }
  end

  # command_call is a parser event representing a method call on an object
  # with arguments and no parentheses. It accepts as arguments the receiver
  # of the method, the operator being used to send the method, the name of
  # the method, and the arguments being passed to the method.
  def on_command_call(receiver, oper, ident, args)
    ending = args || ident

    {
      type: :command_call,
      body: [receiver, oper, ident, args],
      sl: receiver[:sl],
      sc: receiver[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # We keep track of each comment as it comes in and then eventually add
  # them to the top of the generated AST so that prettier can start adding
  # them back into the final representation. Comments come in including
  # their starting pound sign and the newline at the end, so we also chop
  # those off.
  def on_comment(value)
    # If there is an encoding magic comment at the top of the file, ripper
    # will actually change into that encoding for the storage of the string.
    # This will break everything when we attempt to print as JSON, so we need to
    # force the encoding back into UTF-8 so that it won't break.
    body = value[1..-1].chomp.force_encoding('UTF-8')

    start_line = lineno
    start_char = char_pos

    @comments << {
      type: :@comment,
      value: body,
      inline: value.strip != lines[lineno - 1],
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.length - 1
    }
  end

  # const is a scanner event that represents a literal value that _looks like_
  # a constant. This could actually be a reference to a constant. It could also
  # be something that looks like a constant in another context, as in a method
  # call to a capitalized method, a symbol that starts with a capital letter,
  # etc.
  def on_const(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@const,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # A const_path_field is a parser event that is always the child of some
  # kind of assignment. It represents when you're assigning to a constant
  # that is being referenced as a child of another variable. For example:
  #
  #     foo::X = 1
  #
  def on_const_path_field(left, const)
    {
      type: :const_path_field,
      body: [left, const],
      sl: left[:sl],
      sc: left[:sc],
      el: const[:el],
      ec: const[:ec]
    }
  end

  # A const_path_ref is a parser event that is a very similar to
  # const_path_field except that it is not involved in an assignment. It
  # looks like the following example: foo::Bar, where left is foo and const is
  # Bar.
  def on_const_path_ref(left, const)
    {
      type: :const_path_ref,
      body: [left, const],
      sl: left[:sl],
      sc: left[:sc],
      el: const[:el],
      ec: const[:ec]
    }
  end

  # A const_ref is a parser event that represents the name of the constant
  # being used in a class or module declaration. In the following example it
  # is the @const scanner event that has the contents of Foo.
  #
  #     class Foo; end
  #
  def on_const_ref(const)
    const.merge(type: :const_ref, body: [const])
  end

  # cvar is a scanner event that represents the use of a class variable.
  def on_cvar(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@cvar,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # A def is a parser event that represents defining a regular method on the
  # current self object. It accepts as arguments the ident (the name of the
  # method being defined), the params (the parameter declaration for the
  # method), and a bodystmt node which represents the statements inside the
  # method. As an example, here are the parts that go into this:
  #
  #     def foo(bar) do baz end
  #          │   │       │
  #          │   │       └> bodystmt
  #          │   └> params
  #          └> ident
  #
  # You can also have single-line methods since Ruby 3.0+, which have slightly
  # different syntax but still flow through this method. Those look like:
  #
  #     def foo = bar
  #          |     |
  #          |     └> stmt
  #          └> ident
  #
  def on_def(ident, params, bodystmt)
    # Make sure to delete this scanner event in case you're defining something
    # like def class which would lead to this being a kw and causing all kinds
    # of trouble
    scanner_events.delete(ident)

    # Find the beginning of the method definition, which works for single-line
    # and normal method definitions.
    beging = find_scanner_event(:@kw, 'def')

    # If we don't have a bodystmt node, then we have a single-line method
    if bodystmt[:type] != :bodystmt
      return(
        {
          type: :defsl,
          body: [ident, params, bodystmt],
          sl: beging[:sl],
          sc: beging[:sc],
          el: bodystmt[:el],
          ec: bodystmt[:ec]
        }
      )
    end

    if params[:type] == :params && !params[:body].any?
      location = ident[:ec]
      params.merge!(sc: location, ec: location)
    end

    ending = find_scanner_event(:@kw, 'end')

    bodystmt.bind(find_next_statement_start(params[:ec]), ending[:sc])

    {
      type: :def,
      body: [ident, params, bodystmt],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # A defs is a parser event that represents defining a singleton method on
  # an object. It accepts the same arguments as the def event, as well as
  # the target and operator that on which this method is being defined. As
  # an example, here are the parts that go into this:
  #
  #     def foo.bar(baz) do baz end
  #          │ │ │   │       │
  #          │ │ │   │       │
  #          │ │ │   │       └> bodystmt
  #          │ │ │   └> params
  #          │ │ └> ident
  #          │ └> oper
  #          └> target
  #
  def on_defs(target, oper, ident, params, bodystmt)
    # Make sure to delete this scanner event in case you're defining something
    # like def class which would lead to this being a kw and causing all kinds
    # of trouble
    scanner_events.delete(ident)

    if params[:type] == :params && !params[:body].any?
      location = ident[:ec]
      params.merge!(sc: location, ec: location)
    end

    beging = find_scanner_event(:@kw, 'def')
    ending = find_scanner_event(:@kw, 'end')

    bodystmt.bind(find_next_statement_start(params[:ec]), ending[:sc])

    {
      type: :defs,
      body: [target, oper, ident, params, bodystmt],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # A defined node represents the rather unique defined? operator. It can be
  # used with and without parentheses. If they're present, we use them to
  # determine our bounds, otherwise we use the value that's being passed to
  # the operator.
  def on_defined(value)
    beging = find_scanner_event(:@kw, 'defined?')

    paren = source[beging[:ec]...value[:sc]].include?('(')
    ending = paren ? find_scanner_event(:@rparen) : value

    beging.merge!(
      type: :defined,
      body: [value],
      el: ending[:el],
      ec: ending[:ec]
    )
  end

  # do_block is a parser event that represents passing a block to a method
  # call using the do..end keywords. It accepts as arguments an optional
  # block_var event that represents any parameters to the block as well as
  # a bodystmt event that represents the statements inside the block.
  def on_do_block(block_var, bodystmt)
    beging = find_scanner_event(:@kw, 'do')
    ending = find_scanner_event(:@kw, 'end')

    bodystmt.bind(
      find_next_statement_start((block_var || beging)[:ec]),
      ending[:sc]
    )

    {
      type: :do_block,
      body: [block_var, bodystmt],
      beging: beging,
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # dot2 is a parser event that represents using the .. operator between two
  # expressions. Usually this is to create a range object but sometimes it's to
  # use the flip-flop operator.
  def on_dot2(left, right)
    operator = find_scanner_event(:@op, '..')

    beging = left || operator
    ending = right || operator

    {
      type: :dot2,
      body: [left, right],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # dot3 is a parser event that represents using the ... operator between two
  # expressions. Usually this is to create a range object but sometimes it's to
  # use the flip-flop operator.
  def on_dot3(left, right)
    operator = find_scanner_event(:@op, '...')

    beging = left || operator
    ending = right || operator

    {
      type: :dot3,
      body: [left, right],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # A dyna_symbol is a parser event that represents a symbol literal that
  # uses quotes to interpolate its value. For example, if you had a variable
  # foo and you wanted a symbol that contained its value, you would write:
  #
  #     :"#{foo}"
  #
  # As such, they accept as one argument a string node, which is the same
  # node that gets accepted into a string_literal (since we're basically
  # talking about a string literal with a : character at the beginning).
  #
  # They can also come in another flavor which is a dynamic symbol as a hash
  # key. This is kind of an interesting syntax which results in us having to
  # look for a @label_end scanner event instead to get our bearings. That
  # kind of code would look like:
  #
  #     { "#{foo}": bar }
  #
  # which would be the same symbol as above.
  def on_dyna_symbol(string)
    if find_scanner_event(:@symbeg, consume: false)
      # A normal dynamic symbol
      beging = find_scanner_event(:@symbeg)
      ending = find_scanner_event(:@tstring_end)

      beging.merge(
        type: :dyna_symbol,
        quote: beging[:body],
        body: string[:body],
        el: ending[:el],
        ec: ending[:ec]
      )
    else
      # A dynamic symbol as a hash key
      beging = find_scanner_event(:@tstring_beg)
      ending = find_scanner_event(:@label_end)

      string.merge!(
        type: :dyna_symbol,
        quote: ending[:body][0],
        sl: beging[:sl],
        sc: beging[:sc],
        el: ending[:el],
        ec: ending[:ec]
      )
    end
  end

  # else is a parser event that represents the end of a if, unless, or begin
  # chain. It accepts as an argument the statements that are contained
  # within the else clause.
  def on_else(stmts)
    beging = find_scanner_event(:@kw, 'else')

    # else can either end with an end keyword (in which case we'll want to
    # consume that event) or it can end with an ensure keyword (in which case
    # we'll leave that to the ensure to handle).
    index =
      scanner_events.rindex do |event|
        event[:type] == :@kw && %w[end ensure].include?(event[:body])
      end

    event = scanner_events[index]
    ending = event[:body] == 'end' ? scanner_events.delete_at(index) : event

    stmts.bind(beging[:ec], ending[:sc])

    {
      type: :else,
      body: [stmts],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # elsif is a parser event that represents another clause in an if chain.
  # It accepts as arguments the predicate of the else if, the statements
  # that are contained within the else if clause, and the optional
  # consequent clause.
  def on_elsif(predicate, stmts, consequent)
    beging = find_scanner_event(:@kw, 'elsif')
    ending = consequent || find_scanner_event(:@kw, 'end')

    stmts.bind(predicate[:ec], ending[:sc])

    {
      type: :elsif,
      body: [predicate, stmts, consequent],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # This is a scanner event that gets hit when we're inside an embdoc and
  # receive a new line of content. Here we are guaranteed to already have
  # initialized the @embdoc variable so we can just append the new line onto
  # the existing content.
  def on_embdoc(value)
    @embdoc[:value] << value
  end

  # embdocs are long comments that are surrounded by =begin..=end. They
  # cannot be nested, so we don't need to worry about keeping a stack around
  # like we do with heredocs. Instead we can just track the current embdoc
  # and add to it as we get content. It always starts with this scanner
  # event, so here we'll initialize the current embdoc.
  def on_embdoc_beg(value)
    @embdoc = { type: :@embdoc, value: value, sl: lineno, sc: char_pos }
  end

  # This is the final scanner event for embdocs. It receives the =end. Here
  # we can finalize the embdoc with its location information and the final
  # piece of the string. We then add it to the list of comments so that
  # prettier can place it into the final source string.
  def on_embdoc_end(value)
    @comments <<
      @embdoc.merge!(
        value: @embdoc[:value] << value.chomp,
        el: lineno,
        ec: char_pos + value.length - 1
      )

    @embdoc = nil
  end

  # embexpr_beg is a scanner event that represents using interpolation inside of
  # a string, xstring, heredoc, or regexp. Its value is the string literal "#{".
  def on_embexpr_beg(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@embexpr_beg,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # embexpr_end is a scanner event that represents the end of an interpolated
  # expression in a string, xstring, heredoc, or regexp. Its value is the string
  # literal "}".
  def on_embexpr_end(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@embexpr_end,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # embvar is a scanner event that represents the use of shorthand interpolation
  # for an instance, class, or global variable into a string, xstring, heredoc,
  # or regexp. Its value is the string literal "#". For example, in the
  # following snippet:
  #
  #     "#@foo"
  #
  # the embvar would be triggered by the "#", then an ivar event for the @foo
  # instance variable. That would all get bound up into a string_dvar node in
  # the final AST.
  def on_embvar(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@embvar,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # ensure is a parser event that represents the use of the ensure keyword
  # and its subsequent statements.
  def on_ensure(stmts)
    beging = find_scanner_event(:@kw, 'ensure')

    # Specifically not using find_scanner_event here because we don't want to
    # consume the :@end event, because that would break def..ensure..end chains.
    index =
      scanner_events.rindex do |scanner_event|
        scanner_event[:type] == :@kw && scanner_event[:body] == 'end'
      end

    ending = scanner_events[index]
    stmts.bind(find_next_statement_start(beging[:ec]), ending[:sc])

    {
      type: :ensure,
      body: [beging, stmts],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # An excessed_comma is a special kind of parser event that represents a comma
  # at the end of a list of parameters. It's a very strange node. It accepts a
  # different number of arguments depending on Ruby version, which is why we
  # have the anonymous splat there.
  def on_excessed_comma(*)
    find_scanner_event(:@comma).merge!(type: :excessed_comma)
  end

  # An fcall is a parser event that represents the piece of a method call
  # that comes before any arguments (i.e., just the name of the method).
  def on_fcall(ident)
    ident.merge(type: :fcall, body: [ident])
  end

  # A field is a parser event that is always the child of an assignment. It
  # accepts as arguments the left side of operation, the operator (. or ::),
  # and the right side of the operation. For example:
  #
  #     foo.x = 1
  #
  def on_field(left, oper, right)
    {
      type: :field,
      body: [left, oper, right],
      sl: left[:sl],
      sc: left[:sc],
      el: right[:el],
      ec: right[:ec]
    }
  end

  # float is a scanner event that represents a floating point value literal.
  def on_float(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@float,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # fndptn is a parser event that represents matching against a pattern where
  # you find a pattern in an array using the Ruby 3.0+ pattern matching syntax.
  def on_fndptn(const, presplat, args, postsplat)
    beging = const || find_scanner_event(:@lbracket)
    ending = find_scanner_event(:@rbracket)

    {
      type: :fndptn,
      body: [const, presplat, args, postsplat],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # for is a parser event that represents using the somewhat esoteric for
  # loop. It accepts as arguments an ident which is the iterating variable,
  # an enumerable for that which is being enumerated, and a stmts event that
  # represents the statements inside the for loop.
  def on_for(ident, enum, stmts)
    beging = find_scanner_event(:@kw, 'for')
    ending = find_scanner_event(:@kw, 'end')

    # Consume the do keyword if it exists so that it doesn't get confused for
    # some other block
    do_event = find_scanner_event(:@kw, 'do', consume: false)
    if do_event && do_event[:sc] > enum[:ec] && do_event[:ec] < ending[:sc]
      scanner_events.delete(do_event)
    end

    stmts.bind((do_event || enum)[:ec], ending[:sc])

    {
      type: :for,
      body: [ident, enum, stmts],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # gvar is a scanner event that represents a global variable literal.
  def on_gvar(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@gvar,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # hash is a parser event that represents a hash literal. It accepts as an
  # argument an optional assoclist_from_args event which contains the
  # contents of the hash.
  def on_hash(assoclist_from_args)
    beging = find_scanner_event(:@lbrace)
    ending = find_scanner_event(:@rbrace)

    if assoclist_from_args
      # Here we're going to expand out the location information for the assocs
      # node so that it can grab up any remaining comments inside the hash.
      assoclist_from_args.merge!(sc: beging[:ec], ec: ending[:sc])
    end

    {
      type: :hash,
      body: [assoclist_from_args],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # This is a scanner event that represents the beginning of the heredoc. It
  # includes the declaration (which we call beging here, which is just short
  # for beginning). The declaration looks something like <<-HERE or <<~HERE.
  # If the downcased version of the declaration actually matches an existing
  # prettier parser, we'll later attempt to print it using that parser and
  # printer through our embed function.
  def on_heredoc_beg(beging)
    location = {
      sl: lineno,
      el: lineno,
      sc: char_pos,
      ec: char_pos + beging.length + 1
    }

    # Here we're going to artificially create an extra node type so that if
    # there are comments after the declaration of a heredoc, they get printed.
    location
      .merge(
        type: :heredoc,
        beging: location.merge(type: :@heredoc_beg, body: beging)
      )
      .tap { |node| @heredocs << node }
  end

  # This is a parser event that occurs when you're using a heredoc with a
  # tilde. These are considered `heredoc_dedent` nodes, whereas the hyphen
  # heredocs show up as string literals.
  def on_heredoc_dedent(string, _width)
    @heredocs[-1].merge!(body: string[:body])
  end

  # This is a scanner event that represents the end of the heredoc.
  def on_heredoc_end(ending)
    @heredocs[-1].merge!(ending: ending.chomp, el: lineno, ec: char_pos)
  end

  # hshptn is a parser event that represents matching against a hash pattern
  # using the Ruby 2.7+ pattern matching syntax.
  def on_hshptn(const, kw, kwrest)
    pieces = [const, kw, kwrest].flatten(2).compact

    {
      type: :hshptn,
      body: [const, kw, kwrest],
      sl: pieces[0][:sl],
      sc: pieces[0][:sc],
      el: pieces[-1][:el],
      ec: pieces[-1][:ec]
    }
  end

  # ident is a scanner event that represents an identifier anywhere in code. It
  # can actually represent a whole bunch of stuff, depending on where it is in
  # the AST. Like comments, we need to force the encoding here so JSON doesn't
  # break.
  def on_ident(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@ident,
      body: value.force_encoding('UTF-8'),
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # if is a parser event that represents the first clause in an if chain.
  # It accepts as arguments the predicate of the if, the statements that are
  # contained within the if clause, and the optional consequent clause.
  def on_if(predicate, stmts, consequent)
    beging = find_scanner_event(:@kw, 'if')
    ending = consequent || find_scanner_event(:@kw, 'end')

    stmts.bind(predicate[:ec], ending[:sc])

    {
      type: :if,
      body: [predicate, stmts, consequent],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # ifop is a parser event that represents a ternary operator. It accepts as
  # arguments the predicate to the ternary, the truthy clause, and the falsy
  # clause.
  def on_ifop(predicate, truthy, falsy)
    predicate.merge(
      type: :ifop,
      body: [predicate, truthy, falsy],
      el: falsy[:el],
      ec: falsy[:ec]
    )
  end

  # if_mod is a parser event that represents the modifier form of an if
  # statement. It accepts as arguments the predicate of the if and the
  # statement that are contained within the if clause.
  def on_if_mod(predicate, statement)
    find_scanner_event(:@kw, 'if')

    {
      type: :if_mod,
      body: [predicate, statement],
      sl: statement[:sl],
      sc: statement[:sc],
      el: predicate[:el],
      ec: predicate[:ec]
    }
  end

  # ignored_nl is a special kind of scanner event that passes nil as the value.
  # You can trigger the ignored_nl event with the following snippet:
  #
  #     foo.bar
  #        .baz
  #
  # We don't need to track this event in the AST that we're generating, so we're
  # not going to define an explicit handler for it.
  #
  #     def on_ignored_nl(value)
  #       value
  #     end

  # ignored_sp is a scanner event that represents the space before the content
  # of each line of a squiggly heredoc that will be removed from the string
  # before it gets transformed into a string literal. For example, in the
  # following snippet:
  #
  #     <<~HERE
  #       foo
  #         bar
  #     HERE
  #
  # You would have two ignored_sp events, the first with two spaces and the
  # second with four. We don't need to track this event in the AST that we're
  # generating, so we're not going to define an explicit handler for it.
  #
  #     def on_ignored_sp(value)
  #       value
  #     end

  # imaginary is a scanner event that represents an imaginary number literal.
  # They become instances of the Complex class.
  def on_imaginary(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@imaginary,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # in is a parser event that represents using the in keyword within the
  # Ruby 2.7+ pattern matching syntax. Alternatively in Ruby 3+ it is also used
  # to handle rightward assignment for pattern matching.
  def on_in(pattern, stmts, consequent)
    # Here we have a rightward assignment
    return pattern unless stmts

    beging = find_scanner_event(:@kw, 'in')
    ending = consequent || find_scanner_event(:@kw, 'end')

    stmts.bind(beging[:ec], ending[:sc])

    beging.merge!(
      type: :in,
      body: [pattern, stmts, consequent],
      el: ending[:el],
      ec: ending[:ec]
    )
  end

  # int is a scanner event the represents a number literal.
  def on_int(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@int,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # ivar is a scanner event the represents an instance variable literal.
  def on_ivar(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@ivar,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # kw is a scanner event the represents the use of a keyword. It can be
  # anywhere in the AST, so you end up seeing it quite a lot.
  def on_kw(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@kw,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # kwrest_param is a parser event that represents defining a parameter in a
  # method definition that accepts all remaining keyword parameters.
  def on_kwrest_param(ident)
    oper = find_scanner_event(:@op, '**')
    return oper.merge!(type: :kwrest_param, body: [nil]) unless ident

    oper.merge!(
      type: :kwrest_param,
      body: [ident],
      el: ident[:el],
      ec: ident[:ec]
    )
  end

  # label is a scanner event that represents the use of an identifier to
  # associate with an object. You can find it in a hash key, as in:
  #
  #     { foo: bar }
  #
  # in this case "foo:" would be the body of the label. You can also find it in
  # pattern matching, as in:
  #
  #     case foo
  #     in bar:
  #       bar
  #     end
  #
  # in this case "bar:" would be the body of the label.
  def on_label(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@label,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # label_end is a scanner event that represents the end of a dynamic symbol. If
  # for example you had the following hash:
  #
  #     { "foo": bar }
  #
  # then the string "\":" would be the value of this label_end. It's useful for
  # determining the type of quote being used by the label.
  def on_label_end(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@label_end,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # lambda is a parser event that represents using a "stabby" lambda
  # literal. It accepts as arguments a params event that represents any
  # parameters to the lambda and a stmts event that represents the
  # statements inside the lambda.
  #
  # It can be wrapped in either {..} or do..end so we look for either of
  # those combinations to get our bounds.
  def on_lambda(params, stmts)
    beging = find_scanner_event(:@tlambda)

    if event = find_scanner_event(:@tlambeg, consume: false)
      opening = scanner_events.delete(event)
      closing = find_scanner_event(:@rbrace)
    else
      opening = find_scanner_event(:@kw, 'do')
      closing = find_scanner_event(:@kw, 'end')
    end

    stmts.bind(opening[:ec], closing[:sc])

    {
      type: :lambda,
      body: [params, stmts],
      sl: beging[:sl],
      sc: beging[:sc],
      el: closing[:el],
      ec: closing[:ec]
    }
  end

  # lbrace is a scanner event representing the use of a left brace, i.e., "{".
  def on_lbrace(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@lbrace,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # lbracket is a scanner event representing the use of a left bracket, i.e.,
  # "[".
  def on_lbracket(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@lbracket,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # lparen is a scanner event representing the use of a left parenthesis, i.e.,
  # "(".
  def on_lparen(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@lparen,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # magic_comment is a scanner event that represents the use of a pragma at the
  # beginning of the file. Usually it will inside something like
  # frozen_string_literal (the key) with a value of true (the value). Both
  # children come is a string literals. We're going to leave these alone as they
  # come in all kinds of shapes and sizes.
  #
  #     def on_magic_comment(key, value)
  #       @magic_comment = { value: " #{key}: #{value}" }
  #     end

  # massign is a parser event that is a parent node of any kind of multiple
  # assignment. This includes splitting out variables on the left like:
  #
  #     a, b, c = foo
  #
  # as well as splitting out variables on the right, as in:
  #
  #     foo = a, b, c
  #
  # Both sides support splats, as well as variables following them. There's
  # also slightly odd behavior that you can achieve with the following:
  #
  #     a, = foo
  #
  # In this case a would receive only the first value of the foo enumerable,
  # in which case we need to explicitly track the comma and add it onto the
  # child node.
  def on_massign(left, right)
    left[:comma] = true if source[left[:ec]...right[:sc]].strip.start_with?(',')

    {
      type: :massign,
      body: [left, right],
      sl: left[:sl],
      sc: left[:sc],
      el: right[:el],
      ec: right[:ec]
    }
  end

  # method_add_arg is a parser event that represents a method call with
  # arguments and parentheses. It accepts as arguments the method being called
  # and the arg_paren event that contains the arguments to the method.
  def on_method_add_arg(fcall, arg_paren)
    # You can hit this if you are passing no arguments to a method that ends in
    # a question mark. Because it knows it has to be a method and not a local
    # variable. In that case we can just use the location information straight
    # from the fcall.
    if arg_paren[:type] == :args
      return fcall.merge(type: :method_add_arg, body: [fcall, arg_paren])
    end

    {
      type: :method_add_arg,
      body: [fcall, arg_paren],
      sl: fcall[:sl],
      sc: fcall[:sc],
      el: arg_paren[:el],
      ec: arg_paren[:ec]
    }
  end

  # method_add_block is a parser event that represents a method call with a
  # block argument. It accepts as arguments the method being called and the
  # block event.
  def on_method_add_block(method_add_arg, block)
    {
      type: :method_add_block,
      body: [method_add_arg, block],
      sl: method_add_arg[:sl],
      sc: method_add_arg[:sc],
      el: block[:el],
      ec: block[:ec]
    }
  end

  # An mlhs_new is a parser event that represents the beginning of the left
  # side of a multiple assignment. It is followed by any number of mlhs_add
  # nodes that each represent another variable being assigned.
  def on_mlhs_new
    {
      type: :mlhs,
      body: [],
      sl: lineno,
      sc: char_pos,
      el: lineno,
      ec: char_pos
    }
  end

  # An mlhs_add is a parser event that represents adding another variable
  # onto a list of assignments. It accepts as arguments the parent mlhs node
  # as well as the part that is being added to the list.
  def on_mlhs_add(mlhs, part)
    if mlhs[:body].empty?
      part.merge(type: :mlhs, body: [part])
    else
      mlhs.merge!(body: mlhs[:body] << part, el: part[:el], ec: part[:ec])
    end
  end

  # An mlhs_add_post is a parser event that represents adding another set of
  # variables onto a list of assignments after a splat variable. It accepts
  # as arguments the previous mlhs_add_star node that represented the splat
  # as well another mlhs node that represents all of the variables after the
  # splat.
  def on_mlhs_add_post(mlhs_add_star, mlhs)
    mlhs_add_star.merge(
      type: :mlhs_add_post,
      body: [mlhs_add_star, mlhs],
      el: mlhs[:el],
      ec: mlhs[:ec]
    )
  end

  # An mlhs_add_star is a parser event that represents a splatted variable
  # inside of a multiple assignment on the left hand side. It accepts as
  # arguments the parent mlhs node as well as the part that represents the
  # splatted variable.
  def on_mlhs_add_star(mlhs, part)
    beging = find_scanner_event(:@op, '*')
    ending = part || beging

    {
      type: :mlhs_add_star,
      body: [mlhs, part],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # An mlhs_paren is a parser event that represents parentheses being used
  # to deconstruct values in a multiple assignment on the left hand side. It
  # accepts as arguments the contents of the inside of the parentheses,
  # which is another mlhs node.
  def on_mlhs_paren(contents)
    beging = find_scanner_event(:@lparen)
    ending = find_scanner_event(:@rparen)

    if source[beging[:ec]...ending[:sc]].strip.end_with?(',')
      contents[:comma] = true
    end

    {
      type: :mlhs_paren,
      body: [contents],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # module is a parser event that represents defining a module. It accepts
  # as arguments the name of the module and the bodystmt event that
  # represents the statements evaluated within the context of the module.
  def on_module(const, bodystmt)
    beging = find_scanner_event(:@kw, 'module')
    ending = find_scanner_event(:@kw, 'end')

    bodystmt.bind(find_next_statement_start(const[:ec]), ending[:sc])

    {
      type: :module,
      body: [const, bodystmt],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # An mrhs_new is a parser event that represents the beginning of a list of
  # values that are being assigned within a multiple assignment node. It can
  # be followed by any number of mrhs_add nodes that we'll build up into an
  # array body.
  def on_mrhs_new
    {
      type: :mrhs,
      body: [],
      sl: lineno,
      sc: char_pos,
      el: lineno,
      ec: char_pos
    }
  end

  # An mrhs_add is a parser event that represents adding another value onto
  # a list on the right hand side of a multiple assignment.
  def on_mrhs_add(mrhs, part)
    if mrhs[:body].empty?
      part.merge(type: :mrhs, body: [part])
    else
      mrhs.merge!(body: mrhs[:body] << part, el: part[:el], ec: part[:ec])
    end
  end

  # An mrhs_add_star is a parser event that represents using the splat
  # operator to expand out a value on the right hand side of a multiple
  # assignment.
  def on_mrhs_add_star(mrhs, part)
    beging = find_scanner_event(:@op, '*')
    ending = part || beging

    {
      type: :mrhs_add_star,
      body: [mrhs, part],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # An mrhs_new_from_args is a parser event that represents the shorthand
  # of a multiple assignment that allows you to assign values using just
  # commas as opposed to assigning from an array. For example, in the
  # following segment the right hand side of the assignment would trigger
  # this event:
  #
  #     foo = 1, 2, 3
  #
  def on_mrhs_new_from_args(args)
    args.merge(type: :mrhs_new_from_args, body: [args])
  end

  # next is a parser event that represents using the next keyword. It
  # accepts as an argument an args or args_add_block event that contains all
  # of the arguments being passed to the next.
  def on_next(args_add_block)
    find_scanner_event(:@kw, 'next').merge!(
      type: :next,
      body: [args_add_block],
      el: args_add_block[:el],
      ec: args_add_block[:ec]
    )
  end

  # nl is a scanner event representing a newline in the source. As you can
  # imagine, it will typically get triggered quite a few times. We don't need to
  # track this event in the AST that we're generating, so we're not going to
  # define an explicit handler for it.
  #
  #     def on_nl(value)
  #       value
  #     end

  # nokw_param is a parser event that represents the use of the special 2.7+
  # syntax to indicate a method should take no additional keyword arguments. For
  # example in the following snippet:
  #
  #     def foo(**nil) end
  #
  # this is saying that foo should not accept any keyword arguments. Its value
  # is always nil. We don't need to track this event in the AST that we're
  # generating, so we're not going to define an explicit handler for it.
  #
  #     def on_nokw_param(value)
  #       value
  #     end

  # op is a scanner event representing an operator literal in the source. For
  # example, in the following snippet:
  #
  #     1 + 2
  #
  # the + sign is an operator.
  def on_op(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@op,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # opassign is a parser event that represents assigning something to a
  # variable or constant using an operator like += or ||=. It accepts as
  # arguments the left side of the expression before the operator, the
  # operator itself, and the right side of the expression.
  def on_opassign(left, oper, right)
    left.merge(
      type: :opassign,
      body: [left, oper, right],
      el: right[:el],
      ec: right[:ec]
    )
  end

  # operator_ambiguous is a parser event that represents when the parsers sees
  # an operator as ambiguous. For example, in the following snippet:
  #
  #     foo %[]
  #
  # the question becomes if the percent sign is being used as a method call or
  # if it's the start of a string literal. We don't need to track this event in
  # the AST that we're generating, so we're not going to define an explicit
  # handler for it.
  #
  #     def on_operator_ambiguous(value)
  #       value
  #     end

  # params is a parser event that represents defining parameters on a
  # method. They have a somewhat interesting structure in that they are an
  # array of arrays where the position in the top-level array indicates the
  # type of param and the subarray is the list of parameters of that type.
  # We therefore have to flatten them down to get to the location.
  def on_params(*types)
    flattened = types.flatten(2).select { |type| type.is_a?(Hash) }
    location =
      if flattened.any?
        {
          sl: flattened[0][:sl],
          sc: flattened[0][:sc],
          el: flattened[-1][:el],
          ec: flattened[-1][:ec]
        }
      else
        { sl: lineno, sc: char_pos, el: lineno, ec: char_pos }
      end

    location.merge!(type: :params, body: types)
  end

  # A paren is a parser event that represents using parentheses pretty much
  # anywhere in a Ruby program. It accepts as arguments the contents, which
  # can be either params or statements.
  def on_paren(contents)
    lparen = find_scanner_event(:@lparen)
    rparen = find_scanner_event(:@rparen)

    if contents && contents[:type] == :params
      contents.merge!(
        sc: find_next_statement_start(lparen[:ec]),
        ec: rparen[:sc]
      )
    end

    {
      type: :paren,
      lparen: lparen,
      body: [contents],
      sl: lparen[:sl],
      sc: lparen[:sc],
      el: rparen[:el],
      ec: rparen[:ec]
    }
  end

  # If we encounter a parse error, just immediately bail out so that our runner
  # can catch it.
  def on_parse_error(error, *)
    raise ParserError.new(error, lineno, column)
  end
  alias on_alias_error on_parse_error
  alias on_assign_error on_parse_error
  alias on_class_name_error on_parse_error
  alias on_param_error on_parse_error

  # period is a scanner event that represents the use of the period operator. It
  # is usually found in method calls.
  def on_period(value)
    start_line = lineno
    start_char = char_pos

    {
      type: :@period,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }
  end

  # The program node is the very top of the AST. Here we'll attach all of
  # the comments that we've gathered up over the course of parsing the
  # source string. We'll also attach on the __END__ content if there was
  # some found at the end of the source string.
  def on_program(stmts)
    range = { sl: 1, el: lines.length, sc: 0, ec: source.length }

    stmts[:body] << @__end__ if @__end__
    stmts.bind(0, source.length)

    range.merge(type: :program, body: [stmts], comments: @comments)
  end

  # qsymbols_beg is a scanner event that represents the beginning of a symbol
  # literal array. For example in the following snippet:
  #
  #     %i[foo bar baz]
  #
  # a qsymbols_beg would be triggered with the value of "%i[".
  def on_qsymbols_beg(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@qsymbols_beg,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # qsymbols_new is a parser event that represents the beginning of a symbol
  # literal array, like %i[one two three]. It can be followed by any number
  # of qsymbols_add events, which we'll append onto an array body.
  def on_qsymbols_new
    find_scanner_event(:@qsymbols_beg).merge!(type: :qsymbols, body: [])
  end

  # qsymbols_add is a parser event that represents an element inside of a
  # symbol literal array like %i[one two three]. It accepts as arguments the
  # parent qsymbols node as well as a tstring_content scanner event
  # representing the bare words.
  def on_qsymbols_add(qsymbols, tstring_content)
    qsymbols.merge!(
      body: qsymbols[:body] << tstring_content,
      el: tstring_content[:el],
      ec: tstring_content[:ec]
    )
  end

  # qwords_beg is a scanner event that represents the beginning of a word
  # literal array. For example in the following snippet:
  #
  #     %w[foo bar baz]
  #
  # a qwords_beg would be triggered with the value of "%w[".
  def on_qwords_beg(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@qwords_beg,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # qwords_new is a parser event that represents the beginning of a string
  # literal array, like %w[one two three]. It can be followed by any number
  # of qwords_add events, which we'll append onto an array body.
  def on_qwords_new
    find_scanner_event(:@qwords_beg).merge!(type: :qwords, body: [])
  end

  # qsymbols_add is a parser event that represents an element inside of a
  # symbol literal array like %i[one two three]. It accepts as arguments the
  # parent qsymbols node as well as a tstring_content scanner event
  # representing the bare words.
  def on_qwords_add(qwords, tstring_content)
    qwords.merge!(
      body: qwords[:body] << tstring_content,
      el: tstring_content[:el],
      ec: tstring_content[:ec]
    )
  end

  # rational is a scanner event that represents a rational number literal.
  def on_rational(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@rational,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # rbrace is a scanner event that represents the use of a right brace, i.e.,
  # "}".
  def on_rbrace(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@rbrace,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # rbracket is a scanner event that represents the use of a right bracket,
  # i.e., "]".
  def on_rbracket(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@rbracket,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # redo is a parser event that represents the bare redo keyword. It has no
  # body as it accepts no arguments.
  def on_redo
    find_scanner_event(:@kw, 'redo').merge!(type: :redo)
  end

  # regexp_add is a parser event that represents a piece of a regular expression
  # body. It accepts as arguments the parent regexp node as well as a
  # tstring_content scanner event representing string content, a
  # string_embexpr parser event representing interpolated content, or a
  # string_dvar parser event representing an interpolated variable.
  def on_regexp_add(regexp, piece)
    regexp.merge!(
      body: regexp[:body] << piece,
      el: regexp[:el],
      ec: regexp[:ec]
    )
  end

  # regexp_beg is a scanner event that represents the start of a regular
  # expression. It can take a couple of forms since regexp can either start with
  # a forward slash or a %r.
  def on_regexp_beg(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@regexp_beg,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # regexp_end is a scanner event that represents the end of a regular
  # expression. It will contain the closing brace or slash, as well as any flags
  # being passed to the regexp.
  def on_regexp_end(value)
    start_line = lineno
    start_char = char_pos

    {
      type: :@regexp_end,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }
  end

  # regexp_literal is a parser event that represents a regular expression.
  # It accepts as arguments a regexp node which is a built-up array of
  # pieces that go into the regexp content, as well as the ending used to
  # close out the regexp which includes any modifiers.
  def on_regexp_literal(regexp, ending)
    regexp.merge!(
      type: :regexp_literal,
      ending: ending[:body],
      el: ending[:el],
      ec: ending[:ec]
    )
  end

  # regexp_new is a parser event that represents the beginning of a regular
  # expression literal, like /foo/. It can be followed by any number of
  # regexp_add events, which we'll append onto an array body.
  def on_regexp_new
    beging = find_scanner_event(:@regexp_beg)
    beging.merge!(type: :regexp, body: [], beging: beging[:body])
  end

  # rescue is a special kind of node where you have a rescue chain but it
  # doesn't really have all of the information that it needs in order to
  # determine its ending. Therefore it relies on its parent bodystmt node to
  # report its ending to it.
  class Rescue < Node
    def bind_end(ec)
      value.merge!(ec: ec)

      stmts = value[:body][1]
      consequent = value[:body][2]

      if consequent
        consequent.bind_end(ec)
        stmts.bind_end(consequent[:sc])
      else
        stmts.bind_end(ec)
      end
    end
  end

  # rescue is a parser event that represents the use of the rescue keyword
  # inside of a bodystmt.
  def on_rescue(exceptions, variable, stmts, consequent)
    beging = find_scanner_event(:@kw, 'rescue')
    exceptions = exceptions[0] if exceptions.is_a?(Array)

    last_node = variable || exceptions || beging
    stmts.bind(find_next_statement_start(last_node[:ec]), char_pos)

    # We add an additional inner node here that ripper doesn't provide so that
    # we have a nice place to attach inline comment. But we only need it if we
    # have an exception or a variable that we're rescuing.
    rescue_ex =
      if exceptions || variable
        {
          type: :rescue_ex,
          body: [exceptions, variable],
          sl: beging[:sl],
          sc: beging[:ec] + 1,
          el: last_node[:el],
          ec: last_node[:ec]
        }
      end

    Rescue.new(
      self,
      beging.merge!(
        type: :rescue,
        body: [rescue_ex, stmts, consequent],
        el: lineno,
        ec: char_pos
      )
    )
  end

  # rescue_mod represents the modifier form of a rescue clause. It accepts as
  # arguments the statement that may raise an error and the value that should
  # be used if it does.
  def on_rescue_mod(statement, rescued)
    find_scanner_event(:@kw, 'rescue')

    {
      type: :rescue_mod,
      body: [statement, rescued],
      sl: statement[:sl],
      sc: statement[:sc],
      el: rescued[:el],
      ec: rescued[:ec]
    }
  end

  # rest_param is a parser event that represents defining a parameter in a
  # method definition that accepts all remaining positional parameters. It
  # accepts as an argument an optional identifier for the parameter. If it
  # is omitted, then we're just using the plain operator.
  def on_rest_param(ident)
    oper = find_scanner_event(:@op, '*')
    return oper.merge!(type: :rest_param, body: [nil]) unless ident

    oper.merge!(
      type: :rest_param,
      body: [ident],
      el: ident[:el],
      ec: ident[:ec]
    )
  end

  # retry is a parser event that represents the bare retry keyword. It has
  # no body as it accepts no arguments.
  def on_retry
    find_scanner_event(:@kw, 'retry').merge!(type: :retry)
  end

  # return is a parser event that represents using the return keyword with
  # arguments. It accepts as an argument an args_add_block event that
  # contains all of the arguments being passed.
  def on_return(args_add_block)
    find_scanner_event(:@kw, 'return').merge!(
      type: :return,
      body: [args_add_block],
      el: args_add_block[:el],
      ec: args_add_block[:ec]
    )
  end

  # return0 is a parser event that represents the bare return keyword. It
  # has no body as it accepts no arguments. This is as opposed to the return
  # parser event, which is the version where you're returning one or more
  # values.
  def on_return0
    find_scanner_event(:@kw, 'return').merge!(type: :return0)
  end

  # rparen is a scanner event that represents the use of a right parenthesis,
  # i.e., ")".
  def on_rparen(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@rparen,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # sclass is a parser event that represents a block of statements that
  # should be evaluated within the context of the singleton class of an
  # object. It's frequently used to define singleton methods. It looks like
  # the following example:
  #
  #     class << self do foo end
  #               │       │
  #               │       └> bodystmt
  #               └> target
  #
  def on_sclass(target, bodystmt)
    beging = find_scanner_event(:@kw, 'class')
    ending = find_scanner_event(:@kw, 'end')

    bodystmt.bind(find_next_statement_start(target[:ec]), ending[:sc])

    {
      type: :sclass,
      body: [target, bodystmt],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # semicolon is a scanner event that represents the use of a semicolon in the
  # source. We don't need to track this event in the AST that we're generating,
  # so we're not going to define an explicit handler for it.
  #
  #     def on_semicolon(value)
  #       value
  #     end

  # sp is a scanner event that represents the use of a space in the source. As
  # you can imagine, this event gets triggered quite often. We don't need to
  # track this event in the AST that we're generating, so we're not going to
  # define an explicit handler for it.
  #
  #     def on_sp(value)
  #       value
  #     end

  # stmts_add is a parser event that represents a single statement inside a
  # list of statements within any lexical block. It accepts as arguments the
  # parent stmts node as well as an stmt which can be any expression in
  # Ruby.
  def on_stmts_add(stmts, stmt)
    stmts << stmt
  end

  # Everything that has a block of code inside of it has a list of statements.
  # Normally we would just track those as a node that has an array body, but we
  # have some special handling in order to handle empty statement lists. They
  # need to have the right location information, so all of the parent node of
  # stmts nodes will report back down the location information. We then
  # propagate that onto void_stmt nodes inside the stmts in order to make sure
  # all comments get printed appropriately.
  class Stmts < Node
    def bind(sc, ec)
      value.merge!(sc: sc, ec: ec)

      if value[:body][0][:type] == :void_stmt
        value[:body][0].merge!(sc: sc, ec: sc)
      end

      attach_comments(sc, ec)
    end

    def bind_end(ec)
      value.merge!(ec: ec)
    end

    def <<(statement)
      if value[:body].any?
        value.merge!(statement.slice(:el, :ec))
      else
        value.merge!(statement.slice(:sl, :el, :sc, :ec))
      end

      value[:body] << statement
      self
    end

    private

    def attach_comments(sc, ec)
      attachable =
        parser.comments.select do |comment|
          comment[:type] == :@comment && !comment[:inline] &&
            sc <= comment[:sc] && ec >= comment[:ec] &&
            !comment[:value].include?('prettier-ignore')
        end

      return if attachable.empty?

      parser.comments -= attachable
      value[:body] = (value[:body] + attachable).sort_by! { |node| node[:sc] }
    end
  end

  # stmts_new is a parser event that represents the beginning of a list of
  # statements within any lexical block. It can be followed by any number of
  # stmts_add events, which we'll append onto an array body.
  def on_stmts_new
    Stmts.new(
      self,
      type: :stmts,
      body: [],
      sl: lineno,
      el: lineno,
      sc: char_pos,
      ec: char_pos
    )
  end

  # string_add is a parser event that represents a piece of a string. It
  # could be plain @tstring_content, string_embexpr, or string_dvar nodes.
  # It accepts as arguments the parent string node as well as the additional
  # piece of the string.
  def on_string_add(string, piece)
    string.merge!(body: string[:body] << piece, el: piece[:el], ec: piece[:ec])
  end

  # string_concat is a parser event that represents concatenating two
  # strings together using a backward slash, as in the following example:
  #
  #     'foo' \
  #       'bar'
  #
  def on_string_concat(left, right)
    {
      type: :string_concat,
      body: [left, right],
      sl: left[:sl],
      sc: left[:sc],
      el: right[:el],
      ec: right[:ec]
    }
  end

  # string_content is a parser event that represents the beginning of the
  # contents of a string, which will either be embedded inside of a
  # string_literal or a dyna_symbol node. It will have an array body so that
  # we can build up a list of @tstring_content, string_embexpr, and
  # string_dvar nodes.
  def on_string_content
    {
      type: :string,
      body: [],
      sl: lineno,
      el: lineno,
      sc: char_pos,
      ec: char_pos
    }
  end

  # string_dvar is a parser event that represents a very special kind of
  # interpolation into string. It allows you to take an instance variable,
  # class variable, or global variable and omit the braces when
  # interpolating. For example, if you wanted to interpolate the instance
  # variable @foo into a string, you could do "#@foo".
  def on_string_dvar(var_ref)
    find_scanner_event(:@embvar).merge!(
      type: :string_dvar,
      body: [var_ref],
      el: var_ref[:el],
      ec: var_ref[:ec]
    )
  end

  # string_embexpr is a parser event that represents interpolated content.
  # It can go a bunch of different parent nodes, including regexp, strings,
  # xstrings, heredocs, dyna_symbols, etc. Basically it's anywhere you see
  # the #{} construct.
  def on_string_embexpr(stmts)
    beging = find_scanner_event(:@embexpr_beg)
    ending = find_scanner_event(:@embexpr_end)

    stmts.bind(beging[:ec], ending[:sc])

    {
      type: :string_embexpr,
      body: [stmts],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # String literals are either going to be a normal string or they're going
  # to be a heredoc if we've just closed a heredoc.
  def on_string_literal(string)
    heredoc = @heredocs[-1]

    if heredoc && heredoc[:ending]
      @heredocs.pop.merge!(body: string[:body])
    else
      beging = find_scanner_event(:@tstring_beg)
      ending = find_scanner_event(:@tstring_end)

      {
        type: :string_literal,
        body: string[:body],
        quote: beging[:body],
        sl: beging[:sl],
        sc: beging[:sc],
        el: ending[:el],
        ec: ending[:ec]
      }
    end
  end

  # A super is a parser event that represents using the super keyword with
  # any number of arguments. It can optionally use parentheses (represented
  # by an arg_paren node) or just skip straight to the arguments (with an
  # args_add_block node).
  def on_super(contents)
    find_scanner_event(:@kw, 'super').merge!(
      type: :super,
      body: [contents],
      el: contents[:el],
      ec: contents[:ec]
    )
  end

  # symbeg is a scanner event that represents the beginning of a symbol literal.
  # In most cases it will contain just ":" as in the value, but if its a dynamic
  # symbol being defined it will contain ":'" or ":\"".
  def on_symbeg(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@symbeg,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # A symbol is a parser event that immediately descends from a symbol
  # literal and contains an ident representing the contents of the symbol.
  def on_symbol(ident)
    # When ripper is lexing source text, it turns symbols into keywords if their
    # contents match, which will mess up the location information of all of our
    # other nodes. So for example instead of { type: :@ident, body: "class" }
    # you would instead get { type: :@kw, body: "class" }.
    #
    # In order to take care of this, we explicitly delete this scanner event
    # from the stack to make sure it doesn't screw things up.
    scanner_events.pop

    ident.merge(type: :symbol, body: [ident])
  end

  # A symbol_literal represents a symbol in the system with no interpolation
  # (as opposed to a dyna_symbol). As its only argument it accepts either a
  # symbol node (for most cases) or an ident node (in the case that we're
  # using bare words, as in an alias node like alias foo bar).
  def on_symbol_literal(contents)
    if scanner_events[-1] == contents
      contents.merge(type: :symbol_literal, body: [contents])
    else
      beging = find_scanner_event(:@symbeg)
      contents.merge!(type: :symbol_literal, sc: beging[:sc])
    end
  end

  # symbols_beg is a scanner event that represents the start of a symbol literal
  # array with interpolation. For example, in the following snippet:
  #
  #     %I[foo bar baz]
  #
  # symbols_beg would be triggered with the value of "%I".
  def on_symbols_beg(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@symbols_beg,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # symbols_new is a parser event that represents the beginning of a symbol
  # literal array that accepts interpolation, like %I[one #{two} three]. It
  # can be followed by any number of symbols_add events, which we'll append
  # onto an array body.
  def on_symbols_new
    find_scanner_event(:@symbols_beg).merge!(type: :symbols, body: [])
  end

  # symbols_add is a parser event that represents an element inside of a
  # symbol literal array that accepts interpolation, like
  # %I[one #{two} three]. It accepts as arguments the parent symbols node as
  # well as a word_add parser event.
  def on_symbols_add(symbols, word_add)
    symbols.merge!(
      body: symbols[:body] << word_add,
      el: word_add[:el],
      ec: word_add[:ec]
    )
  end

  # tlambda is a scanner event that represents the beginning of a lambda
  # literal. It always has the value of "->".
  def on_tlambda(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@tlambda,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # tlambeg is a scanner event that represents the beginning of the body of a
  # lambda literal. It always has the value of "{".
  def on_tlambeg(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@tlambeg,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # A top_const_field is a parser event that is always the child of some
  # kind of assignment. It represents when you're assigning to a constant
  # that is being referenced at the top level. For example:
  #
  #     ::X = 1
  #
  def on_top_const_field(const)
    beging = find_colon2_before(const)
    const.merge(
      type: :top_const_field,
      body: [const],
      sl: beging[:sl],
      sc: beging[:sc]
    )
  end

  # A top_const_ref is a parser event that is a very similar to
  # top_const_field except that it is not involved in an assignment. It
  # looks like the following example:
  #
  #     ::X
  #
  def on_top_const_ref(const)
    beging = find_colon2_before(const)
    const.merge(
      type: :top_const_ref,
      body: [const],
      sl: beging[:sl],
      sc: beging[:sc]
    )
  end

  # tstring_beg is a scanner event that represents the beginning of a string
  # literal. It can represent either of the quotes for its value, or it can have
  # a %q/%Q with delimiter.
  def on_tstring_beg(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@tstring_beg,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # tstring_content is a scanner event that represents plain characters inside
  # of a string, heredoc, xstring, or regexp. Like comments, we need to force
  # the encoding here so JSON doesn't break.
  def on_tstring_content(value)
    start_line = lineno
    start_char = char_pos

    {
      type: :@tstring_content,
      body: value.force_encoding('UTF-8'),
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }
  end

  # tstring_end is a scanner event that represents the end of a string literal.
  # It can either contain quotes, or it can have the end delimiter of a %q/%Q
  # literal.
  def on_tstring_end(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@tstring_end,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # A unary node represents a unary method being called on an expression, as
  # in !, ~, or not. We have somewhat special handling of the not operator
  # since if it has parentheses they don't get reported as a paren node for
  # some reason.
  def on_unary(oper, value)
    if oper == :not
      node = find_scanner_event(:@kw, 'not')

      paren = source[node[:ec]...value[:sc]].include?('(')
      ending = paren ? find_scanner_event(:@rparen) : value

      node.merge!(
        type: :unary,
        oper: oper,
        body: [value],
        el: ending[:el],
        ec: ending[:ec],
        paren: paren
      )
    else
      # Special case instead of using find_scanner_event here. It turns out that
      # if you have a range that goes from a negative number to a negative
      # number then you can end up with a .. or a ... that's higher in the
      # stack. So we need to explicitly disallow those operators.
      index =
        scanner_events.rindex do |scanner_event|
          scanner_event[:type] == :@op && scanner_event[:sc] < value[:sc] &&
            !%w[.. ...].include?(scanner_event[:body])
        end

      beging = scanner_events.delete_at(index)
      beging.merge!(
        type: :unary,
        oper: oper[0],
        body: [value],
        el: value[:el],
        ec: value[:ec]
      )
    end
  end

  # undef nodes represent using the keyword undef. It accepts as an argument
  # an array of symbol_literal nodes that represent each message that the
  # user is attempting to undefine. We use the keyword to get the beginning
  # location and the last symbol to get the ending.
  def on_undef(symbol_literals)
    last = symbol_literals.last

    find_scanner_event(:@kw, 'undef').merge!(
      type: :undef,
      body: symbol_literals,
      el: last[:el],
      ec: last[:ec]
    )
  end

  # unless is a parser event that represents the first clause in an unless
  # chain. It accepts as arguments the predicate of the unless, the
  # statements that are contained within the unless clause, and the optional
  # consequent clause.
  def on_unless(predicate, stmts, consequent)
    beging = find_scanner_event(:@kw, 'unless')
    ending = consequent || find_scanner_event(:@kw, 'end')

    stmts.bind(predicate[:ec], ending[:sc])

    {
      type: :unless,
      body: [predicate, stmts, consequent],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # unless_mod is a parser event that represents the modifier form of an
  # unless statement. It accepts as arguments the predicate of the unless
  # and the statement that are contained within the unless clause.
  def on_unless_mod(predicate, statement)
    find_scanner_event(:@kw, 'unless')

    {
      type: :unless_mod,
      body: [predicate, statement],
      sl: statement[:sl],
      sc: statement[:sc],
      el: predicate[:el],
      ec: predicate[:ec]
    }
  end

  # until is a parser event that represents an until loop. It accepts as
  # arguments the predicate to the until and the statements that are
  # contained within the until clause.
  def on_until(predicate, stmts)
    beging = find_scanner_event(:@kw, 'until')
    ending = find_scanner_event(:@kw, 'end')

    # Consume the do keyword if it exists so that it doesn't get confused for
    # some other block
    do_event = find_scanner_event(:@kw, 'do', consume: false)
    if do_event && do_event[:sc] > predicate[:ec] && do_event[:ec] < ending[:sc]
      scanner_events.delete(do_event)
    end

    stmts.bind(predicate[:ec], ending[:sc])

    {
      type: :until,
      body: [predicate, stmts],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # until_mod is a parser event that represents the modifier form of an
  # until loop. It accepts as arguments the predicate to the until and the
  # statement that is contained within the until loop.
  def on_until_mod(predicate, statement)
    find_scanner_event(:@kw, 'until')

    {
      type: :until_mod,
      body: [predicate, statement],
      sl: statement[:sl],
      sc: statement[:sc],
      el: predicate[:el],
      ec: predicate[:ec]
    }
  end

  # var_alias is a parser event that represents when you're using the alias
  # keyword with global variable arguments. You can optionally use
  # parentheses with this keyword, so we either track the location
  # information based on those or the final argument to the alias method.
  def on_var_alias(left, right)
    beging = find_scanner_event(:@kw, 'alias')

    paren = source[beging[:ec]...left[:sc]].include?('(')
    ending = paren ? find_scanner_event(:@rparen) : right

    {
      type: :var_alias,
      body: [left, right],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # var_ref is a parser event that represents using either a local variable,
  # a nil literal, a true or false literal, or a numbered block variable.
  def on_var_ref(contents)
    contents.merge(type: :var_ref, body: [contents])
  end

  # var_field is a parser event that represents a variable that is being
  # assigned a value. As such, it is always a child of an assignment type
  # node. For example, in the following example foo is a var_field:
  #
  #     foo = 1
  #
  def on_var_field(ident)
    if ident
      ident.merge(type: :var_field, body: [ident])
    else
      # You can hit this pattern if you're assigning to a splat using pattern
      # matching syntax in Ruby 2.7+
      { type: :var_field, body: nil }
    end
  end

  # vcall nodes are any plain named thing with Ruby that could be either a
  # local variable or a method call. They accept as an argument the ident
  # scanner event that contains their content.
  #
  # Access controls like private, protected, and public are reported as
  # vcall nodes since they're technically method calls. We want to be able
  # add new lines around them as necessary, so here we're going to
  # explicitly track those as a different node type.
  def on_vcall(ident)
    @controls ||= %w[private protected public].freeze

    body = ident[:body]
    type =
      if @controls.include?(body) && body == lines[lineno - 1].strip
        :access_ctrl
      else
        :vcall
      end

    ident.merge(type: type, body: [ident])
  end

  # void_stmt is a special kind of parser event that represents an empty lexical
  # block of code. It often will have comments attached to it, so it requires
  # some special handling.
  def on_void_stmt
    { type: :void_stmt, sl: lineno, el: lineno, sc: char_pos, ec: char_pos }
  end

  # when is a parser event that represents another clause in a case chain.
  # It accepts as arguments the predicate of the when, the statements that
  # are contained within the else if clause, and the optional consequent
  # clause.
  def on_when(predicate, stmts, consequent)
    beging = find_scanner_event(:@kw, 'when')
    ending = consequent || find_scanner_event(:@kw, 'end')

    stmts.bind(predicate[:ec], ending[:sc])

    {
      type: :when,
      body: [predicate, stmts, consequent],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # while is a parser event that represents a while loop. It accepts as
  # arguments the predicate to the while and the statements that are
  # contained within the while clause.
  def on_while(predicate, stmts)
    beging = find_scanner_event(:@kw, 'while')
    ending = find_scanner_event(:@kw, 'end')

    # Consume the do keyword if it exists so that it doesn't get confused for
    # some other block
    do_event = find_scanner_event(:@kw, 'do', consume: false)
    if do_event && do_event[:sc] > predicate[:ec] && do_event[:ec] < ending[:sc]
      scanner_events.delete(do_event)
    end

    stmts.bind(predicate[:ec], ending[:sc])

    {
      type: :while,
      body: [predicate, stmts],
      sl: beging[:sl],
      sc: beging[:sc],
      el: ending[:el],
      ec: ending[:ec]
    }
  end

  # while_mod is a parser event that represents the modifier form of an
  # while loop. It accepts as arguments the predicate to the while and the
  # statement that is contained within the while loop.
  def on_while_mod(predicate, statement)
    find_scanner_event(:@kw, 'while')

    {
      type: :while_mod,
      body: [predicate, statement],
      sl: statement[:sl],
      sc: statement[:sc],
      el: predicate[:el],
      ec: predicate[:ec]
    }
  end

  # word_new is a parser event that represents the beginning of a word
  # within a special array literal (either strings or symbols) that accepts
  # interpolation. For example, in the following array, there are three
  # word nodes:
  #
  #     %W[one a#{two}a three]
  #
  # Each word inside that array is represented as its own node, which is in
  # terms of the parser a tree of word_new and word_add nodes. For our
  # purposes, we're going to report this as a word node and build up an
  # array body of our parts.
  def on_word_new
    { type: :word, body: [] }
  end

  # word_add is a parser event that represents a piece of a word within a
  # special array literal that accepts interpolation. It accepts as
  # arguments the parent word node as well as the additional piece of the
  # word, which can be either a @tstring_content node for a plain string
  # piece or a string_embexpr for an interpolated piece.
  def on_word_add(word, piece)
    if word[:body].empty?
      # Here we're making sure we get the correct bounds by using the
      # location information from the first piece.
      piece.merge(type: :word, body: [piece])
    else
      word.merge!(body: word[:body] << piece, el: piece[:el], ec: piece[:ec])
    end
  end

  # words_beg is a scanner event that represents the start of a word literal
  # array with interpolation. For example, in the following snippet:
  #
  #     %W[foo bar baz]
  #
  # words_beg would be triggered with the value of "%W".
  def on_words_beg(value)
    start_line = lineno
    start_char = char_pos

    node = {
      type: :@words_beg,
      body: value,
      sl: start_line,
      el: start_line,
      sc: start_char,
      ec: start_char + value.size
    }

    scanner_events << node
    node
  end

  # words_sep is a scanner event that represents the separate between two words
  # inside of a word literal array. It contains any amount of whitespace
  # characters that are used to delimit the words. For example,
  #
  #     %w[
  #       foo
  #       bar
  #       baz
  #     ]
  #
  # in the snippet above there would be two words_sep events triggered, one
  # between foo and bar and one between bar and baz. We don't need to track this
  # event in the AST that we're generating, so we're not going to define an
  # explicit handler for it.
  #
  #     def on_words_sep(value)
  #       value
  #     end

  # words_new is a parser event that represents the beginning of a string
  # literal array that accepts interpolation, like %W[one #{two} three]. It
  # can be followed by any number of words_add events, which we'll append
  # onto an array body.
  def on_words_new
    find_scanner_event(:@words_beg).merge!(type: :words, body: [])
  end

  # words_add is a parser event that represents an element inside of a
  # string literal array that accepts interpolation, like
  # %W[one #{two} three]. It accepts as arguments the parent words node as
  # well as a word_add parser event.
  def on_words_add(words, word_add)
    words.merge!(
      body: words[:body] << word_add,
      el: word_add[:el],
      ec: word_add[:ec]
    )
  end

  # xstring_new is a parser event that represents the beginning of a string
  # of commands that gets sent out to the terminal, like `ls`. It can
  # optionally include interpolation much like a regular string, so we're
  # going to build up an array body.
  #
  # If the xstring actually starts with a heredoc declaration, then we're
  # going to let heredocs continue to do their thing and instead just use
  # its location information.
  def on_xstring_new
    heredoc = @heredocs[-1]

    if heredoc && heredoc[:beging][3] = '`'
      heredoc.merge(type: :xstring, body: [])
    elsif RUBY_MAJOR <= 2 && RUBY_MINOR <= 5 && RUBY_PATCH < 7
      { type: :xstring, body: [] }
    else
      find_scanner_event(:@backtick).merge!(type: :xstring, body: [])
    end
  end

  # xstring_add is a parser event that represents a piece of a string of
  # commands that gets sent out to the terminal, like `ls`. It accepts two
  # arguments, the parent xstring node as well as the piece that is being
  # added to the string. Because it supports interpolation this is either a
  # tstring_content scanner event representing bare string content or a
  # string_embexpr representing interpolated content.
  def on_xstring_add(xstring, piece)
    xstring.merge!(
      body: xstring[:body] << piece,
      el: piece[:el],
      ec: piece[:ec]
    )
  end

  # xstring_literal is a parser event that represents a string of commands
  # that gets sent to the terminal, like `ls`. It accepts as its only
  # argument an xstring node that is a built up array representation of all
  # of the parts of the string (including the plain string content and the
  # interpolated content).
  #
  # They can also use heredocs to present themselves, as in the example:
  #
  #     <<-`SHELL`
  #       ls
  #     SHELL
  #
  # In this case we need to change the node type to be a heredoc instead of
  # an xstring_literal in order to get the right formatting.
  def on_xstring_literal(xstring)
    heredoc = @heredocs[-1]

    if heredoc && heredoc[:beging][3] = '`'
      heredoc.merge!(body: xstring[:body])
    else
      ending = find_scanner_event(:@tstring_end)
      xstring.merge!(type: :xstring_literal, el: ending[:el], ec: ending[:ec])
    end
  end

  # yield is a parser event that represents using the yield keyword with
  # arguments. It accepts as an argument an args_add_block event that
  # contains all of the arguments being passed.
  def on_yield(args_add_block)
    find_scanner_event(:@kw, 'yield').merge!(
      type: :yield,
      body: [args_add_block],
      el: args_add_block[:el],
      ec: args_add_block[:ec]
    )
  end

  # yield0 is a parser event that represents the bare yield keyword. It has
  # no body as it accepts no arguments. This is as opposed to the yield
  # parser event, which is the version where you're yielding one or more
  # values.
  def on_yield0
    find_scanner_event(:@kw, 'yield').merge!(type: :yield0)
  end

  # zsuper is a parser event that represents the bare super keyword. It has
  # no body as it accepts no arguments. This is as opposed to the super
  # parser event, which is the version where you're calling super with one
  # or more values.
  def on_zsuper
    find_scanner_event(:@kw, 'super').merge!(type: :zsuper)
  end
end

-->

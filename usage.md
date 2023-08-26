## Usage

If you'd like to see the syntax tree that Ripper generates for Ruby code, you can try Ripper in `irb`. The `::sexp_raw` method shows the syntax tree that is constructed from the scanner and parser events, while `::lex` shows the scanner events. You can also use the `::sexp` method to group events in a more readable way:

```ruby
$ irb
irb(main):001:0> require "ripper"
=> false
irb(main):002:0> pp Ripper.sexp_raw("a=7")
[:program,
 [:stmts_add,
  [:stmts_new],
  [:assign, [:var_field, [:@ident, "a", [1, 0]]], [:@int, "7", [1, 2]]]]]
irb(main):003:0> pp Ripper.lex("a=7")
[[[1, 0], :on_ident, "a", CMDARG],
 [[1, 1], :on_op, "=", BEG],
 [[1, 2], :on_int, "7", END]]
irb(main):004:0> pp Ripper.sexp("a=7")
[:program,
 [[:assign, [:var_field, [:@ident, "a", [1, 0]]], [:@int, "7", [1, 2]]]]]
```

Scanner events tend to include the location where the token starts as an array of two integers, such as `[1,2]` above. The first number is a 1-indexed line number (in this case line 1) and a 0-indexed character number within the line (for `[1,2]`, 2 means the third character of that line.)

Ripper works in an evented style internally. To use it that way, you'll usually want to inherit from the `Ripper` parent class, define some event handlers, and parse your code. Here's a very simple example using the [heredoc_beg](events#heredoc_beg) scanner event:

```ruby
require "ripper"

class HeredocFinder < Ripper
  def on_heredoc_beg(value)
    if value.start_with?("<<") && !value.start_with?("<<~")
      puts "Warning: use only indenting heredocs!"
      puts "Problem in #{filename} at location #{lineno}:#{column}"
    end
  end
end

HeredocFinder.new(DATA, __FILE__, DATA.lineno + 1).parse

__END__
puts <<EXAMPLE
  This is a multiline
  heredoc in Ruby
EXAMPLE
```

Parser events can be handed in the same way. Remember that if you return different values from handlers (like `on_args_new`/`on_args_add`), this will change the values other handlers receive (like `on_method_add_arg`.)

The Ruby parser can be complicated, and can generate a variety of nodes and structures. You should probably check the source code you care about using `Ripper.sexp_raw` and `Ripper.lex` to see what events occur for that source code. `Ripper::PARSER_EVENT_TABLE` contains a mapping of all events to their arity (number of arguments). The parser table can be useful when detecting many or all events at once, or when forwarding all unrecognised events to a different piece of code.

Here is a more complicated event-based parser example:

```ruby
require "ripper"

class CallArgListPrinter < Ripper
  attr_reader :calls

  def initialize(*)
    @calls = []
    super
  end

  private

  def on_method_add_arg(called, call_item)
    @calls << [called, call_item]
    [called, call_item]
  end

  # If you don't return a value for args_new/args_add, you won't get argument
  # data for on_method_add_arg.
  def on_args_new
    { values: [], lineno: lineno, column: column }
  end

  def on_args_add(args, arg)
    args[:values] << arg
    args
  end
end

puts CallArgListPrinter.new(DATA, __FILE__, DATA.lineno + 1).tap(&:parse).calls

__END__
method(1, 2, 3)

object.method(4, 5, 6)

# Not detected because it uses call instead of fcall and method_add_arg
object.method

# Detected but generates a significantly different AST, which confuses our simple logic
"ruby".split("u")
```

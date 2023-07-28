## Lists

Often your Ruby code will contain a variable-length list of items, like parameters to a method call, items in an array, or statements inside a block. Ripper doesn't use variable-length events. Instead, every event has a fixed number of items.

In order to keep the number of parameters fixed for each event, Ripper often creates an empty item and then adds to it. For instance, here is how array literals are parsed by `Ripper::SexpBuilder`:

```ruby
2.7.1 :001 > pp Ripper.sexp_raw("[1, 2, 3, 4, 5]")
[:program,
 [:stmts_add,
  [:stmts_new],
  [:array,
   [:args_add,
    [:args_add,
     [:args_add,
      [:args_add,
       [:args_add, [:args_new], [:@int, "1", [1, 1]]],
       [:@int, "2", [1, 4]]],
      [:@int, "3", [1, 7]]],
     [:@int, "4", [1, 10]]],
    [:@int, "5", [1, 13]]]]]]
```

Notice that [args_new](events#args_new) has no additional parameters, and the other array elements are added one at a time using [args_add](events#args_add).

Other events, such as [qsymbols_new](events#qsymbols_new)/[qsymbols_add](events#qsymbols_add) work very much like [args_new](events#args_new)/[args_add](event#args_add). Here's what [qsymbols_new](events#qsymbols_new)/[qsymbols_add](events#qsymbols_add) looks like in `Ripper::SexpBuilder`:

```ruby
2.7.1 :001 > pp Ripper.sexp_raw("%i[one two three]")
[:program,
 [:stmts_add,
  [:stmts_new],
  [:array,
   [:qsymbols_add,
    [:qsymbols_add,
     [:qsymbols_add, [:qsymbols_new], [:@tstring_content, "one", [1, 3]]],
     [:@tstring_content, "two", [1, 7]]],
    [:@tstring_content, "three", [1, 11]]]]]]
```

One of the differences between the `Ripper::SexpBuilder` and `Ripper::SexpBuilderPP` classes is that it replaces `*_new`/`*_add` chains with a Ruby array, such as [stmts_new](events#stmts_new)/[stmts_add](events#stmts_add) being replaced here:

```ruby
2.7.1 :001 > pp Ripper.sexp("[1, 2, 3, 4, 5]")
[:program,
 [[:array,
   [[:@int, "1", [1, 1]],
    [:@int, "2", [1, 4]],
    [:@int, "3", [1, 7]],
    [:@int, "4", [1, 10]],
    [:@int, "5", [1, 13]]]]]]
```

The accomplish this same behavior in your own Ripper parser, you can duplicate the approach that `Ripper::SexpBuilderPP` takes by handling `*_new` events with a new array and `*_add` methods by adding onto that array, like [here](https://github.com/ruby/ruby/blob/38d255d023373a665ce0d2622ed6e25462653a2a/ext/ripper/lib/ripper/sexp.rb#L178-L184).

The convention to use `*_new`/`*_add` events to build lists is used in many places in the Ripper parser. The complete list is given below:

* [args_new](events#args_new)/[args_add](events#args_add)/[args_add_block](events#args_add_block)/[args_add_star](events#args_add_star)/[args_forward](events#args_forward) - argument lists
* [mlhs_new](events#mlhs_new)/[mlhs_add](events#mlhs_add) - left-hand side of a multiple assignment list
* [mrhs_new](events#mrhs_new)/[mrhs_add](events#mrhs_add) - right-hand side of a multiple assignment list
* [qsymbols_new](events#qsymbols_new)/[qsymbols_add](events#qsymbols_add) - `%i` array literal parts
* [qwords_new](events#qwords_new)/[qwords_add](events#qwords_add) - `%w` array literal parts
* [regexp_new](events#regexp_new)/[regexp_add](events#regexp_add) - regular expression literal parts
* [stmts_new](events#stmts_new)/[stmts_add](events#stmts_add) - statement lists
* [string_content](events#string_content)/[string_add](events#string_add) - string literal parts
* [symbols_new](events#symbols_new)/[symbols_add](events#symbols_add) - `%I` array literal parts
* [word_new](events#word_new)/[word_add](events#word_add) - word parts within a `%W` array literal
* [words_new](events#words_new)/[words_add](events#words_add) - `%W` array literal parts
* [xstring_new](events#xstring_new)/[xstring_add](events#xstring_add) - `%x` command string literal parts

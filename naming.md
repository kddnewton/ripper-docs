## Naming

Certain nodes are named similarly which can indicate similar functionality.

### Prefixes

A couple of common prefixes you will find in the list of events include:

* `assoc*` - having to do with the contents of a hash. This includes [assoc_new](events#assoc_new) (a key/value pair in a hash), [assoc_splat](events#assoc_splat) (a splatted expression in a hash), and [assoclist_from_args](events#assoclist_from_args) (the node that wraps a list of the `assoc_*` nodes inside a hash).
* `m*` - mostly nodes that relate to mass assignment. This includes [massign](events#massign) (the overall parent node), [mlhs](events#mlhs) (the left-hand side (LHS) of a mass assignment), [mlhs_add_post](events#mlhs_add_post) (LHS of a mass assignment nodes that are after a splat), [mlhs_add_star](events#mlhs_add_star) (LHS of a mass assign nodes that are splatted), [mlhs_paren](events#mlhs_paren) (parentheses used in the LHS of a mass assign node, usually for destructuring), [mrhs](events#mrhs) (the right-hand side (RHS) of a mass assignment), [mrhs_add_star](events#mrhs_add_star) (RHS of a mass assignment that is being splatted), and [mrhs_new_from_args](events#mrhs_new_from_args) (creating an [mrhs](events#mrhs) node from a list of arguments, as in a list of rescued exceptions).

### Suffixes

A couple of common suffixes you will find in the list of events include:

* `*_add` - typically adding an element to a list. This includes events like [args_add](events#args_add) which adds onto an [args_new](events#args_new) list.
* `*field` - anything that can be assigned to. This includes [aref_field](events#aref_field) (assigning using `#[]=`), [const_path_field](events#const_path_field) (assigning to a nested constant), [field](events#field) (assigning to a value on an expression), [top_const_field](events#top_const_field) (assigning to a top-level constant), and [var_field](events#var_field) (a regular identifier being used in an assignment).
* `*_mod` - the modifier form of some keyword. This includes [if_mod](events#if_mod), [rescue_mod](events#rescue_mod), [unless_mod](events#unless_mod), [until_mod](events#until_mod), and [while_mod](events#while_mod). Each of the prefixes of those event names tells you which keyword is being used.
* `*_new` - typically creating a new list of elements. This includes events like [args_new](events#args_new) which are then followed by any number of [args_add](events#args_add) nodes.
* `*ptn` - patterns used in pattern matching. This includes [aryptn](events#aryptn) (array patterns), [fndptn](events#fndptn) (find patterns), and [hshptn](events#hshptn) (hash patterns).

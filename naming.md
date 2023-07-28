## Naming

Certain nodes are named similarly which can indicate similar functionality.

### Prefixes

A couple of common prefixes you will find in the list of events include:

* `assoc*` - having to do with the contents of a hash. This includes [assoc_new](events.md#assoc_new) (a key/value pair in a hash), [assoc_splat](events.md#assoc_splat) (a splatted expression in a hash), and [assoclist_from_args](events.md#assoclist_from_args) (the node that wraps a list of the `assoc_*` nodes inside a hash).
* `m*` - mostly nodes that relate to mass assignment. This includes [massign](events.md#massign) (the overall parent node), [mlhs_add](events.md#mlhs_add) (the left-hand side (LHS) of a mass assignment), [mlhs_add_post](events.md#mlhs_add_post) (LHS of a mass assignment nodes that are after a splat), [mlhs_add_star](events.md#mlhs_add_star) (LHS of a mass assign nodes that are splatted), [mlhs_paren](events.md#mlhs_paren) (parentheses used in the LHS of a mass assign node, usually for destructuring), [mrhs_add](events.md#mrhs_add) (the right-hand side (RHS) of a mass assignment), [mrhs_add_star](events.md#mrhs_add_star) (RHS of a mass assignment that is being splatted), and [mrhs_new_from_args](events.md#mrhs_new_from_args) (creating an [mrhs_add](events.md#mrhs_add) node from a list of arguments, as in a list of rescued exceptions).

### Suffixes

A couple of common suffixes you will find in the list of events include:

* `*_add` - typically adding an element to a list. This includes events like [args_add](events.md#args_add) which adds onto an [args_new](events.md#args_new) list.
* `*field` - anything that can be assigned to. This includes [aref_field](events.md#aref_field) (assigning using `#[]=`), [const_path_field](events.md#const_path_field) (assigning to a nested constant), [field](events.md#field) (assigning to a value on an expression), [top_const_field](events.md#top_const_field) (assigning to a top-level constant), and [var_field](events.md#var_field) (a regular identifier being used in an assignment).
* `*_mod` - the modifier form of some keyword. This includes [if_mod](events.md#if_mod), [rescue_mod](events.md#rescue_mod), [unless_mod](events.md#unless_mod), [until_mod](events.md#until_mod), and [while_mod](events.md#while_mod). Each of the prefixes of those event names tells you which keyword is being used.
* `*_new` - typically creating a new list of elements. This includes events like [args_new](events.md#args_new) which are then followed by any number of [args_add](events.md#args_add) nodes.
* `*ptn` - patterns used in pattern matching. This includes [aryptn](events.md#aryptn) (array patterns), [fndptn](events.md#fndptn) (find patterns), and [hshptn](events.md#hshptn) (hash patterns).

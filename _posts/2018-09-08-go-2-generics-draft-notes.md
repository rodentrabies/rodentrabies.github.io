---
layout: post
title:  "Go 2 generics draft notes"
date:   2018-09-08 11:36:00 +0300
categories: jekyll update
---

Last week Go team published a page with detailed [specification drafts][1] for
Go 2 generics and error handling tools. The proposal is both well-thought and
brave, since one of the main ideas is to introduce contract system akin to C++
concepts. Contracts, in Go parlance, are (possibly syntactically restricted)
named functions, parameterized both by value and by its type, which are used by
compiler to ensure some contract are held by types as stated. What's great about
contracts is that the procedure of contract checking is very simple - since it's
just a function body, compilers type-checks, compiles and then discards its body
and that's it.

But, simplicity for the compiler often comes with the cost for the user. This
post is an attempt to analyze the proposal and collect any notes I made while
reading the document. The [Syntax](#syntax) section gives few notes on the
proposed syntax, while [Semantics](#semantics) section describes... well,
semantics.




## Syntax
One of the most important *little* things about Go is a minimal and clean
syntax. Generally, proposal keeps it that way but there are a few moments that
make just a little bit too much in some places.

#### Function type parameter declarations
Since methods seem to not be having type parameters, for standalone functions
they can be declared in place of method receiver argument:


```golang
func (type T) Print(slice []T) {
    for _, v in range slice {
        fmt.Printf("%v", v)
    }
}
```

This both simplifies the `(type ...)(parameters ...)` form and is more logical
from the point of view that type parameters and regular parameters should be
clearly separated. The problem is such a syntax is a little bit ambiguous
visually, but still it is much more simple from a notation point of view than
`(type ...)(parameters ...)` form.

#### Contract as another type of type
To avoid Go 1 compatibility complexities with making contract a keyword,
contacts can be considered another type of type declaration, i.e. use syntax

```golang
type Convertible(_ To, f From) contract {}
```

This, however, may pose a semantic problem, since Go generally adheres to direct
mapping between types and memory representation of values: structs map to
sequences of named fields, interfaces map to double-pointer `(val, vtable)`
values, while contracts do not map to anything, since they are purely
compile-time constructs.




## Semantics

#### Contracts or no contracts?
Stating it this far in the post may be strange, but one of the first notes I
made about contract proposal is that I actually don't like the idea of having
function bodies that are not function bodies, which may be misleading. Imposing
restrictions on contract bodies requires extending Go's grammar and having no
restrictions whatsoever may cause eye injuries while reading contract bodies
with `if`/`for`/`defer` statements. Allowing only assignment statements and
expressions in a contract body won't prevent it from containing any logic beyond
pure declaration of a set of permitted operations, which can still be very
misleading to the reader.

Consider the following structures:

```golang
type Pair struct {
    first, second int
}

type Cons struct {
    head, tail int
}
```

Just by looking at these declarations, we can tell that they are structurally
identical and, according to Go specifications, even have the same representation
in memory. In other words these two structures are *isomorphic* and even if we
have never worked with Lisp language and haven't seen `Cons` structure before,
we can easily deduce how it behaves, since it's isomorphic to the `Pair`, a
simple structure everyone knows.

Similarly, consider these two interfaces:

```golang
type Index interface {
    Clear(i int)
    Set(i int, value string)
}

type StringMap interface {
    Insert(k string, value interface{})
    Remove(k string)
}
```

Even though this is a deliberately hard example and operations in these
interfaces are completely different, they have the same signature structure and
thus can be deemed isomorphic and understood to perform a similar tasks of
including and changing some value at a given key.

Now, consider these two contracts:

```golang
contract strseq1(x T) {
	[]byte(x)
	T([]byte{})
	len(x)
}

contract strseq2(a T) {
    var c int = len(a)
    var bs []byte = []byte(a)
    T(bs)
}
```

They declare the same set of operations, but in order to tell that you have to
place them side by side and do some thinking. It's not easy to informally check
these two contracts are isomorphic, and it's extremely hard to do this formally
(see [Graph isomorphism][2]). In other words, structs and interfaces are
declarative and simple to understand while contracts are imperative, and in
order to understand a contract, you have to follow their statements. Still, all
they do, is just declare a set of operations on some types. Go already has
interfaces for that, so a more simple approach is to add operator methods, come
up with a way to restrict overloading and just go with interfaces.

#### Contracts from interfaces
Assuming contracts are here to stay, we can compromise between contracts and
interfaces by allowing using interface names as contract names in declarations
in order to reuse trivial contract declarations, i.e. every interface
declaration should generate equivalent contract as shown below:

```golang
type ReadWrite interface { 
   Read() string
   Write(string)
}

contract ReadWrite(x T) {
    val s string = x.Read()
    x.Write(s)
}
```

This will allow using interfaces for most cases where contracts describe just a
set of methods and resort to contracts only for hard case like conversions and
operators. An important consequence of such correspondence is standard library
of contracts automatically derived from standard interfaces.

## Conclusion
Current Go generics proposal is very well-though and goes to great extent to
avoid all the problems with parametric polymorphism systems in languages like
C++ and Java. Contracts are a very interesting concept (_no_ pun intended), but
they have some semantic drawbacks that are not yet solved and may never have a
simple solution. Keeping that in mind, if there's any change to substitute them
with clean and pure old interfaces, we should do our best to try to avoid adding
them to the language in the end as, once added, such features are very hard to
get rid of.

[1]: https://go.googlesource.com/proposal/+/master/design/go2draft.md
[2]: https://en.wikipedia.org/wiki/Graph_isomorphism

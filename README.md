# Math::Symbolic

This is a Perl 6 symbolic math module. It parses, manipulates, evaluates, and
outputs mathematical expressions and relations. This module is PRE-ALPHA
quality.

This document talks almost as much about what doesn't yet exist and what could
exist, as it does about what is already there. This is quite intentional and is
meant to serve as an invitation and inspiration to prospective contributors, as
well as the beginnings of a project roadmap.

## Synopsis

TODO

    say Math::Symbolic.new("y=m*x+b").isolate("x"); # x=(y-b)/m

## Usage

### Command line

A basic command line interface is provided in the root directory of the module
(meaning it will not be installed to your PATH, for now), called "alg.p6". This
script is used for testing during development, but may become more useful. It
takes at least one positional argument: the expression/relation to work with.
If a second positional is passed, it is the name of the variable to isolate. If
any named args are passed, they are substituted into the expression for the
variables they name. Each named argument's value is parsed as an expression
itself, so it doesn't just have to be a numeric value to give the variable.

The resulting expression will be printed after applying all requested
transformations, and attempting to simplify.

### API

At the time of this writing, most of the API is too unstable to be worth
documenting, as a formal design/spec has yet to be established. Things like
tree nodes and operations are a bit vague and cumbersome, as they are
incomplete ideas at this time, and in many cases are the jumbled result of
several passes of incremental redesign already, and late-night creative
rampages. Many common uses aren't yet encapsulated in methods, some classes are
used where they probably ought to be roles, and the op tree is generated by
crawling the match tree instead of connecting actions to the grammar. This
entire module grew to a 600-line jumbled mess of globals, overlapping scopes,
and much learning, before being chopped up and re-written as a module.

To minimize exposure to the aforementioned chaos in the interim, a minimal
temporary public interface is implemented as a small collection of simple
methods in the main Math::Symbolic class.

#### .new(Str:D $expression)

Creates a new Math::Symbolic object, initialized with the tree resulting from a
parse of $expression (which may also be a relation; currently only "=" equality
is supported for relations).

#### .isolate(Str:D $var)

Arranges a relation with $var on the left and everything else on the right.  Or
attempts to. It currently only supports simple operation inversion, it can't
yet combine multiple instances of a variable in an equation, factor
polynomials, etc. If called on an equation with more than one instance of the
variable, it will isolate the first instance it encounters in the tree
structure. If called on a non-relation expression, a butterfly dies.

This is one area which the author looks forward to experimenting in. Non-
obvious solutions could be discovered automatically by traversing a network of
possible "arrangement" nodes, interconnected by "manipulation" edges.  The
network could be entirely generated automatically from the operation list, a
manipulation list (which doesn't exist yet), and a few more properties on each
operation to control which operations interact how (e.g.  commutivity and
distributivity, some of which already exists), so as to filter out logically
invalid combinations. This approach would allow simple-to-visualize geometric
solutions to otherwise complex problems.  Isolating, simplifying, or, more
generally, reaching any success criteria, boils down to the comparatively
simple problem of searching an undirected graph. The search could possibly be
guided by a more sophisticated pathfinding algorithm making use of metrics like
tree complexity/node count (for simplifying) or a measure of
similarity/difference between two arrangements (for example, to compare each
node as you traverse it with the desisred form of the result, to prioritize
searching more similar branches over more different ones).

#### .evaluate(\*%values)

Replaces all instances of variables named by the keys of the hash, with the
expressions in the values of the hash, and simplifies the result as much as
possible (see .simplify below). If the resulting expression has no variables,
this means it can be fully evaluated down to a single value.

Note that fully evaluating an equation with valid values would result in
something mostly unhelpful like "0=0" if the simplifier is smart enough.
Though in the future, when such a relation can be evaluated for truth, that
will become useful.

#### .simplify()

Makes a childish attempt to reduce the complexity of the expression by
evaluating operations on constants, removing operations on identity values (and
eventually other special cases like 0, inverse identity, etc). Also already
does a very small number of rearrangements of combinations of operations, like
transforming a/b/c into a\*c/b.

.simplify is currently always called at the end of .new, .isolate, and
.evaluate, making calling it directly pointless in most cases. An option to
disable this for performance will be added.

This method is another candidate for the network-searching discussed above
under .isolate(), or at least more extensive use of the properties of the
operations, instead of hard-coded patterns of reduction.

#### .count()

Returns the number of nodes in the expression's tree. This could be useful to
determine if an expression has been fully evaluated, or used as a crude
complexity metric.

#### .dump_tree()

Prints out the tree structure of the expression. This really should return the
string instead, and perhaps be renamed.

#### .Str()

Returns the expression re-assembled into a string by the same syntactic
constructs which govern parsing. As with all Perl 6 objects, this is also the
method which gets called when the object is coerced to a string by other means,
e.g. interpolation, context, or the ~ prefix. The .gist() method is also
handled by this routine, for easy printing of a readable result.

Passing the result of .Str() back in to .new() should always yield a
mathematically equivalent structure (exact representation may vary by some
auto-simplification), giving the same type of round-trip characteristics to
expressions that .perl() and EVAL() provide for Perl 6 objects. This allows a user
to, for instance, isolate a variable in an equation, then plug the result in to
.evaluate() for that variable in a different equation, all with the simplicity
of strings; no additional classes or APIs for the user to worry about (albeit
at a steep performance penalty).

TODO the example here won't actually work without an intervening s:s/ \w+ \= //
to convert the isolation result from an equation to a relation. Do something
about that, and generally think more about relations vs expressions, especially
as pertains to complexity of the public API. This issue is orthogonal to the
use of strings vs objects.

#### .Numeric()

Returns the expression coerced first to a string (see above), then to a number.
This will fail if the expression hasn't already been evaluated/simplified (see
further above) to a single constant value. As with all Perl 6 objects, this is
also the method which gets called when the object is coerced to a number by
other means, e.g. context or the + prefix.

## Syntax and Operations

All whitespace is currently optional. Implicit operations, e.g. multiplication
by putting two variables in a row with no infix operator, is not supported, and
likely never will be. It leads to far too many consequences, compromises,
complexities and non-determinisms, mainly in the syntax/parsing.

The available operations and syntax in order of precedence are currently:

* Terms
    * Variables
        * mixed-case alphabetic and "_" are valid in variable names
    * Values
        * optional sign (only "-" for now)
        * E notation supported
            * case insensitive
            * no restriction on value of exponent, with sign and decimal
            * subexpressions not supported (e is numeric syntax, not an op)
        * leading zeros before decimals not required
        * imaginary, complex, quaternion, etc NYI
        * vector, matrix, set, etc NYI
* Circumfix
    * () Grouping Parens
    * || Absolute Value
        * cannot invert this op for solving/manipulating, ± NYI
* Postfix
    * ! Factorial
        * syntax only, no functional implementation
* Prefix
    * - Negate (same as * -1)
* Infix Operation
    * Power
        * ^ Exponentiation, like perl's **
            * mathematical convention dictates that this operation be chained
            right-to-left, which is NYI
        * ^/ Root, x^/n is the same as x^(1/n), or ⁿ√x
            * this is temporary until there is support for √ which doesn't exist
            yet because the order of its operands is backwards from its inverse
            op, which is NYI
            * evaluates to positive only (± NYI)
            * imaginary numbers NYI, haven't even tried -1^/2
    * Scale
        * * Multiplication
        * / Division
    * Shift
        * + Addition
        * - Subtraction
* Infix Relation
    * = Equality is currently the only relation supported
        * this is really because proper relations are more-or-less NYI
    * note that relations are optional in the input string, it automatically
    detects whether it is working with an expression or a relation
        * really it just breaks if you call 'solve' on an expression ATM

## TODO

See above.

## BUGS

Many, in all likelihood. Please report them to raydiak@cyberuniverses.com. Patches graciously accepted.

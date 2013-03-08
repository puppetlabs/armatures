(ARM-2) Iteration
=================

Summary
-------

This arm proposes a solution for handling "iteration".

The proposal is based on ruby/java-8 syntax for lambda and enumerables
support adapted to the Puppet Language.

An alternative idea based on a "Unix Pipes" is also presented.
This alternative also makes use of lambda expressions.

The "ruby/java-8" like proposal has been implemented
to allow usability studies of the various options in this proposal.
At present this implementation supports all of the presented options at
the same time to allow users to experiment.

Real world examples of the proposed solution are found in [Examples](examples.md).

Goals
-----

* Provide a solution to the puppet redmine issue for 'iteration'
  [#11331 - "foreach"](http://projects.puppetlabs.com/issues/11331)
* Provide iteration suitable for iterative resource creation
* Provide support for data transformation (e.g. iterating/filtering/
  transforming arrays and hashes).
* Syntax for iteration should be forwards compatible  (migration to a version
  supporting iteration should not require changes to existing manifests. This means
  that no new keywords should be introduced as that potentially breaks existing
  logic).
* Solution should be extensible. There are many different ways to iterate and
  transform data. It should be possible to extend the core functionality in
  modules. Naturally, these should not require language changes.

Secondary / Internal Goals
--------------------------

* Provide a step in the direction of improved "separation of concerns" in
  the puppet logic chain: lexing => parsing => semantic-model => validation => evaluation.
  (With the rationale: increased quality, flexibility in introduction of new
  features, increased developer productivity).
* Provide the foundation to validate parsed logic using different sets of validation rules.
* Provide the foundation to give users better (more detailed and relevant) error messages.
* Enable selective introduction of new versions of evaluation logic (and enable possible selective
  backwards/bug compatibility evaluation options).
* Increase test coverage of logic dealing with syntax and semantics

Non Goals
---------

* Provide Closures for lambdas. This proposal does not include the ability
  to assign lambdas to variables, or to pass them as parameters to resources
  or classes.
* Provide named functions written in the Puppet Language.
* Alter call semantics to offer calls with argument to parameter binding by name
  instead of by position.
* Forward bug compatibility - changes to existing logic may solve bugs.
* Forward error reporting compatibility - some (syntax/semantic) problems may be reported differently.
* Forward grammar restriction enforcement - some expressions that were not possible because of reduce/reduce conflicts
  in the grammar are now possible (e.g. foo()[2], [1,2,3][1]). It is not a goal to maintain these restrictions artificially.
* Provide multiple implementations of validation rules (i.e. "3.1", "3.2 with iteration", "4.x" etc).
* Decide if the term "lambda" is to be used in user facing documentation/error messages, or if the term should be something
  less "computer science oriented" like "parameterized block", or simply "block". 

Success Metrics
---------------
No Success Metrics have been defined, but could in theory be based on measured reduced use of ruby functions/templates in modules
at the forge. Such measures are however difficult to implement accurately (is it doing iteration/data transformation?)
and will have a long lead time until they can be computed.

It could be of interest to base metrics on language related bugs.

Motivation
----------

This work should be done because users should not have to use Ruby / ERB templates
to solve iteration and common data transformation tasks.

The earlier ideas that a Ruby-DSL should be provided to allow users more power turned out
to be motivated largely by the need to iteratively create resources based on data/structures
obtained from external sources, or to transform such data to suitable parameter values before creating
resources.

Compared to the competition that use an internal Ruby DSL / Ruby as the description
language, the Puppet 3.1 version of the Puppet DSL simply has no support for iteration.

Extending the Puppet Language to directly support iteration and data transformation is of
value as the majority of the users are not developers well versed in Ruby, thus making
"write a function/ruby template" a task with a very steep learning curve. There is also
an impedance mismatch going from Puppet DSL to Puppet's internal advanced (a.k.a complex)
API where mistakes are easily made and enforcement of puppet language semantics is not possible.

In addition to these external benefits, the implementation also provides many internal enablers for
future improvements (see [Secondary / Internal Goals](#secondary--internal-goals).

Description
===========

This description is based on the recommended implementation (this to not overwhelm the reader
with all the various alternatives and options described and discussed
in [Alternatives and Recommendation](#alternatives-and-recommendation)).

The notion of "iteration" is implemented using functions and parameterized code blocks/lambdas.
The language itself does not have a loop concept.

> One detail about the implementation becomes important when reading the overview of the syntax as it changes
> the internal grammatical notion of statements vs. expressions. Where the description talks about "expression" it
> should be read as "statement or expression" (using the current terms). See ["Expression Based Grammar"](#expression-based-grammar)
> for more details.

> A note about terminology:  
> This proposal uses the terms "parameterized block", "lambda" and "block" to mean the same thing.

### Parameterized Code Block / Lambda

Parameterized code blocks (a.k.a lambdas/anonymous functions) are available with the following
syntax:

    | <parameters> | { <expressions> }

They may only appear after [calls](#calls).

> This restriction is due to the current internal complexities in the implementation of scope
> and difficulties in supporting general closures with this implementation. This restriction may
> be lifted in future versions.

#### Lambda Parameters

A lambda's parameters are defined between `| |´ (pipes) and consists of an optional comma separated list of parameter declarations
(using the same syntax as everywhere else in the puppet language. e.g.):

    | $foo |
    | $foo = 10 |
    | $foo, $bar |

In the (rare) event that a lambda needs no parameters, this is simply `||`.

Since calls pass arguments _by position_, parameters with default values can not appear to the left of
parameters without defaults.

> None of the proposed "iteration functions" makes use of the ability to declare default values, but the ability was
> kept to make these parameter declarations the same as everywhere else, and they have value for
> custom extensions. Default values are fully supported in the available implementation.

#### Lambda Body

The body of the Lambda is a sequence of (white-space separated) expressions, and it produces the value of the last evaluated
expression. The body's expressions are evaluated in a local inner scope of the point where the lambda 
is invoked. It is possible to set variables in this scope, but these are only visible in the lambda's scope
(and any nested inner scopes; ephemeral scopes, nested lambdas). These variables are immutable just like all other variables. 
When the lambda is invoked many times (as is the case with iteration), the block is re-initialized; variable 
values set in the previous iteration does not survive. Side effects of lambdas (such as creating a resource, or anything 
else that ends up in the catalog) takes place in evaluation order.

### Calls

This proposal introduces a new form of function call reminicent of a _method call_ using an infix `.` (dot) operator
notation. This is easier to both read and construct compared to regular nested function calls when a chain
of operations are wanted (e.g. filter and then transform).

Method call syntax:

    <receiver_expression>.<function_name>
    <receiver_expression>.<function_name>()
    <receiver_expression>.<function_name>(<arguments>)

Consider a filter operation followed by a transform - first using infix `.` notation...:

    $array.select |$x| {$x =~ /^0-9+$/}.collect |$x| { $x * 2 }
    
... then in function call notation, this would become (the much harder to read):

    collect(select($array) |$x| {$x =~ /^0-9+$/}) |$x| { $x * 2 }

Note that the existing function call syntax is unaltered (except allowing an optional lambda).

### Passing Lambdas in Calls

Lambdas are passed to calls by placing them to the right of the call.

Examples:

    each($array) |$x| { }
    slice($array, 2) |$x| { }
    
    $array.each |$x| { }
    $array.slice(2) |$x| { }
    
Although no such function is proposed in this ARM, it is possible to pass a lambda to a function that does not require
any parameters (other than the lambda). Here an imaginary custom user defined function collecting data from some source:

    our_data_for_webhosts() |$hostname, $classes| { }

> Note that lambdas can be used with all call variations except the statement-like non-parenthesized function call.
> This because it is ambiguous - consider the meaning of `a a ||{ 1 }`, is this `a(a) ||{ 1 }` or `a(a ||{ 1 })`?
> Given that none of the iteration functions require this ability and the "work around" is to use parentheses around
> the arguments a resolution of this ambiguity was not attempted.
>> Implementation Note: The non parenthesized call is the source of much of the grammars complexity and it is not possible to handle it
>> with regular racc precedence rules (not without carefully specializing rules to the point we end up with a grammar
>> like the existing). A solution may be possible, but at a high implementation cost (making a lambda an expression, which implies
>> closures, and turning lambda parameters into a binary operator of suitable precedence).


### Custom (Ruby) Function with Lambda

Implementing a function that accepts and operates on a lambda is very simple.
A lambda (if present) is always added as the last argument to a function. 

Here is an example of how a custom function (in Ruby) would deal with a lambda, checking that it is given in the call,
and then invoking it with some arguments.

    lambda = args[-1]
    raise ArgumentError, "a block is required" unless lambda.is_a? Puppet::Parser::AST::Lambda 
    ...
    args = [...]
    result = lambda.call(scope, *args)     

> This is how the "iteration functions" are also implemented.

### Functions for iteration and transformation

The proposed functions for iteration and data transformation are:

* `each` - iterate over collection
* `slice` - produces slices (chunks; pairs, triplets, ...) of a collection and optionally iterates over it
* `collect` - produces a new array by collecting each result of the given lambda
* `select` - produces a new array filtered by the given (inclusion predicate) lambda
* `reject` - produces a new array filtered by the given (rejection predicate) lambda
* `reduce` - produces a single (reduced) value from a collection of values

#### each

Applies a parameterized block to each element in a sequence of selected entries from the first
argument and returns the result returned by the last application.
  
This function takes two mandatory arguments: the first should be an Array or a Hash, and the second
a parameterized block as produced by the puppet syntax:
  
    $a.each |$x| { ... }
      
When the first argument is an Array, the parameterized block should define one or two block parameters.
For each application of the block, the next element from the array is selected, and it is passed to
the block if the block has one parameter.args If the block has two parameters, the first is the elements
index, and the second the value. The index starts from 0.
  
    $a.each |$index, $value| { ... }
      
When the first argument is a Hash, the parameterized block should define one or two parameters.
When one parameter is defined, the iteration is performed with each entry as an array of `[key, value]`,
and when two parameters are defined the iteration is performed with key and value.
  
    $a.each |$entry|       { notice "key ${$entry[0]}, value ${$entry[1]}" } 
    $a.each |$key, $value| { notice "key ${key}, value ${value}" }

#### slice

Applies a parameterized block to each _slice_ of elements in a sequence of selected entries from the first
argument and returns the first argument, or if no block is given returns a new array with a concatenation of
the slices.
    
This function takes two mandatory arguments: the first should be an Array or a Hash, and the second
the number of elements to include in each slice. The optional third argument should be a
a parameterized block as produced by the puppet syntax:
    
      |$x| { ... }
        
The parameterized block should have either one parameter (receiving an array with the slice), or the same number
of parameters as specified by the slice size (each parameter receiving its part of the slice).
In case there are fewer remaining elements than the slice size for the last slice it will contain the remaining
elements. When the block has multiple parameters, excess parameters are set to :undef for an array, and to
empty arrays for a Hash.
      
      $a.slice(2) |$first, $second| { ... }
        
When the first argument is a Hash, each key,value entry is counted as one, e.g, a slice size of 2 will produce
an array of two arrays with key, value.
    
      $a.slice(2) |$entry|          { notice "first ${$entry[0]}, second ${$entry[1]}" } 
      $a.slice(2) |$first, $second| { notice "first ${first}, second ${second}" }

When called without a block, the function produces a concatenated result of the slices.
  
      slice($[1,2,3,4,5,6], 2) # produces [[1,2], [3,4], [5,6]]

#### collect

Applies a parameterized block to each element in a sequence of entries from the first
argument and returns an array with the result of each invocation of the parameterized block.

This function takes two mandatory arguments: the first should be an Array or a Hash, and the second
a parameterized block as produced by the puppet syntax:

    $a.collect |$x| { ... }

When the first argument is an Array, the block is called with each entry in turn. When the first argument
is a hash the entry is an array with `[key, value]`.

*Examples*

    # Turns hash into array of values  
    $a.collect |$x| { $x[1] }
      
    # Turns hash into array of keys  
    $a.collect |$x| { $x[0] }

#### select

Applies a parameterized block to each element in a sequence of entries from the first
argument and returns an array with the entires for which the block evaluates to true.

This function takes two mandatory arguments: the first should be an Array or a Hash, and the second
a parameterized block as produced by the puppet syntax:

    $a.select |$x| { ... }

When the first argument is an Array, the block is called with each entry in turn. When the first argument
is a hash the entry is an array with `[key, value]`.

*Examples*

    # selects all that end with berry
    $a = ["raspberry", "blueberry", "orange"]  
    $a.select |$x| { $x =~ /berry$/ }

#### reject

Works the same way as select, but returns a new array with the entries for which the block does *not* evaluate to true.

#### reduce

Applies a parameterized block to each element in a sequence of entries from the first
argument (_the collection_) and returns the last result of the invocation of the parameterized block.

This function takes two mandatory arguments: the first should be an Array or a Hash, and the last
a parameterized block as produced by the puppet syntax:

    $a.reduce |$memo, $x| { ... }

When the first argument is an Array, the block is called with each entry in turn. When the first argument
is a hash each entry is converted to an array with `[key, value]` before being fed to the block. An optional
'start memo' value may be supplied as an argument between the array/hash and mandatory block.
  
If no 'start memo' is given, the first invocation of the parameterized block will be given the first and second
elements of the collection, and if the collection has fewer than 2 elements, the first
element is produced as the result of the reduction without invocation of the block.
  
On each subsequent invocations, the produced value of the invoked parameterized block is given as the memo in the
next invocation. 

*Examples*

    # Reduce an array  
    $a = [1,2,3]
    $a.reduce |$memo, $entry| { $memo + $entry }
    #=> 6

    # Reduce hash values  
    $a = {a => 1, b => 2, c => 3}
    $a.reduce |$memo, $entry| { [sum, $memo[1]+$entry[1]] }
    #=> [sum, 6]
   
It is possible to provide a starting 'memo' as an argument.    
  
*Examples*
    # Reduce an array  
    $a = [1,2,3]
    $a.reduce(4) |$memo, $entry| { $memo + $entry }
    #=> 10
    
    # Reduce hash values  
    $a = {a => 1, b => 2, c => 3}
    $a.reduce([na, 4]) |$memo, $entry| { [sum, $memo[1]+$entry[1]] }
    #=> [sum, 10]


Other Changes
-------------

### Changes to Evaluation of + and <<

The following additions to the existing evaluator are made:

* `Array + Array` - produces a new array being a concatenation of the right onto the left
* `Hash + Hash` - produces a new Hash being a merge of the right into the left
* `Array << Object` - produces a new Array with the right appended

It is not allowed to concatenate with anything but an array, nor merge anything but another hash.
Appending any object is allowed, when appending an Array it is not flattened.

    [1,2,3] +  [4,5,6] == [1,2,3,4,5,6]
    [1,2,3] << [4,5,6] == [1,2,3,[4,5,6]]
    [1,2,3] << 4       == [1,2,3,4]

    {a=>1, b=>2} + { c=>3 } == {a=>1, b=>2, c=>3}
    {a=>1, b=>2} + { b=>4 } == {a=>1, b=>4 }
    
    
These additions were made as it was difficult to write transformation logic without this support.
(Arguably there are functions in standard library that could help with this, but proposal
feels incomplete without also offering these).

### Expression Separator

Since the [expression based grammar](#expression-based-grammar) allows an expression to be "a statement"
and white-space is used to separate one expression from another, an additional expression separator is required
in order to deal with two corner cases. The optional separator is `;` (i.e. optional in the sense - not required when
there is no special case, but may be used).

Consider:

    $a = 2 + $x [1]

If the intention is to assign `2 + $x` to `$a` and then produce a literal array with the single value `1` then the above is
wrong as the `[]` operator has high precedence (it must be higher that most other operators to work in general arithmetic
expressions), and thus, the parsing of the example will instead attempt to access index 1 in the array assigned to `$x` and
then add 2.

Now, using the expression separator, the example will be parsed as intended:

    $a = 2 + $x; [1]
    
> Since whitespace is generally not significant in the Puppet Language, the use of an expression separator was
> considered to be a better solution than making a distinction between `$x[1]` and `$x [1]`.

The golden rule is: Use the expression separator when the following expression would alter the meaning of it's preceding
expression's last operand. This is typically only the case before a final expression producing
a literal array `[...]` or a literal hash `{...}`.  

> Note: no existing puppet logic is affected by this, as the only expression combinations that would cause
> this ambiguity are illegal in puppet 3.1 (an expression can not be a statement in 3.1).

Internal Changes
----------------
Changes that are not directly visible to a user of the puppet language.

### Changes to Scope

In order to support lambdas, there is the need to provide a classic _local scope_ where variable definitions
do not externally visible and (potentially) shadow variable in an outer scope. An implementation of a
local scope was possible by extending the existing mechanism for "ephemeral scope" (local scope is not simply
an ephemeral scope, that would not work, but it is part of the same mechanism). 

> Going forward, the scope implementation should be refactored as it is overloaded with complex behavior.
> The complexities in the current scope implementation is also the main reason why support for Closures is
> a non goal for this ARM.

### Changes to Function "type"

Functions that are of _rvalue type_ (that produce a value) may now be called as statements.

> This since it is meaningful to call the iteration functions only for the side effect while they still produce a
> useful result. That the produces value is unused has no consequences.

### BlockExpression v.s. ASTArray

The representation of "statements" is changed from being an `ASTArray` to being a `BlockExpression`. The difference is
that the `ASTArray` produces an array with the concatenation of each evaluated expression, whereas the `BlockExpression`
produces a single value; the last evaluated expression.

### Changes to produced / evaluation result

Some AST objects do not produce a meaningful result, they should either produce nil, or something meaningful.

> This is w.i.p as there are no current tests. As an example, a Resource creation should produce a resource reference, or
> an array of resource references, and an if statement should produce the last evaluated result of the taken branch.


### Grammar Switch

This feature allows passing settings to the parser; use the latest, turn on some experimental feature etc.

> T.B.D(escribed) - this is not yet implemented. 

Testing and Evaluation
======================
It is intended that this proposal is evaluated by performing UX studies:
* Is the Ruby/java-8 style preferred over the Unix Pipes style?
* Within the Ruby/Java-8 style what are the preferences for:
  * Parameters outside (`|$x| { ... }`)
  * Parameters inside (`{|$x| ... }`
  * Use of fat arrow (`|$x| => { ... }`), or not
* The names of the iteration functions
  * Both `each` and `foreach` are available in the implementation (`each` is recommended).
  * The names of additional functions where choosen based on the corresponding functions in Ruby (and to some
    extent Java). Are these names good?

The available implementation of the Ruby/Java-8 style allows all of these at the same time to allow for experimentation.

Currently, to evaluate, the implementation is found at: https://github.com/hlindberg/puppet/tree/pops
In addition, the RGen gem must be installed (the current release has an issue that is triggered by puppets unit tests), and
a patch has been submitted. The patched working version is found here: https://github.com/hlindberg/rgen

Alternatives and Recommendation
===============================

Many different options and alternatives were considered. These are broken into three groups; _rejected as not meeting goals_ (these are
found in a [separate document](alternatives.md), _viable alternatives and options_ to the ruby/java-8 like recommended
implementation (described below), and the alternative separately documented ["Unix Pipe" proposal](pipe_alternative.md).

The recommended implementation is described above in [Description](#description), and a detailed decision tree is found in
[Evaluation](evaluation.md), and the rationale in the document [Recommendation](recommendation.md).  

The rest of this section contains viable alternatives and options not included in the recommendation.

Alternative: 'Parameters inside' lambda syntax
--------------------------------------------

A "ruby like" option is to place the lambda parameters inside the block expression.

    {|<parameters>| <statements or expression> }

Here is an example:

    {|$x,$y| ... } # lambda with two parameters    
    {|| ... }      # lambda with no parameters

> This option was not selected because it seems to confuse non developers.
> The idea is that the recommended "parameters before" style is easier to grasp.
>
> The placement of parameters before the block also makes it possible to have other types
> of elements on the right side; such as a function reference (the notion of being a lambda
> is more associated with the `| |´ than the block expression, and creates a sense of "piping"
> that is thought to appeal to system administrators.

> This option is available in the exploratory implementation.

Alternative: Using Fat Arrow in Lambda
--------------------------------------

It may be more readable to use a fat arrow between the lambda parameters and the BlockExpression.

    $a.each |$x| => { ... }
    
> This option is available in the exploratory implementation.

Option: Bare Expression as body instead of BlockExpression
----------------------------------------------------------

The recommended solution uses a BlockExpression as the body of a lambda. It could also have been
a single expression as in this example:

   $array.select |$x| $x == 2

But this increases the amount of ambiguities that has to be addressed by the user, and practically makes
chaining be more verbose with required parentheses. This would also expose users to the effect of
precedence rules in more than just a few corner cases.

Consider:

    $a.collect |$x| $x + 2 . select ...

What is the select applied to? (A: the literal 2).

Option: Function references
---------------------------

It is useful to be able to directly pass a named function where a lambda
is allowed. The `&` operator can be used for this:

    &<NAME>

These two are then equivalent:

    $array.collect &upcase
    $array.collect |$x| { upcase($x) }

The proposal is to only allow a literal reference to a function at the
same locations as a lambda is allowed (i.e. to not support something
like `&$x` for a dynamic reference, and not accepting `&<NAME>` as an
r-value) - see "Discussion about Function Reference" below.

The addition of a function reference is optional, it is not required to
support "iteration".

Discussion about Lambda Syntax
------------------------------

When the Ruby like syntax was shown to non-programmers, there seems to
be an unnecessary cognitive complication due to the placement of the
parameters inside the braces - as this is different from say a regular
function declaration which people seems to understand.

Moving the lambda parameters outside of the body is somewhat
grammatically difficult (as we will see in a bit). In Java 8, lambdas
are written like this:

    (x, y) -> { }

Using more puppet like syntax, this could be:

    ($x, $y) => { }    
    () => { }

This however means that calls where there are no parameters must use
parentheses, as a call is otherwise ambiguous. (This would be  a drastic
change to the puppet language).

A suggested solution to this is to use a syntactic marker such as `$`:

    $(x,y)=>{}    
    ${}

But `${}` clashes with old style "non double quoted interpolated string"
that was available in some versions of puppet. `$(x,y)` is preferred to
`$($x,$y)` simply because it is lighter, but then this is not
consistent with how parameters to classes and defines are expressed.
Also, taken in isolation a `${}` syntax may be confused with
interpolation in general.

Using some other free operator works:

    &(x,y) => {}    
    &{}

This style is used in the "pipe" proposal. because that proposal also
uses `&` as a function reference, and a lambda is an anonymous function
(although defined instead of referenced).

If variables should be preceded with `$` to be consistent with other
function like declarations it starts to look heavy:

    &($x,$y) => {}    
    &{}

Some languages (e.g. go) uses a keyword `func` to introduce an anonymous
function. We could do the same (but this adds a keyword), or we could
use the convention that the name `_` means anonymous. If used as an
operator, and dropping the fat arrow, a lambda would look like this:

    _($x,$y) {}
    _{}

That looks quite magical compared to the keyword variant:

    func($x,$y) {}    
    func($x,$y) {}
    func {}

It is possible to use the pipe operators (at least in the "Ruby like"
proposal):

    |$x, $y| => { }
    || => { }

and also works without the arrow:

    |$x, $y| { }    
    || { }

Further, it is of value to be able to write a lambda that is not a block
(such as a function reference). In this case the fat arrow separator may
look more appealing. Compare:

    |$x| => &upcase($x)
    |$x| &upcase($x)
    
    $array.each |$x| {file { "/tmp/$x": }}
    $array.each |$x| => {file { "/tmp/$x": }}

Naturally, both styles (without arrow for a braced block, and with arrow for a function reference) could be
used - maybe to set them apart, and to provider better error feedback (but this may be of debatable value).

Alternative: Automatic currying
-------------------------------

You may have reacted to examples like this:

    $array.collect |$x| => &upcase($x)

as it has redundant information, this could just as well have been
written

    $array.collect &upcase

Automatic currying with a lambda could look like this but needs the
magic variable 'it':

    $array.collect &{ upcase($_) }

Since the introduction of lambda and iteration will be new to the puppet
community, it is perhaps best to always be explicit as things are
already complex and magic enough without also adding automatic currying.

Alternative: UTF-8 lambda kwyword
---------------------------------

It has been suggested in several conversations about operators and
lambdas that it would be of value to use operators in the wider UTF-8
character set. An obvious choice for lambda is then the greek lower case
lambda letter λ

    λ(x,y) {}    
    λ{}

Irrespective of the merits and problems of using the lambda character,
using UTF-8 characters has several inherent problems; they are much more
difficult to type, and there may be encoding issues in users tool-chains
that makes them drop or misinterpret such characters.

There is already criticism against having too much special punctuation, and
the addition of Greek letters the syntax starts to look like "math".

Discussion about Function Reference
-----------------------------------

As shown in the summary of the proposals for lambda, a function
reference could be created by using `&<NAME>. As there is no closure
requirement, a function reference can be allowed as an r-value, and thus
any expression can be used after the &  as long as it evaluates to the
name of a function. This means that the grammar could be:

    & <expression>

And that expression must evaluate to a string that is the name of the
function - ie. `&` is a function pointer.

This allows composition and functional oriented programming styles to be
used. This may however be far too complex for the target audience. There
are other obvious issues regarding static validation of such statements
as the correctness is not known until evaluation time.

The functional reference can allow specification of additional
arguments, i.e. an uncompleted function call, and the `it` variable
represents where the curried (value is inserted). A concrete binding for
`it` is needed; it could be spelled out as `$it`, or use a more special
`$_` syntax.

This example shows a call where the curried variable is in the second
position when the call is made.

    &myfunc(1, $_, "text")

Additional rules could be:

* if no arguments are specified, the call is made with the single 
  argument `$_`

* if arguments are specified, the `$_` is the first argument (unless
  it is included in the list)

or

* if arguments are specified, the `$_` must be included in order to
  pass it (this is less magical but also adds noise).

The addition of uncompleted calls is optional.

Option: Closures
----------------

It is proposed that lambdas are (at least initially) restricted to be
used in association with calls to functions/the runtime and not be
allowed as r-values as this requires a rewrite of scope, with potential
breakage as a consequence.

Since there are difficulties to support full closures (i.e. that lambda
is a r-value) they may only appear as an extra element in function
calls. It is the responsibility of the implementor of a function to
ensure that the scope given in the call is not referenced after the
function returns.


Risks and Assumptions
=====================

Expression Based Grammar Risks
------------------------------
### Major implementation

The expression based grammar is a major rewrite of the grammar. There is a risk that some corner cases will be handled
differently as the implementation does not have as a goal to be fully bug compatible.

This is mitigated by:

* The design where the parser produces a semantic model that is transformed into an AST model for the execution. This ensures
  that problems are most likely related to parsing (a possible type of problem is precedence issues).
* A fairly large test suite (currently 642 examples) covering syntax and transformation in a non trivial way is in place.
* Unit tests performing catalog compilation based on iteration functions are in place (currently 27 examples).
* Problems are early in the chain (lexing/parsing) and are easy to find/fix as there is a foundation for
  conveniently testing the various steps. As an example, a problem reported for 3.x that is also present in the implementation
  of this ARM could have been a lexing problem, a problem in the parser, an evaluation problem, a problem in the resource type,
  or the catalog. (The issue is http://projects.puppetlabs.com/issues/19632). Within about 15 minutes of work (using 3 copy/pasted
  and modified unit tests) it was possible to pinpoint the culprit.
* The implementation of this ARM was done with a design principle to change as little existing code as possible, and instead
  re-route and transform the call flow. This makes A/B comparisons much easier.
* If extended tests /UX studies show there are (non anticipated) extensive problems, it is possible to implement this proposal
  on top of the existing grammar (such an implementation has already been made), albeit with some undesirable syntax restrictions. 

## New Technology

The available implementation makes use of the Ruby implementation of Ecore models - RGen. This is new technology in Puppet, and
as such poses risks.

This is mitigated by:

* Ecore is technology that is well specified and have been around for a very long time
* It is an industry standard
* Ecore has a large eco system
* The implementation of RGen has been in use for many years.
* It is possible to write an inhouse version of the parts of RGen that are used by the implementation of this ARM
  in case of a disaster scenario.

Assumptions
-----------

It is assumed that:

* doing more in Puppet DSL is favored over writing code in Ruby.
* the implementation of Puppet is going in a modular / service oriented and more technology agnostic direction (although the impact
  of this assumption on this proposal is quite low).
  
Dependencies
============

This ARM does not depend on any other ARM being implemented.

The implementation has a dependency on [RGen](https://github.com/mthiede/rgen).
RGen in turn has an optional dependency on [Nokogiri](http://nokogiri.org/) if XML features are used (which they are not
in the proposed implementation).

Impact
======

* Other Puppet components:  
  * There is impact on Puppet RDoc (which seems to be somewhat broken even without changes in this ARM).
  * Tools such as Geppetto and Puppet Lint needs to be updated to handle the new syntax.
  * Standard Library - While this proposal was being prepared, an unfortunate addition of a "reduce" function was made in the
    standard puppet library.
* Compatibility: The proposal is future compatible.
* Security: Neutral.
* Performance/scalability:  
  While the initial implementation is marginally slower (it creates a semantic model that is translated
  into an AST and delegates to the "old" ways of doing things in a less than optimal way), there is potential for better
  performance in future versions.
* User experience: THe intention is to make a UX study presenting the various options.
* I18n/L10n: Neutral
* Accessibility: Neutral
* Portability: This increases portability as user code can be moved to Puppet DSL from Ruby
* Packaging/installation: The implementation is based on the RGen Gem. It needs to be included in the packaging.
* Documentation: Documentation naturally needs to be updated - since most of the implementation is about additions, and this ARM
  contains both detailed explanations and examples, the task of revising documentation should not be too hard.
* Spin-offs/Future work:  
  There are many spin-offs and interesting future work that this ARM enables. The more immediate benefits are listed as subgoals/internal
  goals (various enablers), but there are also long term benefits from using an expression based grammar.
  * The use of modeling technology makes it easier to implement services written in other languages
  * A better "Puppet RDoc" should be more easily achieved


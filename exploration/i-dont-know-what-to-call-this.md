 I don't know what to call this -- function registry guidance?

Status: **Proposed**

<details>
	<summary>Metadata</summary>
	<dl>
		<dt>Contributors</dt>
		<dd>@catamorphism</dd>
		<dt>First proposed</dt>
		<dd>2024-03-xx</dd>
		<dt>Pull Requests</dt>
		<dd>#000</dd>
	</dl>
</details>

## Objective

_What is this proposal trying to achieve?_

* Define the requirements for a "resolved value"

The spec currently leaves the term "resolved value"
undefined.
At the same time, the spec implicitly constrains
the form of a resolved value
through the operations on resolved values that it defines.
Implementing the spec requires inferring those constraints.
The primary objective is to make those constraints explicit
so that this inference only has to be done once.
This objective implies several subsidiary goals.

* Define the meaning of `.local` (and `.input`).

> In a declaration, the resolved value of the expression is bound to a variable, which is available for use by later expressions
(["Expression and Markup Resolution"](https://github.com/unicode-org/message-format-wg/blob/main/spec/formatting.md#expression-and-markup-resolution))

The term "resolved value" is left undefined, but its meaning affects the observable behavior of the formatter.

* Define the types that custom functions manipulate.

> Call the function implementation with the following arguments... If the expression includes an operand, its resolved value.
> ....If the call succeeds, resolve the value of the expression as the result of that function call.
(["Function Resolution"](https://github.com/unicode-org/message-format-wg/blob/main/spec/formatting.md#function-resolution) )

The same term, "resolved value", is used to describe the values that functions consume and produce.
An implementation that supports custom functions needs to define concrete types
(whose details depend on the underlying programming language of the implementation)
that capture all the details of "resolved values".
Implementers would benefit from guidance
on how to map the MessageFormat concept of a "resolved value"
onto a concrete type.

* Define the semantics of composability

A function can consume a value produced by another function,
since the language provides `.local` declarations.
The meaning of this composition depends on the definition
of "resolved value".
Even with a precise definition of "resolved value",
an additional question is raised of what the contract
with custom functions needs to be
in order to conform with the semantics of MessageFormat.
Disambiguating the composability question
affects observable behavior.

* Provide useful names for different kinds of functions

Currently, a given MessageFormat function can be thought of a
formatter, a selector, or both.
Because the implementations for formatters and selectors
naturally have different type signatures
(a formatter consumes and produces a resolved value,
while a selector produces a list of keys),
a single MessageFormat function that supports both formatting and selection
is expected to symbolize multiple implementations.
It may be useful to think of some functions as "transformers"
that consume and produce resolved values,
and others as "formatters" that consume a resolved value
and produce a formatted value.
Defining these terms is another objective.

## Background

The meaning of "resolved value" affects the observable behavior of formatting,
and the desired semantics of MessageFormat,
particularly with respect to function composition,
in turn impose requirements
on the expressivity of a "resolved value".

Comments about resolved values, from [intro](https://github.com/unicode-org/message-format-wg/blob/main/spec/formatting.md):

> The form of the resolved value is implementation defined and the value might not be evaluated or formatted yet. However, it needs to be "formattable", i.e. it contains everything required by the eventual formatting.

What "everything required" means depends on the semantics of formatting.

And from the ["Expression and Markup Resolution"](https://github.com/unicode-org/message-format-wg/blob/main/spec/formatting.md#expression-and-markup-resolution) section:
> Since a variable can be referenced in different ways later, implementations SHOULD NOT immediately fully format the value for output.

Is there a distinction between a "resolved value"
and a "fully formatted" resolved value?
This text suggests that there is.
Again, the distinction affects observable behavior.

### Motivating example

TODO: incorporate comments/examples from https://github.com/unicode-org/message-format-wg/pull/686

_What context is helpful to understand this proposal?_

Example: same or different?

```
.local $n = {|1.00123|}
.local $x = {$n: number maxFrac=2}
.local $y = {$n: number maxFrac=3}
{{$x} {$y}}
```

Are `$x` and `$y` the same or different?
1. Different because the result of formatting the message to a string
is "1.00 1.001", not "1.00 1.00" or "1.001 1.001".
2. Different because named values include the options
in any expressions that were evaluated in order to create the value.
3. The same when passed into functions,
because the underlying value, `$n`, is the same and the previous options are
not passed into functions; but different when formatted in a pattern.

TODO: This might need more context to make sense

### Custom and built-in functions

An implementation that supports custom functions
would be expected to define an interface between the message formatter
and function implementations.
An implementation is free to choose whether to use the same interface
for calling built-in functions,
or to build the implementations of those functions
directly into the message formatter.

For that reason, this document refers to custom functions
when discussing the function interface.
However, in some implementations, the same questions arise
for built-in functions.

### Semantics vs. implementation types

An implementation is free to use any types
and set of operations for coercing between types
as long as it preserves the observable behavior
of a message.

To choose such a set of types, it's necessary
to understand the constraints on a "resolved value".

### Semantics: named values

In the spec, the "value bound to a variable"
when processing a `.local` (or `.input`) declaration
is a "resolved value",
an undefined term.

A "resolved value", in the spec,
is also the operand of a function.

We want to understand whether the same
structure is actually used for both.

_The following is drawn from comments by Mark Davis._

Example A1:
```
    .local $n = {|1|}
    .local $x = {$n :number maxFrac=2}
```

The formatter processes the right-hand side
of the `.local` declaration of `$x`
and binds it to a value (a "resolved value",
per the spec.)

The use of "resolved value" implies that
the following two concepts:
* "the value of `$x` in the formatter's local environment"
and
* "the meaning of `$x` as an operand to a function"
denote the same entity.

So what is that entity? There are at least two interpretations:
Interpretation 1: The meaning of `$x` is
a value that effectively represents the string `"1.00"`.

Consider another example:

Example A2:
```
    .local $n1 = {|1|}
    .local $n2 = {|1.00123|}
    .local $x1 = {$n1 :number maxFrac=2}
    .local $x2 = {$n2 :number maxFrac=2}
```

Under interpretation 1, `$x1` and `$x2` are
interchangeable in any further piece of the message
that follows this fragment.
No processing can distinguish the resolved values
of the two variables.

Interpretation 2: The meaning of `$x` is
a value that represents
a formatted string `"1.00"`,
bundled with information about the source value (`"1"`),
formatter name, and formatter options.
If the resolved value of `$x`, V, is passed to another function,
that function can distinguish V from another value V1
that represents the same formatted string,
with different options.

The choice of interpretation affects the meaning of
function composition,
since it affects which values a function implementation
can distinguish from each other.

The two interpretations affect behavior
only if `$x` is passed to another function;
if it's only used unannotated in a pattern,
then the two interpretations imply the same result.

In other words, in example A2,
functions can distinguish the resolved value of `$x1`
from the resolved value of `$x2`.

An implication of interpretation 2 is that the meaning
of `$x2` (in example A2) depends on context.
When it appears in a pattern, with no annotation,
it is interchangeable with `$x1`.
When it appears in a function annotation,
it is distinguishable from `$x1`.
If every expression has a single "resolved value"
that is determined in a context-free way,
that rules out interpretation 2.

> TODO: I think this is important, maybe put it earlier

Another way to express interpretation 2 is to
express the semantics of an unannotated variable
in a pattern in this way:

`{{$x}}` is implicitly
`{{$x :format}}`

where `:format` extracts the string (or "formatted parts")
representation of `$x`'s resolved value
and discards everything else, which is not needed for
producing the final formatting result.
If this implicit processing
(an implicit type coercion?)
is introduced into the spec,
the meaning of variable names
is once again context-independent.

Also consider:

Example A3:

```
    .local $n1 = {|1.00123|}
    .local $n2 = {|1.00|}
    .local $x0 = {$n2 :number maxFrac=2}
    .local $x = {$n1 :number maxFrac=2}
    .local $y = {$x :number maxFrac=5}
    {{$x} {$y}}
```

Interpretation 1: The result of formatting this message to a string
is `"1.00 1.00000"`.
`$x` is bound to a resolved value denoting a formatted string `1.00`,
with no metadata.
The second call to `:number`, with `$x`'s resolved value as its operand,
cannot distinguish the resolved value of `$x`
from the resolved value of `$x0`.
Therefore, its result is the same as if it was passed
the resolved value of `$x0`.

Interpretation 2: The result of formatting this message to a string
is `"1.00 1.00123"`.
Under this interpretation, the second call to `:number`
can distinguish the resolved value
of `$x` from the resolved value of `$x0`.
The original string that `$x` was constructed from
is part of the resolved value,
and can be accessed to reformat that string
with more digits of precision.

When it comes to example A3,
either interpretation might be surprising to some users.

Under interpretation 2, some users might be surprised that `$y`
has more precision than `$x`.
If their mental model is that `$y` can only depend
on the result of formatting the right-hand side
of the declaration of `$x`,
then the intuition is that the extra digits are "lost"
upon assigning a value to `$x`.

To understand why interpretation 1 might be surprising
to other users, consider an analogy.

#### Spreadsheet analogy

_This idea is from Mark Davis._

Interpretation 2 treats variables analogously to cells in a spreadsheet.
In a cell of a spreadsheet, referring to a value by name
creates a reference,
rather than copying its value.
Unlike cells in spreadsheets, variables in MessageFormat are immutable.
However, like cells in spreadsheets, the definitions of variables
can refer to other variables that are in scope.

Consider a tabular rendering of example A3 (with a pseudo-spreadsheet syntax)
(this would be the "formula" view):

|    | A       |     B   | C                |  D              |   E              |
|----|---------|---------|------------------|-----------------|------------------|
| 1  | 1.00123 | 1.00    | (=B1, maxFrac=2) | =(A1, maxFrac=2 | =(D1, maxFrac=5) |

with the following mappings between cells and variables:

| A1 | n1 |
|----|----|
| B1 | n2 |
|----|----|
| C1 | x0 |
|----|----|
| D1 | x  |
|----|----|
| E1 | y  |

And the "output" view

|    | A       |     B   | C                |  D              |   E              |
|----|---------|---------|------------------|-----------------|------------------|
1    | 1.00123 | 1.00    | 1.00             | 1.00            | 1.00123          |

In E1, the reference to D1 is a reference to the _value_ of A1,
with added formatting options that can be extended or overridden.

This is consistent with interpretation 2, in which `$x` is bound to a structure
that contains the value of `$n1`.
(This could be implemented by copying, since MessageFormat variables are immutable.)

### Semantics: return values for functions

Turning to the question of what a function returns,
this at first glance is the same question as
"what is a named value",
as both are "resolved values" according to the spec.
But both interpretation 1 and interpretation 2 complicate that.

Alternative 1: A function returns a "formatted value".
This matches interpretation 1, where formatted values
are bound to names.

Alternative 2: A function returns a composite value
that conceptually pairs a base value (possibly the
operand of the function, but possibly not; see Example B1)
with options.
This matches interpretation 2.
If we preserve the single usage of "resolved value"
in the spec, this implies that the (base value, options)
representation applies to all resolved values,
not just those returned by functions.

For more details, see the "model implementations" section.

## Use-Cases

_Includes examples contributed by Markus Scherer and Elango Cheran_

### Composition

The presence of local variable declarations
allows function composition.

The following is _not_ a syntactically valid
MessageFormat 2 message:

```
{{{|1| :number} :number}}
```

No nesting of function annotations is allowed.

However, by substitution,
one can write an observationally equivalent message:

```
.local $x = {|1| :number}
{{$x :number}}
```

This implies that the composition of two functions
(conceptually, `number . number` in this case)
has to have a meaning,
since it can be expressed syntactically
(albeit indirectly with every intermediate result named).

That doesn't mean that every possible combination
of functions has to make sense if composed.
Functions are allowed to signal errors
and return values that indicate that an error occurred.

Where composition is meaningful,
its meaning depends on the choice of interpretations
as outlined in the "Semantics: named values" section,
because how functions compose depends on what they return
and what they can accept as arguments.
In turn, what they can accept as arguments
depends on what a variable name denotes.

### Options

In the previous examples, options are only overridden in
subsequent calls.
A question also arises of how to combine different options.

Example A4:

```
    .local $x = {|1| :number minInt=3}
    .local $y = {$x :number maxFrac=5}
    {{$x} {$y}}
```

If Interpretation 2 is adopted, then
this example is equivalent to:

Example A5:

```
    .local $x = {|1| :number maxFrac=5 minInt=3}
    {{$x} {$x}}
```

In compositions of calls to the same function,
it might make intuitive sense to union together
the option sets, letting the outermost enclosing
call take precedence if the same option is specified
multiple times.

It's less obvious what to do with compositions of
calls to different functions, which may have different
option sets.

How options "override" each other is also not straightforward.
Perhaps some do, others don't, and the details are
specific to each function.

Example A6:
Should $y be formatted as ٩٨٧٦٥ or 98765?

```
    input $num = 98765
    .local $x = {$num :number numberSystem=arab}
    .local $y = {$x :number numberSystem=latn}
```

Example A7:
Should $y be formatted as +98765 or 98765?
```


    input $num = 98765
    .local $x = {$num :number signDisplay=always}
    .local $y = {$x :number signDisplay=auto}
```

While some use cases don't work well
(or at least work surprisingly)
if options are not encoded in named values,
defining what it means to compose options
is not straightforward.


### Computation vs. applying formatting

Example B1:
```
    .local $age = {$person :getAge}
    .local $y = {$age :duration skeleton=yM}
    .local $z = {$y :uppercase}
```

Although there is also a pipeline of functions
(conceptually, `uppercase . duration . getAge`),
the formatted value returned by `:getAge` is _not_
just "the argument with formatting options applied",
but rather, a component of the argument.
Other functions can be imagined that do more general
computation on arguments.

It would not be correct to say that
`:uppercase` converts a person to uppercase,
nor would it be correct to say that `:uppercase`
converts a number to uppercase.

This example only makes sense if `:uppercase`
operates on the "formatted result" of `$y`.

This suggests a representation for named values
that allows functions to choose whether to
inspect the "formatted result", the "input value" and options,
or both.

This is compatible with interpretation 2.

This suggests that a type system for functions
(at least, an optional type system where
typechecking would be a lint pass)
might be useful:

```
getAge : Person -> Number
duration : Number -> Options -> String
uppercase : String -> String
```

or even

```
duration : Number -> Options{"skeleton"=String ...} -> String
```

However, this would be future work.
The [function registry data model](https://github.com/unicode-org/message-format-wg/blob/main/spec/registry.md)
attempts to do some of this, but does not define
the structure of the values produced by functions.


### More composition

TODO

Suggestion from Elango: allow _both_ kinds of composition? "operates-on-output" vs. "pipelines-input"

```
    .local $x = {$num :number maxFrac=2}
    .override $y = {$x :number maxFrac=5 padStart=3}
    {{$x} {$y}}
```

If `$num` is `0.33333`,
then the result of formatting would be

```
0.33 000.33333
```

(note: the keyword "override" should probably be avoided due to its unrelated OO connotations.)

accumulating options vs. "consuming"; overriding (last takes precedence) vs. ignoring-subsequent (first takes precedence)

TODO: I'm not sure what this means (from Markus)

```
This would still not work if the second number function were actually called with a reasonable output of the first.

This would require that the MessageFormatter code applied the override, creating the second formatter function object with the union of the options of the two expressions. With that, $y would get the result of applying a single number function, something like ({type=FixedDecimal 000.33333}, "000.33333").

This would also require that the first and second function are the same function (number), because mixing options of different functions makes no sense.

```

Mark's argument is that the "composition" model by default is better:

TODO: summarize

```
So in ICU terms I think the model should be:

$A3 = [1.32, NumberFormatter.with()]

$B3 = [A3.baseValue, A3.numberFormatter.minFractionDigits(1)]

$C3 = [A3.baseValue, B3.numberformatter.minFractionDigits(2)]

And so on. Each variable that ‘descends’ from the original input just copies that input, copies the format, and then adds / replaces the settings on that formatter.

It is a much cleaner and simpler model. What would it even mean to modify the underlying value for signDisplay=always, or useGrouping=min2? We surely don’t want numberSystem=arab to change the physical digits of the baseValue.

As well as being unnecessary, modifying the base value is actually trickier to implement in ICU (probably in other libraries as well). There isn’t an apparatus in ICU to take a numberFormatter that has particular settings and get out a Number that reflects the application of (some of them!) to the base value. And it is more work than just accumulating a (logical) set of options in the formatter.
```

### Allowing different kinds of functions

some composable/some not

Mark:

```
The internal values of a variable should be, from a spec viewpoint, only matter in terms of its demonstrable behavior. It would effect 

    The formatting of the variable
    The matching of the variable
    The generation of a new variable via a function

It is the last bit that concerns us. Otherwise, it doesn't really matter whether options change the internals or simply have an effect on the formatting/matching behavior.

Let's call a function1 "stable" (we can pick another term) iff for every set of options and inputs:

.local  $a ={$x :function1 optionsMap1} // or mutatis mutandis .input {$a :function1 optionsMap1}
.local  $b ={$a :function1 optionsMap2}

The second line is equivalent to:

.local  $b ={$a :function1 mergeRetainingLatest(optionsMap1, optionsMap2}

Thus if :number is defined to be stable, then

.local  $a ={$x :number style=percent signDisplay=always}
.local  $b ={$a :number style=decimal maximumFractionDigits=0}

is guaranteed to have the same behavior as

.local  $a ={$x :number style=percent  signDisplay=always}
.local  $b ={$x :number style=decimal  maximumFractionDigits=0 signDisplay=always}

For the message {{a is {$a} and b is {$b}}} and the value of $x=1.2, both would cause the formatting to be: 
"a is +12% and b is +1"

Note that this does not govern anything about switching functions, especially when one of the functions is mutates. So we could have:

.local $a ={$x :number style=percent signDisplay=always}
.local $c ={$a :roundTo base=1} // defined to modify
.local $d={$c :number style=decimal maximumFractionDigits=0}

In this case, $d is disconnected from $a's options, so the formatted result of {{a is {$a} and d is {$d}}} would be: 
"a is +12% and d is 0"

So my proposal would be that all the initial standard functions (:number, :datetime, ... ) should be stable. 
```

("switching functions" = switching between different functions)

### Overriding options

In MessageFormat, composition lends itself to multiple interpretations.

```
    .local $x = {$num :number maxFrac=2}
    .local $y = {$x :number maxFrac=5}
    {{$x} {$y}}
```

If the external input value of `$num` is "0.33333",
what should this message format to?

1. `0.33 0.33333`
2. `0.33 0.33`

If our model of function composition is as follows:
1. Evaluate `$num` to a value and pass it to the `:number` function,
   along with named options `{"maxFrac": "2"}`
2. Bind `$x` to the result, which is an object `X`
   encapsulating the following fields:
     * The source value, `"0.33333"`
     * The fully-evaluated options, `{"maxFrac": "2"}`
     * The formatted result, a `FormattedNumber` object
       representing the string `"0.33"`
3. Evaluate `$y` to a value, which is `X`,
   and pass it to the `:number` function,
   along with named options `{"maxFrac": "5"}`
4. Bind `$y` to the result, which is an object `Y`
   encapsulating the following fields:
     * The source value, `"0.33333"` (same as `X`'s source value)
     * The fully-evaluated options, `{"maxFrac": "5"}`
       (note: the original `maxFrac` option value has been deleted)
     * The formatted result, a `FormattedNumber` object
       representing the string `"0.33333"`

then the formatted result is option (1).

If on the other hand our model is:
1. Evaluate `$num` to a value and pass it to the `:number` function,
   along with named options `{"maxFrac": "2"}`
2. Bind `$x` to the result, which is:
     * The formatted result F, a `FormattedNumber` object
       representing the string `"0.33"`
3. Evaluate `$y` to a value, which is F,
   and pass it to the `:number` function,
   along with named options `{"maxFrac": "5"}`
4. Inside the `:number` function, check the type
   of the input, notice that it is already a
   `FormattedNumber`, and return it as-is
5. Bind `$y` to the result, which is F.

then the formatted result is option (2).

In terms of implementation, the result depends on
what the nature is of the value that is bound to
a local variable in the environment used in evaluation
within the message formatter.

The value could be a simple "formatted value" as in the second model,
and analogously to MessageFormat 1.

Or it could be a more structured value that captures
both the "formatted value", and everything that was used to construct it,
as in the first model.

The third option...
TODO: summarize this comment from Markus
```
More precisely, since we supposedly operate on (value-object, string-with-meta) pairs, I would expect something like this:

    $num = 0.33333333333 turns into ({type=double 0.33333333333}, none) input to the first function
    The first function ignores the string field, formats the double, and outputs $x = ({type=FixedDecimal 0.33}, "0.33")
    The second function gets that, ignores the string field, formats the FixedDecimal, and outputs $y = ({type=FixedDecimal 0.33000}, "0.33000")

With "FixedDecimal" I mean the ICU-internal, ICU4X-public representation of a decimal number with a defined set of digits before and after the decimal point. Rounding/truncation/padding applied. Somewhat similar to Java BigDecimal.
```

Although it's a different evaluation model, the third option is observably the same as the second one.

orthogonal choices:
* "string" named values vs. "structured" name values
* "looking back" for the original value, vs. returning a different "source value"

the choice of internal value influences the result!
(Or put differently, the desired result constrains the choice of internal value.)

Currently (2024-02-27), the spec leaves the behavior implementation-dependent.

Markus's words: "the expectation that the second function operates on the output of the first".

I don't know that it's meaningful to refer to "the expectation",
but the spec should specify the behavior
so that it's clear what the expectation is: to do what the spec says.

but it's a useful framing:
* "the second function operates on the output of the first"
vs.
* "the second function operates on the input of the first, plus 'hints' supplied by the first"

TODO: another option is `0.33000`

### Extracting a field / arbitrary computation

TODO: summarize this (repeated?)

```
Another example: A function that takes a Person object and extracts/computes the person's age.

    .local $age = {$person :getAge}
    .local $y = {$age :duration skeleton=yM}
    .local $z = {$y :uppercase}

TODO: explained above

I expect

    Input: $person --> pair ({type=Person Markus Scherer 178cm 19690408}, none)
    The getAge function takes this pair (asserting that the value is of type Person), and computes the age as now-birthday
    Let's say it returns $age = ({type=duration 54y10mo}, none)
    The duration function takes this $age and formats it; outputs something like $y = ({type=duration 54y10mo}, "54 Jahre und 10 Monate")
    The uppercase function takes $y, ignores the value object, and formats=uppercases the string, possibly preserving metadata: $z = ({type=string [same as second field]}, "54 JAHRE UND 10 MONATE")

Note:

    Functions need to be able to accept a pair that has no string-with-meta -- since that's often the real input.
    Functions need to be able to return a value object without string-with-meta. (getAge)
    Functions may ignore the input value object. (uppercase)
    Functions may return a pair with a value object of a different type than the input pair's value object.
    Even if it's the same type, the value may differ. (e.g., number taking & returning a FixedDecimal)

The function registry needs to be explicit for each function about what acceptable inputs are (including types of value objects), what the outputs are (including value type), in addition to what happens in between. Otherwise, implementations will handle the details wildly differently, and interoperability suffers.
```

In short: this isn't about preserving options, but rather "what is the internal representation"; types; and which functions compose with each other

Mark's comment on this:
"Now, we could definitely add mutating options, and we could add ‘extracting’ options. But I think we need to be very clear which are which. For example, we could have functions that take one datatype and extract a part of the input, as in your example of plucking a timestamp (birth date, I presume) from an input object representing Person information.

i.e. explicitly declaring what a function does / what options mean

subsequent email from Mark:

```
What we really have also hand-waved is how to function type X can extract some useful — and expected — value from function type Y

Suppose we have a PersonName datatype input passed to a message. (Cf. https://www.unicode.org/reports/tr35/tr35-personNames.html#person-name-attributes)

.input {$name :u:personname length=long formality=informal}

Then we write:

.local $date = {$name :u:datetime skeleton=MMMyy}

What to do about the value? I think the best that we could do is to specify in the registry which function's variables a given other function can read. And where it doesn't make any sense, it is simply an error. We could conceivably do the same for another function's options, though that starts to get complicated. I don't think a string binding helps with communcation, not unless — again — the second function understands the details of the first's.

```

(n.b. he's using "variables" in the weird way here)

### Offset

Mark (not sure where this came from):

```
BTW, I did implement offset=x in my mockup, as a test. MockMessageFormat.java

It turned out to be fairly easy. The original value needs to be retained for selecting literals (0, 1, ...), while I added an offsetted value for formatting and for selecting plural or ordinal categories. For composition, one would have to make a choice of which value to pass on to a successive function, although in this instance I think such composition would be of limited use in practice (and could be dangerous). Example:

match {$count :number :offset=1}{$bookTitle}
0 {{You have no books}}
1 {{You have {$bookTitle}}}
2 {{You have {$bookTitle} and one other book}}
* {{You have  {$bookTitle} and {$count} other books}}

When we format $count in the * line, it clearly needs to use the offsetValue. So far, so good. That means if you want to do something special, you need to have keep another variable

.locale $countOther {$count : number}
match {$countOther :number :offset=1}{$bookTitle}
0 {{You have no books}}
1 {{You have {$count} book, {$bookTitle}}}
2 {{You have {$bookTitle} and one other book; for a total of {$count} books}}
* {{You have {$bookTitle} and {$countOther} other books; for a total of {$count} books}}

We really want to steer users away from the danger of changing the format in the submessage to be different than the selection.

match {$count :number minFractionDigits=0}
one {{The library has an average of {$count minFractionDigits=1} book per person}}
* {{The library has an average of {$count} books per person}}

That can result in the malformed message (for English, but similar cases happen in other languages): 

The library has an average of 1.0 book per person
```

## Requirements

_What properties does the solution have to manifest to enable the use-cases above?_

- Guarantee interpretation 2 in the spec

* composition (in a way that makes sense)

which implies
* preservation of options
* preservation of original operand value when appropriate
* passing through a new operand value _and_ options when appropriate, as with functions that abstract fields


## Constraints

_What prior decisions and existing conditions limit the possible design?_

prior decision: using the same "resolved value" definition everywhere
* However, I think it still works, just with "resolved value" = "annotated value with metadata"

difficulty of matching a more-specific "resolved value" design (in the semantics)
with the concrete function interface, in an implementation-language-neutral way

## Proposed design

_Describe the proposed solution. Consider syntax, formatting, errors, registry, tooling, interchange._

We show how we can evolve a proposed design starting from an "obvious" solution,
working towards one that meets the requirements.

Separating:
* "there should be a richer 'resolved value' type, & here's how that accompanies the function interface

from

the specifics (in #645)

from

type system / registry / how functions would actually declare the structures of their options/result types
(which isn't necessary! for now, functions can error out whenever they want)

### Implementation: type signatures: formatters, selectors

(partially based on feedback from Markus Scherer)

The following uses an abstract notation for type signatures.

In MessageFormat 1, conceptually a formatter had the type:

```
Object -> FormattedValue
```

(Java), or

```
Formattable -> FormattedValue
```

(C++)

Where a `FormattedValue` is a string with metadata.

In MessageFormat 2, the language of "formatted values" is richer.
Formatters can return arbitrary objects, so
the MessageFormat 2 equivalent of a `FormattedValue`
contains something with more structure.

In defining an interface for custom functions,
an implementer might initially choose a type signature like:

```
Formattable -> FormattedValue
```

where `Formattable` is a type that can express
the types that literals and the values of
external input variables might have.

Note that the simplest interpretation of this set of type signatures
forces interpretation 1 (from the "Semantics: named values" section).

Question 1: how does the formatter
coerces a `FormattedValue` to a resolved value?

Incorporating options, the type might look like:

```
Formattable -> Map<String, Formattable> -> FormattedValue
```

where `Options` is a map from strings to `Formattable`s.

Question 2: how does the formatter coerce
a resolved value to a `Formattable`?

Given the semantics for selection,
the type signature for selectors might look like:

```
Formattable -> Options -> [Key] -> [Key]
```

(where `[Key]` refers to a list of keys.)

(The implementation-specific `MatchSelectorKeys()` method
"...takes as arguments a resolved selector value `rv`"
-- ["Resolve Preferences"](https://github.com/unicode-org/message-format-wg/blob/main/spec/formatting.md#resolve-preferences))

The same question arises: how is a resolved value
coerced to a `Formattable`?

(A single function that supports both formatting and selection
would map to two different implementations with the same name,
given the difference between the type signatures.)

Question 3: what are options bound to in the option map?

> If the option's right-hand side successfully resolves to a value, bind the identifier of the option to the resolved value in the mapping.
([Option Resolution](https://github.com/unicode-org/message-format-wg/blob/main/spec/formatting.md#option-resolution))

This text suggests that the range of the option map,
the argument type of formatters,
and the argument type of selectors should all be the same type.

There are two possibilities:

1. The formatter coerces between resolved values and `Formattable` values
(valid inputs to functions).
It must also coerce `FormattedValue`s to resolved values.
The question is how to do that without loss of information
that is necessary for a future function call.
This doesn't match how the spec is written,
given the text "If the expression includes an operand, its resolved value."
In this option, instead of "resolved value",
it would be something like:
"If the expression includes an operand, the result of coercing its resolved value
to T", where T is the implementation's argument type for functions.
2. Rather than taking `Formattables` or returning `FormattedValue`s,
a single type, isomorphic to the formatter's internal "resolved value" type,
is both the argument type and return type of a function.

Note that these questions do not arise only
in an implementation in a typed language,
since an implementer working in an unityped language
might still choose to use a different structure
for internal "resolved values" and for
function argument and result types,
and conversion that changes the representation
of the values might be necessary.

Under interpretation 2:

> a function like :number produces a composite value specific to that function, which we can think of as a [numberDatatype, numberOptions] pair.`
(Mark)

see implementation notes below... "composite value specific to that function" _or_ a _generalized_ composite value


> TODO: I think Possibility 1 doesn't work, because the formatter doesn't "know" what the base value should be.
> Consider example B1.
> In addition, some options might be "dropped" and the details are specific to each function.


### Model function registry interface

One design,
[code](https://github.com/macchiati/cldr/blob/MF-Function-Mock/tools/cldr-code/src/test/java/org/unicode/cldr/tool/MockMessageFormat.java)
contributed by Mark Davis/@macchiati.


Mock interface expressed in Java:

```
  /**
     * A factory that represents a particular function. It is used to create a FunctionVariable from
     * either an input, or another variable (
     */
    public interface FunctionFactory {
        public FunctionVariable fromVariable(
                String variableName, MFContext context, OptionsMap options);

        public FunctionVariable fromInput(Object input, OptionsMap options);

        public boolean canSelect();

        public boolean canFormat();
    }

    /** A variable with information that results from applying a function (factory). */
    abstract static class FunctionVariable {
        private OptionsMap options;
        // The subclasses will have a value as well

        public OptionsMap getOptions() {
            return options;
        }

        public void setOptions(OptionsMap options) {
            this.options = options;
        }

        public abstract boolean match(MFContext context, String matchKey);

        public abstract String format(MFContext context);

        // abstract Parts formatToParts(); // not necessary for mock
   }
```

Some issues:
* Calling the interface `FunctionVariable` is misleading since the object
may appear directly in a pattern; it may not ultimately be bound to a variable
* Having `fromVariable()` take a MessageFormat variable name directly
(and a "local environment" implicitly in the `MFContext`)
blurs the boundary between the message formatter and functions' execution context.
It also makes the function responsible for "forcing" thunks in a lazy
implementation; it's simpler if "forcing" is done before calling the function.

This does have the advantage that it directly expresses the ways
data can flow into a function:
* From the value bound to a variable (`fromVariable()`)
* From the value of an external input variable (`fromInput()`)

The third way data can flow into a function is from a literal.
The interface here omits a `fromLiteral()` method that would take a `String literal`.

The `FunctionVariable` interface encapsulates the options that were passed in.
More fields could be added (by subclassing `FunctionVariable`),
such as the input `Object` or `String`.
Rather than having a `setOptions()` method,
it might be preferable to define a `FunctionVariable` constructor
that takes an options map as an argument.
As it is, the return value of a function has mutable state
and this is not consistent with the pure functional semantics
of MessageFormat.

> [!NOTE]
>
> Or maybe it should? What if the formatter "wants" to do additional processing
> on the return value of the function?

Also, `match()` should take a list of keys (see [`MatchSelectorKeys()`](https://github.com/unicode-org/message-format-wg/blob/main/spec/formatting.md#resolve-preferences))

In C++, there is no equivalent of `Object` and so the type signature for `fromInput()`
would need to look something like:

```
FunctionVariable fromInput(const Formattable& input, OptionsMap options);
```

where the `Formattable` class is defined to encompass all possible types
that input values can have.
(See the existing [`icu::Formattable`](https://unicode-org.github.io/icu-docs/apidoc/dev/icu4c/classicu_1_1Formattable.html)
class; the draft ICU4C implementation defines its own `Formattable` class
to work around a bug in the existing ICU class.)

### Composition

The above interface handles composition by passing variable names
directly into functions. As already noted, this is not ideal.

If we want to avoid passing in a local variable name directly,
we have to define what variables are bound to in the local environment
used by the formatter.
In an interpreter, a local environment maps names to values.
The question is what the "value" is.
The answer is implementation-dependent,
but granting the ability to define custom functions means exposing --
if not the value type itself -- a type that the value type
can be coerced to, without losing any relevant information.

One solution:

```
    public interface FunctionFactory {
        public FunctionVariable fromVariable(
                FunctionVariable variable, OptionsMap options);

        public FunctionVariable fromInput(Object input, OptionsMap options);

        // other methods are unchanged
    }

```

Now, `fromVariable()` takes a `FunctionVariable` instead of a
variable name, and no longer needs an `MFContext` argument.
(It might be desirable to pass other context,
but for expository purposes we omit it.)

### Definition of `FunctionVariable`

Calling it a `FunctionVariable` is misleading at this point,
but we'll stick with the name for now.

Consider the following class that inherits from `FunctionVariable`:

```
    /** A variable with information that results from applying a function (factory). */
    class InputFunctionVariable implements FunctionVariable {

        public OptionsMap getOptions() {
            // Returns an empty options map
        }

        public void setOptions(OptionsMap options) {
            // Throws an exception
        }

        // Would perform literal matching if this is a string;
        // perhaps stringify the contents and then do literal matching
        // otherwise?
        public abstract boolean match(MFContext context, String matchKey);

        // A string formats to itself; otherwise would apply default formatters
        public abstract String format(MFContext context);

        // abstract Parts formatToParts(); // not necessary for mock

        private Object contents;
   }
```

With this definition, we have a way of representing an input
as a `FunctionVariable`, and no longer need the `fromInput()` method.
The `Object` representing the input is encapsulated in a `FunctionVariable`.

So we just need:

```
    public interface FunctionFactory {
        public FunctionVariable fromVariable(
                FunctionVariable variable, OptionsMap options);

        // fromInput() is deleted
        // other methods are unchanged
    }

```

This is just another alternative.
With the initial design, the class for a specific function
could implement a version of `fromVariable()` that throws an exception,
and only provide a non-throwing implementation for `fromInput()`,
or vice versa.
With the second design, the method would internally
use `instanceof` on its `variable` argument and throw exceptions
in cases where the type is not supported.

The second version of the `FunctionFactory` interface
can be translated directly to C++.
In the `InputFunctionVariable` class,
the `Object contents` would have to be represented as a `Formattable` instead.

### A third design for `FunctionVariable`

Alternately, we could require all classes
implementing `FunctionVariable` to have a `getValue()` method:

```
    /** A variable with information that results from applying a function (factory). */
    abstract static class FunctionVariable {
        private OptionsMap options;
        // The subclasses will have a value as well

        public Object getValue() // or call it getInput()? Can't have a default implementation

        public OptionsMap getOptions() {
            return options;
        }

        public void setOptions(OptionsMap options) {
            this.options = options;
        }

        public abstract boolean match(MFContext context, String matchKey);

        public abstract String format(MFContext context);

        // abstract Parts formatToParts(); // not necessary for mock
   }
```

In C++ it would be:
```
const Formattable& getValue();
```

This has the advantage (?) of enforcing that _every_ `FunctionVariable`
carries with it the input (or literal) value that was ultimately used to create it,
before the pipeline of transformations was applied.

### A fourth design (eager evaluation of outputs)

note that "match"/"format" are *lazy* here --
also, what happens when you call match() on something returned by a format-only
function, or format() on something returned by a selector-only function?

so in place of `match()` and `format()`, either have a `FormattedValue` or...
well, whatever we need for selection. (Probably this means we need a separate
`FormatterFactory` and `SelectorFactory`.)

### A fifth design (separate FormatterFactory and SelectorFactory)

TODO: For selection, we don't really need to save all that prior
state because now there is no "output", hence no need to relate it to input

### A sixth design (allow errors)

In Java we would expect errors to be signaled using exceptions,
but in C++ a value would have to be returned (this document is written
with libicu in mind, which does not use C++ exceptions).

```

class FunctionVariable {
   virtual bool isError();
   // other methods
}

class ErrorFunctionVariable : FunctionVariable {
   bool isError() { return true; }
   const UnicodeString& errorMessage() { /* returns a string describing the error */ }
}
```

A new virtual method is added that returns true
iff this is a special "error" value,
and a special class `ErrorFunctionVariable` can be instantiated
to create an "error" return value when needed.

### Function registry interface as done in ICU4J

Elango:

"As far as the impact to our ability to implement, Mihai says that there's not a problem, actually. Also, his ICU4J implementation, even as it stands right now, shows that there's not a problem. Instead of returning as a string, we have the common interface FormattedValue that we have all concrete types implement. Those concrete types will likely be more structured types than the original input, anyways, ex: FormattedNumber. And we need runtime type information (RTTI). Java provides that RTTI in the language itself (ex: `instanceof`), while Tim has hacked RTTI into his ICU4C implementation using the `tag()` interface method added to the C++ FormattableObject.

Mihai also feels that interpreting the right-hand side expression of a .local as truly behaving like functions -- which means interpreting them as returning values -- has more benefit than the idea of interpreting them as representing the set of options passed to a function invocation. The use case here is that we can represent composition of functions, which isn't possible in MF1, but could be useful to support operations for inflected languages, etc. The obvious use case for interpreting a .local as a set of options is concision -- saving the user from repeating substrings across different .local declarations, but this is something that copy+paste trivially solves -- it doesn't confer new functionality that the user didn't already have."


### Miscellaneous


TODO

following Mark's suggestion:

```
That is, I think a function FX should have the job of converting (literals, input, its type of variable, possibly other variables) to its type of variable VX, which has its internal type plus formatting/selecting. Then VX must support matching literals or formatting or both (depending on what kind of variable it is). It could also provide API that would allow extraction of other characteristics. And those characteristics may or may not depend on post-formatting, as below. 

.input {$height :measure}

.local $unitGender = {$height :gender}

(the formatting unit might be inches, feet & inches, centimeters, meters, etc. gender will vary by formatting unit and locale)


> because it violates the expectation that the second function operates on the output of the first.

I agree; but given the model of the first being a pair, it doesn’t mean that the formatting/selecting options have to modify the base value.

```

Building on this: not just a pair. `FormattedPlaceholder` is conceptually a tuple, though


### Type system?

Mark:
"I  am afraid if we are going "typeless", then the issue becomes much trickier on the implementation side, and less predictable for clients. It kinda forces the intermediate value to be a stringified literal as the least common denominator, which will have various implementation costs (and require some pretty careful specification).

I agree on the mutation, that it is much cleaner to separate that. 

In the meeting a few people were talking about the case of "passing" a value to a different function, whereby it is left up to the receiving function to decide which option pairs to consider and which to disregard. 

I think that could work, but would want to work out the details."

(Also relates to stuff from #645)

## Alternatives Considered

_What other solutions are available?_
_How do they compare against the requirements?_
_What other properties they have?_

Elango:

```
    Option A1: .local represents function option overrides when functions are same, but standard functions need to stabilized if you want function composition guarantees when functions differ
    Option A2: .local represents function option overrides when functions are same, but disallow composition of .local when functions differ
    Option B: .local represents function invocation results (which allows composition) regardless of functions being the same/different, and create syntax .override to represent function option overrides when functions are the same
    Option C: Altogether disallow the argument to a .local being a previous .local declaration name. This allows us to defer a decision on what .local actually means until later, and it simplifies our short-term implementation work as a result. The tradeoff is that, until such time, function options must be fully expressed (no concision of overrides), and no support for function composition.

Option A1 is my understanding of Mark's position. Option B is what Markus and I talked about. I'm adding A2 & C to provide other alternatives / more completeness.
```

(rather than "stabilized", "composable")
TODO: add pros/cons of both
In OptionC, it really has to say "a previous .local or .input declaration name" -- can only name an unannotated input variable + an annotation
and then, that effectively means getting rid of `.local` completely, because `.input` already exists to annotate
an unannotated input variable
well, I guess you can still use `.local` for literals and annotated literals
what about named values in option values? like

```
.local $w = {|2| :number}
.local $x = {|1| :func x=$w}
```

Mark:
"
In any event, I am coming to think that in the Tech Preview, we need to simply disallow applying one function to another function's variables. We can loosen that restriction once we figure out how it should work."
(using "variables" to mean "results", again)

Rich:
"One thing that might be helpful is to draw a clear distinction between functions that format (take some input and produce an annotated human-readable string) or select (choose something from a list of options based on some input value) and functions that mutate the value in some way.  I think having one “function” type that does both of these things isn’t going to end well."

(transform -- return a different thing _as a function of_ the input -- not mutate)

also Rich:
"We see that in some of the discussion of numeric values.  Is setting maxFractionDigits simply specifying the output format, or is it mutating the number?  And how does this apply to, say, setting the sign style or grouping interval, which really CAN’T mutate the underlying number?  I’d argue we shouldn’t be trying to format and calculate with the same “function.”

I’m thinking maybe for now we focus on the formatting and selecting and talk about performing calculations later."

## Acknowledgments

This document incorporates comments and suggestions from Elango Cheran, Mark Davis, and Markus Scherer.

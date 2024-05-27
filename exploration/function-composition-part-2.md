# Function Composition - Part 2

Status: **Proposed**

<details>
	<summary>Metadata</summary>
	<dl>
		<dt>Contributors</dt>
		<dd>@catamorphism (also see Acknowledgments)</dd>
		<dt>First proposed</dt>
		<dd>2024-03-xx</dd>
		<dt>Pull Requests</dt>
		<dd>#000</dd>
	</dl>
</details>

## Objective

_What is this proposal trying to achieve?_

In part 1... (link)

* A concrete definition of "resolved value"

## Constraints

_What prior decisions and existing conditions limit the possible design?_

One prior decision is that the same definition of
"resolved value" appears in multiple places in the spec.
If "resolved value" is defined broadly enough
(an annotated value with rich metadata),
then this prior decision need not be changed.

A second constraint is
the difficulty of developing a precise definition of "resolved value"
that can be made specific in the interface for custom functions,
which is implementation-language-neutral.

A third constraint is the "typeless" nature of the existing MessageFormat spec.
The idea of specifying which functions are able to compose with each other
resembles the idea of specifying a type system for functions.
Specifying rules for function composition, while also remaining typeless,
seems difficult and potentially unpredictable.

## Alternatives to consider

_Describe the proposed solution. Consider syntax, formatting, errors, registry, tooling, interchange._

> This design document does not present a proposed design. What's provided here
> is alternatives considered so far.
> A concrete design will be presented in a future design document,
> once agreement is reached on what the problem space is.

In lieu of the usual "Proposed design" and "Alternatives considered" sections,
we offer some alternatives already considered in separate discussions,
which are not meant to constitute a complete list.

We show how we can evolve a proposed design starting from an "obvious" solution,
working towards one that meets the requirements.

Separating:
* "there should be a richer 'resolved value' type, & here's how that accompanies the function interface

from

the specifics (in #645)

from

type system / registry / how functions would actually declare the structures of their options/result types
(which isn't necessary! for now, functions can error out whenever they want)

### Typed functions

The following option aims to provide a general mechanism
for custom function authors
to specify how functions compose with each other.

"Resolved value" could be defined as the most general type
in a system of user-defined types.
Each custom function could declare its own argument type
and result type.

This does not imply the existence of any static typechecking.
A function passed the wrong type could signal a runtime error.
This does require some mechanism for dynamically inspecting
the type of a value.

Consider Example B1 from part 1 of the document:

Example B1:
```
    .local $age = {$person :getAge}
    .local $y = {$age :duration skeleton=yM}
    .local $z = {$y :uppercase}
```

Informally, we can write the type signatures for
the three custom functions in this example:

```
getAge : Person -> Number
duration : Number -> Options -> String
uppercase : String -> String
```

or even

```
duration : Number -> Options{"skeleton"=String ...} -> String
```

`Number` and `String` are assumed to be primitive types.
Functions with options can also mention an `Options` type,
which is a map from strings to resolved values,
and perhaps even specify which options must be present.

The [function registry data model](https://github.com/unicode-org/message-format-wg/blob/main/spec/registry.md)
attempts to do some of this, but does not define
the structure of the values produced by functions.

An optional static typechecking pass (linting)
would then detect any cases where functions are composed in a way that
doesn't make sense. For example:

Semantically invalid example:
```
.local $z = {$person: uppercase}
```

A person can't be converted to uppercase; or, `:uppercase` expects
a `String`, not a `Person`. So an optional tool could flag this
as an error, assuming that enough type information
was included in the registry.

### Composition operates on output

A less general solution is to have a single "resolved value"
type, and specify that if function `g` consumes the resolved value
produced by function `f`,
then `g` operates on the output of `f`.

```
    .local $x = {$num :number maxFrac=2}
    .local $y = {$x :number maxFrac=5 padStart=3}
```

In this example, `$x` would be bound to the formatted result
of calling `:number` on `$num`. So the `maxFrac` option would
be "lost" and when determining the value of `$y`, the second
set of options would be used.

For built-ins, it suffices to define resolved value as something like:

```
FormattedNumber | FormattedDateTime | String
```

because no information about the input needs to be
incorporated into the resolved value.

(and all custom functions would return strings?)

### Composition operates on input plus metadata

"Pipelining" input

Another definition of composition:

 If function `g` consumes the resolved value
produced by function `f`,
then `g` operates on the **input** of `f`,
possibly with metadata (resolved options) attached.

_Are the options part of the resolved value, or a separate thing?_

```
Resolved value = InputType x OutputType x Options
OutputType = FormattedNumber | FormattedDateTime | String
```

(This is "alternative 2" under "Semantics: return values for functions
in the first doc.)

### Allow both kinds of composition (with different syntax)

TODO: Make sure this doesn't repeat anything from "Overriding options"

We've identified two different meanings that composition could have:

1. Operating on output: If function `g` consumes the resolved value
produced by function `f`,
then `g` operates on the output of `f`.
2. Pipelining input: If function `g` consumes the resolved value
produced by function `f`,
then `g` operates on the **input** of `f`,
possibly with metadata attached.

Even if we choose option 2, we still have to answer
the question of how different option sets (that "metadata")
get combined:

* Are options "accumulated" through the pipeline?
* Or does each function choose whether to keep or discard each option?

Elango suggested allowing _both_ meanings, with different syntax:

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

TODO: another option is `0.33000`

because the `maxFrac` provided in the definition of `$y`
overrides the `maxFrac` provided in the definition of `$x`.

(note: the keyword "override" should probably be avoided due to its unrelated OO connotations.)

The question remains: what is the input to `:number`?

a. Does the MessageFormatter code do the work of combining the options returned
by the first call to `:number`
with the options provided in the second call?
b. Or does the MessageFormatter code pass an object that encodes
both sets of options, and the implementation of `:number` takes care
of combining eerything together in the correct way?

And these questions only make sense because the function is the same
for both `$x` and `$y`;
for pairs of different functions, different rules are needed
(possibly it's always an error in that case.)

What is passed to `:number`?

* "underlying value plus options"
or
* "single set of options", determined by the MessageFormatter implementation

### Don't allow composition

```
number : Number -> FormattedNumber
```

then it would be a runtime error to pass a `FormattedNumber` into `number`

_Possibly get rid of everything after this point?_

### "Stable" functions

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


### Where does this go? TODO

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

TODO: incorporate comments/examples from https://github.com/unicode-org/message-format-wg/pull/686

#### offset (not sure where this goes)


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

#### other

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


#### other (?)

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
```


#### other - relating to the spreadsheet example

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


## Acknowledgments

This document incorporates comments and suggestions from Elango Cheran, Mark Davis, and Markus Scherer.

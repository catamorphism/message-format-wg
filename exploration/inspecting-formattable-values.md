# Formatted Parts

Status: **Proposed**

<details>
	<summary>Metadata</summary>
	<dl>
		<dt>Contributors</dt>
		<dd>@catamorphism</dd>
		<dt>First proposed</dt>
		<dd>2024-02-05</dd>
		<dt>Pull Request</dt>
		<dd></dd>
	</dl>
</details>

## Objective

Custom formatting functions should be able to
inspect the raw value and formatting options
of their arguments.
In addition, while a custom formatter may eagerly
format its operand to a string,
returning the raw underlying value
and the formatting options used for formatting
are also useful,
in case another function wants to extend these options
or use them for other logic.

Making the underlying structure of its inputs,
as well as requiring formatters to return structured outputs,
is helpful for building interoperable custom formatters.

## Background

In the accepted version of the spec (as of this writing),
the term "resolved value" is used for several different kinds
of intermediate values,
and the structure of resolved values is left completely
implementation-specific.

Providing a mechanism for custom formatters to inspect more
detailed information about their arguments requires the
different kinds of intermediate values to be differentiated
from each other and more precisely specified.

At the same time, the implementation can still be given freedom
to define the underlying types for representing formattable values
and formatted results. This proposal just defines wrappers
for those types that implementations must use in order to
make custom functions as flexible as possible.

## Use-Cases

Use cases from [issue 515](https://github.com/unicode-org/message-format-wg/issues/515):

The following code fragment
invokes the `:number` formatter on the literal `1`,
binds the result to `$a`, and then invokes `:number` on the
value bound to `$a`.

If the value of `$a` does not allow for inspecting the previous options
passed to the first call to `:number`,
then the `$b` would format as `1.000`.

### Example 1
```
.local $a = {1 :number minIntegerDigits=3} // formats as 001.
.local $b = {$a :number minFractionDigits=3} // formats as 001.000
// min integer digits are preserved from the previous call.
```

In other words: the user likely expects this code to be equivalent to:

### Example 2
```
.local $b = {1 :number minIntegerDigits=3 minFractionDigits=3}
```

But without `:number` being able to access the previously passed options,
the two fragments won't be equivalent.
This requires `:number` to return a value that encodes
the options that were passed in, the value that was passed in,
and the formatted result;
not just the formatted result.

Another example:
### Example 3
```
.input {$item :noun case=accusative count=1}
.local $colorMatchingGrammaticalNumberGenderCase = {$color :adjective accord=$item}
```

The `:adjective` function is a hypothetical custom formatter.
If the value of its `accord` option is a string, it's hard for `:adjective`
to use the value of `accord` to inflect the value of `$color` appropriately
given the value of `$item`.
We want to pass not the formatted result of `{$item :noun case=accusative count=1}`
into `:adjective`, but rather, a structure that encodes that formatted result,
along with the resolved value of `$item` and the names and values of the options
previously passed to `:noun`: `case` and `count`.

## Requirements

- Define the structure passed in as an argument to a custom formatting function.
- Define the structure that a custom formatting function should return.
- Maintain the options passed into the callee as a _separate_ argument to the
  formatter, to avoid confusion. (See Example 4 below.)
- The structure returned as a value must encode the formatted result,
input value, and options that were passed in.
- Articulate the difference between a "formattable value"
(which is the domain of the input mapping (argument mapping),
and the result of evaluating a _literal_)
and a "formatted value"
(which is what a formatting function (usually) returns).
- Clarify the handling of formattable vs. formatted values:
does a formatting function take either, or both?
  - This proposal proposes that formatter _inputs_ are a superset of
  formatter _outputs_ (in other words, the output of a formatter can
  be passed back in to another formatter).

Any solution should support the examples shown in the "Examples" section.

## Constraints

### Implementation-defined behavior

- The spec says:
* "The form of the resolved value is implementation defined"

If the value returned by `:number` is specified,
but the value bound to `$a` (in the above example) is
implementation-defined,
then it's hard to tell if an implementation is compliant.

### Requirements for being formattable

On the other hand...
* "it needs to be "formattable", i.e. it contains everything required by the eventual formatting."

So what does "everything required" mean? (The point of this proposal is to define that.)

### Evaluation order

- The spec does not require eagerness or laziness

Which is not a problem, since what we're defining is the _value_ type
and not the _thunk_ (suspension) type. This is orthogonal to the
question of eager vs. lazy evaluation.

### The function registry

Function registry contains specifications of the correct "input",
but that's distinct from what we're saying is the "input".
Need the right terminology:
* `:number` takes a string or number (for example),
but that's always wrapped in a (?) `FormattedPlaceholder` thing
that also has fields representing the options from the previous
formatter.

- See the "Function Resolution" section of the spec

Step 4: "Call the function implementation with the following arguments..."
* "If the expression includes an operand, its resolved value.

If the form of the resolved value is implementation-defined,
it's hard to say what the form of the input to the formatting function is,
and likewise its result.

https://github.com/unicode-org/message-format-wg/blob/main/spec/formatting.md#function-resolution

## Proposed Design

### Taxonomy

This proposal introduces several new concepts
and eliminates the term "resolved value" from the spec.

* _Nameable value_: A value that a variable can be bound to
in the runtime environment that defines local variables.
This concept is introduced to make it easier for the spec
to formally account for both eager and lazy evaluation.
In an eager implementation, "nameable value" is synonymous
with "annotated formattable value",
while in a lazy evaluation, a "nameable value" would be a
closure (code paired with in-scope set of variables).

* _Formattable value_: An implementation-specific type
that is the range of the input mapping (message arguments),
as well as what literals format to.

* _Formatted value_: The result of a formatting function.
This type is implementation-specific, but expected to
at least include strings.
A sequence of one or more _formatted values_
is also the ultimate result of formatting a message,
since formatters either format to a single string,
or a sequence of "parts"; the implementation-specific "parts"
are expected to have the implementation-specific
_formatted value_ type.

* _Fallback value_: A value representing a formatting error.

* _Annotated formattable value_: Encapsulates any
value that can be passed as an argument to a formatting function,
including:
  * formattable values that don't yet have formatted output
    associated with them
  * previously formatted values, which can be reformatted by
    another formatting function
  * fallback values
The values of named options passed to formatting functions
are also _annotated formattable values_.

* _Markup value_: The result of formatting a markup item
(disjoint from _annotated formattable values_).

* _Preformatted value_: A formattable value paired with a
formatted value (and some extra information).

In the current spec, a "resolved value" can be a
nameable value, formattable value, preformatted value, or
option value, depending on context.

This proposal keeps the "formattable value" and "formatted value"
types implementation-specific while defining wrapper types
around them, which is intended to strike a balance between
freedom of choice for implementors
and specifiability.

### Interfaces

```
type FormatterInput =
  | FallbackValue
  | FormattableValue
  | PreformattedValue;

interface AnnotatedFormattableValue = {
  source: string;
  value: FormatterInput;
}

interface FallbackValue = {
  fallback: string;
}

type FormattableValue = /* Implementation-dependent */;

type FormattedValue = /* Implementation-dependent */;

interface FormattableOptionValue = {
  value: FormattableValue;
}

interface FormattedOptionValue = {
   value: FormattedValue;
}

interface PreformattedValue = {
  options: Iterable<{ name: string; value: AnnotatedFormattableValue}>;
  formatter: string;
  input: AnnotatedFormattableValue;
  output: FormattedValue;
}
```

Implementations are free to extend the `AnnotatedFormattableValue`
and `PreformattedValue` interfaces with additional fields.

### Function Signatures

In C++, for example, the interface for custom formatters could look like:

```
virtual AnnotatedFormattableValue customFormatter(AnnotatedFormattableValue&& argument,
                                                  std::map<std::string, AnnotatedFormattableValue>&& options) const = 0;
```

Note that this proposal is orthogonal to the existing
[data model and validation rules for the function registry](https://github.com/unicode-org/message-format-wg/blob/main/spec/registry.md).
Input validation can still be applied to the underlying _formattable values_ and _formatted values_
wrapped in the `argument` and the values for the `options`.

## Examples

### Example 4

Same code as Example 1:
```
.local $a = {1 :number minIntegerDigits=3} // formats as 001.
.local $b = {$a :number minFractionDigits=3} // formats as 001.000
```

Assuming eager evaluation,
the right-hand side of `$a` is evaluated to the following _annotated formattable value_:

```
{ source-text: '{1 :number minIntegerDigits=3}',
  preformatted-value: {
    options: { 'minIntegerDigits': { source-text: '3', formattable-value: '3'}},
    formatter: Number,
    formatter-input: { source-text: '1', formattable-value: '1' },
    formatter-output: FormattedNumber('001.') }}
```

(In a lazy implementation, `$a` would be bound to a _nameable value_
that is forced to an _annotated formattable value_ when it is used
in a formatter or selector call
or inside a pattern that has been selected for formatting.
The _nameable values_ still need to include everything required
to produce this eventual result.)

Then, the right-hand side of `$b` is evaluated
in an environment that binds the name `a`
to the above _annotated formattable value_.
The result is:
```
{ source-text: '{$a :number minFractionDigits=3}',
  preformatted-value: {
    options: { 'minFractionDigits': { source-text: '3', formattable-value: '3'}},
    formatter: Number,
    formatter-input: /* The _annotated formattable value_ shown above */
    formatter-output: FormattedNumber('001.000') }}
```

When calling the implementation of the built-in `number`
formatting function (call it `Number::format()`),
while evaluating the right-hand side of `$b`,
`number` receives two sets of options, in different ways:
* The option `minFractionDigits=3` is passed to `Number::format()`
  in its `options` argument.
* The option `minIntegerDigits=3` is embedded in the `argument` value.

In general, the previous formatter might be different from
`Number::format()`, so all the "previous" options have to be
separated from the options in the option map.
The solution in this proposal encodes them in a tree-like structure
representing the chain of previous formatting calls
(really a list-like structure, since formatters have a single argument).

### Example 5

This example motivates why option values need to be
_annotated formattable values_.

Same code as Example 3:
```
.input {$item :noun case=accusative count=1}
.local $colorMatchingGrammaticalNumberGenderCase = {$color :adjective accord=$item}
```

When processing the `.input` declaration,
supposing that `item` is bound in the input mapping
to the string 'balloon',
and `color` is bound in the input mapping
to the string 'red',
the name `item` is bound in the runtime environment
to a _nameable value_ equivalent to
the following _annotated formattable value_:

```
{ source-text: '{$item :noun case=accusative count=1}',
  preformatted-value: {
    options: { 'case': {source-text: 'accusative', formattable-value: 'accusative'},
               'count': {source-text: '1', formattable-value: '1' }},
    formatter: Noun,
    formatter-input: { source-text: '$item', { formattable-value: 'balloon' }},
    formatter-output: 'balloon' }}
```

(As this example uses English, the `case` option has no effect
on the formatted output.)

Then, when processing the right-hand side of the `local` declaration,
the argument to the `adjective` formatter is as follows:

```
{ source-text: '$color',
  formattable-value: 'red' }
```

and the option mapping maps the name 'accord' to the same
_annotated formattable value_ that was returned by the `noun` formatter,
shown above.

The result of the call to the `adjective` formatter looks like:

```
{ source-text: '{$color :adjective accord=$item}',
  preformatted-value: {
    options: { 'accord': /* Same _annotated formattable value_ returned by the `noun` formatter */ },
    formatter: Adjective,
    formatter-input: { source-text: '$color', { formattable-value: 'red' }},
    formatter-output: 'red' }}
```

(Again, since the example uses English, the `accord` option has
no effect on the output.)

If the output of the `adjective` formatter was formatted by a subsequent
formatter, it would be able to inspect the value of the output's
`accord` option, along with all of **its** options.

## Alternatives Considered

### Not defining the shape of inputs or outputs to custom formatters

Leave it to implementations,
as is currently done in the spec with "resolved values".

### Functions return minimal results; formatter fills in extra results

In this alternative: the function argument would still be
an _annotated formattable value_,
but the function can just return a... _formatted value_...?
since otherwise, it's just copying identical fields into the
result (the source text doesn't change; the formatter name
and resolved options are already known by the caller code
in the MesageFormat implementation; etc.)

* But, what happens if a function "wants" to just
return the _formattable value_ that is passed in;
if the result type of the function is the same as _formatted value_,
then this can't be expressed. This suggests:

### Functions return the union of _formatted value_ and a _formattable value_

This is hard to express in some programming languages' type systems.

### Encode metadata in formatting context

The argument to the function would be the union of
a _formattable value_ and a _formatted value_ (to allow reformatting)
and the function would have to access the formatting context to
query its previously-passed options, etc.

This does make the common case simple (most custom functions will
probably not need to inspect values in this way),
but allowing the argument to be a union type
also makes it hard to express the function signature
in some programming languages.

### Functions don't preserve options and can't inspect previous options

May violate intuition (as with the number example)
or make grammatical transformations much harder to implement
(as in the accord example)

### Other representations

* Alternative: No _annotated formattable values_; just
a _formattable value_ with some required fields, and
specify that the implementation may add further fields.

* Alternative: Still have an _annotated formattable value_
that wraps a _formattable value_,
but instead of having either a `formattable-value` or
a `preformatted-value` field,
have a list of fields such that some are optional
for a `formattable-value`.

* Alternative: flat structure instead of tree structure. Consider:

Consider:

```
{ source-text: /* ... */,
  preformatted-value: {
    options: {'a', 1},
    formatter: F,
    formatter-input: { source-text: /* ... */,
                       preformatted-value: {
                         options: {'b', 2}
                         formatter: F,
                         formatter-input: { source-text: /* ... */,
                                            preformatted-value: {
                                              options: {'c', 3},
                                              formatter: F,
                                              formatter-input: { source-text: /* ... */, formattable-value: 'foo'  },
                                              formatter-output: /* ... */ } },
                         formatter-output: /* ... */ }},
     formatter-output: /* ... */ }}
```

for some formatter F. Recursing through this tree structure to find all the previous options
might be tedious.
For nested values where the formatter is the same for all the nested values,
a single `options` list might be more convenient.
However, it's unclear how to use a flat representation if the nested values
are produced by different formatters that take different options.

## Incidental notes

The spec currently says:

> Function access to the _formatting context_ MUST be minimal and read-only,
> and execution time SHOULD be limited.

With the right representation for _annotated formattable values_,
will functions need to access the _formatting context_ at all?

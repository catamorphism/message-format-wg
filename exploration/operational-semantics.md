# An Operational Semantics for MessageFormat

Status: **Proposed**

<details>
	<summary>Metadata</summary>
	<dl>
		<dt>Contributors</dt>
		<dd>@catamorphism</dd>
		<dt>First proposed</dt>
		<dd>2024-02-12</dd>
		<dt>Pull Request</dt>
		<dd></dd>
	</dl>
</details>

## Objective

The following is a big-step operational semantics
for MessageFormat that takes inspiration from
the style of semantics presented by Benjamin Pierce in
_Types and Programming Languages_ (ch. 11).

This provides a declarative specification for formatting,
to be used alongside the existing non-executable
imperative specification and the various executable specifications
(implementations of MessageFormat).
It is non-normative.
The goal is to validate the "inspecting formattable values"/
"dataflow" design document.

This is a call-by-value semantics, describing _eager_ evaluation.
A call-by-name semantics could also be written,
which would model lazy evaluation
The two semantics would be observationally equivalent,
because MessageFormat has no side effects.

### Limitations

The model elides error handling
(all error conditions result in "stuck" states).

Markup is not covered.

## Background

> TODO: see other PR

In [other PR], I propose defining the various kinds of runtime values
in order to better specify the flow of data through the formatter,
particularly regarding the flow of data into and out of
built-in and custom formatting functions.

This semantics shows how those runtime values
may be used in a declarative model of MessageFormat.
While this is intended as non-normative,
it may be helpful to guide implementation
and/or as a tool for understanding the normative spec.

## Term Language

The following can be thought of as an alternative notation
for [the data model spec](https://github.com/unicode-org/message-format-wg/blob/main/spec/data-model/message.json),
describing the same semantic domain with a different surface representation.

The "syntax" used in the definitions is minimal and not intended
to map directly on to [the ABNF grammar](https://github.com/unicode-org/message-format-wg/blob/main/spec/message.abnf).

### Messages

The metavariable `m` stands for messages.

A message is either a pattern message or a selectors message.
Both kinds of messages have a sequence of declarations.
A selectors message also has a non-empty sequence of expressions
and a non-empty sequence of variants. The two sequences
may have different lengths.

<pre>
m ::=
   d p
|  d &lt;e<sub>i</sub>, <sup>i &isin; 1..n</sup>&gt; &lt;var<sub>i</sub>, <sup>i &isin; 1..m</sup>&gt;
</pre>

### Declarations

The metavariable `d` stands for sequences of declarations.

<pre>
d ::=   &empty;
     |  d, local v = e
     |  d, input v = e
</pre>

### Variants

The metavariable `var` stands for variants.

Each key list must have the same length
as the sequence of expressions
in the enclosing message,
but this constraint is not syntactically enforced.

<pre>
var ::= &lt;k<sub>i</sub>, <sup>i &isin; 1..n</sup>&gt; p.
</pre>

### Keys

The metavariable `k` stands for keys.

<pre>
k ::= *
   |  t
</pre>

### Text

The metavariable `t` stands for arbitrary text.

### Patterns

The metavariable `p` stands for patterns.

<pre>
p ::= &empty;
    | p, t
    | p, e
</pre>

### Variable names

The metavariable `v` stands for a variable name.

### Expressions

The metavariable `e` stands for expressions.

<pre>
e ::= v
   |  l
   |  ae
</pre>

### Annotated expressions

The metavariable `ae` stands for annotated expressions,
i.e. function calls.

<pre>
ae ::= v a
     | l a
     | a
</pre>

### Annotations

The metavariable `a` stands for annotations.
An annotation represents a function name together with
a sequence of named arguments.
The name of a function is arbitrary text.

<pre>
a ::= t
    | t o
</pre>

### Options

The metavariable `o` stands for sequences of options.

<pre>
o ::= &empty;
    | o, t = l
    | o, t = v
</pre>

### Literals

The metavariable `l` stands for literals.

<pre>
l ::= t
</pre>

(For the purposes of this presentation,
a literal is just arbitrary text.)

## Value Language

A call-by-name semantics would introduce another
"nameable value" value type. In this semantics,
"nameable value" is just a synonym for
"annotated formattable value" and thus the term
"nameable value" is not used.

### Annotated Formattable Values

The metavariable `afv` stands for annotated formattable values.

<pre>
afv ::= fv
     |  pv
</pre>

Note: to model error handling, we could change
this to:

<pre>
afv ::= fv
      | pv
      | ev
</pre>

where `ev` would be a distinguished set of error values.
For expository purposes, we use the first definition.


Optional fields could be added
and modeled in this semantics, but for simplicity,
they are elided.

For example, a `source-text` field could be
added that would be passed through for
error handling use. In the semantics, this would look like:

<pre>
afv ::= t fv
      | t pv
</pre>

or if we wanted to model that an annotated formattable value
contains an arbitrary sequence of other key-value pairs:

<pre>
afv ::= md fv
      | md pv

md ::= &empty;
     | md, { t = t }
</pre>

### Formattable Values

The metavariable `fv` stands for formattable values.
Formattable values are a superset of strings
(regardless of implementation-specific behavior,
strings must be in the set of formattable values).

<pre>
fv ::= t
    |  uninterpretedFormattableValue
</pre>

### Formatted Values

The metavariable `dv` stands for formatted values.
Similarly to formattable values,
formated values are a superset of strings
(regardless of implementation-specific behavior,
strings must be in the set of formatted values).

<pre>
dv ::= t
    |  uninterpretedFormattedValue
</pre>

### Preformatted Values

The metavariable `pv` stands for preformatted values.

A preformatted value has an option sequence,
a formatter name (arbitrary text), an input
(an annotated formattable value), and an output
(a formatted value).
<pre>
pv ::= opts, t, afv, dv
</pre>

> TODO: This doesn't quite spell out how the
> returned _preformatted value_ has to relate
> to the input and options -- but maybe that's
> exactly as it should be

### Option Sequences

The metavariable `opts` stands for a sequence
of runtime options, which may be empty.

<pre>
opts ::= &empty;
    | opts, t = afv
</pre>

### Callable Values

The metavariable `cv` stands for a function name
along with its evaluated options.

<pre>
cv ::= t opts
</pre>

## Runtime Mappings

### Local Environments

Since we are describing eager evaluation,
we could dispense with an environment
and use substitution instead.
For clarity, we introduce an environment anyway.

An environment is a mapping from variable names
to annotated formattable values:

<pre>
E ::= &empty;
    | E[v &rarr; afv]
</pre>

The shorthand `E(v)` stands for the result of
applying the following procedure:

<pre>
E(v) = &perp; if E = &empty;
E(v) = afv if E = E<sub>1</sub>[v &rarr; afv]
E(v) = E(E<sub>1</sub>) if E<sub>1</sub>[v<sub>1</sub> &rarr; afv]
</pre>

where &perp; is a special value that is not equal to any other value
in the value language.

### Global Environment

A second mapping, from variable names to
_formattable values_, is assumed to exist.
This mapping models the "message arguments"
or "external variables".

<pre>
A ::= &empty;
   | A[v &rarr; fv]
</pre>

and `A(v)` is analogous to `E(v)`.

### Literal Mapping

A mapping from literals to strings
is assumed to exist.

<pre>
L : l &rarr; t
</pre>

### Function Call Mapping

The following mapping models
calling a function with an argument and options.
The behavior of the function itself is uninterpreted.
`CALL` might not be defined on every `(cv, afv)` pair.

<pre>
CALL : (cv, afv) &rarr; afv
</pre>

`CALL` also has a unary form,
to model calling a function with no argument:

<pre>
CALL : cv &rarr; afv
</pre>

### Formatting Mapping

An uninterpreted mapping from _annotated formattable values_
to _formatted values_ is presumed to exist.

<pre>
FORMAT : afv &rarr; dv
</pre>

### Selection Mapping

> TODO: could go to the spec and try to encode the selection algorithm declaratively

A mapping from a non-empty sequence of _annotated formattable values_,
and a non-empty sequence of variants,
to an integer is assumed to exist.
The integer result is interpreted as an index into the sequence of input variants
(where the starting index is 1).
The two input sequences may have different lengths.

<pre>
SELECT : (afv<sup>+</sup>, var<sup>+</sup>) &rarr; &#8469;
</pre>

## Evaluation Rules

### Literals

_Rule type: l &rarr; fv_


<pre>

--------- [E-LIT]

l &xrarr; L(l)

</pre>

A literal evaluates to its string value
according to the literal mapping `L`.
By previous definition,
a string value is a formattable value.

### Variables

_Rule type: (E, v) &rarr; afv_

<pre>
E(v) &ne; &perp;
----------- [E-VAR-LOCAL]
E, v &xrarr; E(v)
</pre>

<pre>
E(v) = &perp;
A(v) &ne; &perp;
----------- [E-VAR-GLOBAL]
E, v &xrarr; A(v)
</pre>

(No rule matches if both `E(v)` = &perp; and `A(v)` = &perp;).

The `E-VAR-GLOBAL` rule is well-formed because
`A(v)` is a _formattable value_
and a _formattable value_ is, by definition,
an _annotated formattable value_.

### Calls

_Rule type: (E, a) &rarr; cv_


<pre>
-------------------- [E-CALL-NO-OPTS]
  E, t &empty; &xrarr; t &empty;
</pre>

<pre>
  E, t o &xrarr; t opts
  l &xrarr; fv
-------------------- [E-CALL-OPT-LIT]
  E, t o, t<sub>1</sub> = l &xrarr; t opts, t<sub>1</sub> = fv
</pre>

<pre>
  E, t o &xrarr; t opts
  E, v &xrarr; afv
-------------------- [E-CALL-OPT-VAR]
  E, t o, t<sub>1</sub> = v &xrarr; t opts, t<sub>1</sub> = afv
</pre>

### Annotated expressions

_Rule type: (E, ae) &rarr; afv_

<pre>
  E, v &xrarr; afv
  E, a &xrarr; cv
-------------------- [E-ANN-VAR]
  E, v a &xrarr; CALL(cv, afv)
</pre>

<pre>
  l &xrarr; fv
  E, a &xrarr; cv
-------------------- [E-ANN-LIT]
  E, l a &xrarr; CALL(cv, fv)
</pre>

Note that this rule is well-formed
because `fv` is an annotated formattable value,
by definition.

<pre>
  E, a &xrarr; cv
-------------------- [E-ANN-STANDALONE]
  E, a &xrarr; CALL(cv)
</pre>

### Patterns

<i>Rule type: (E, p) &rarr; dv<sup>*</sup></i>

<pre>
----------- [E-PAT-EMPTY]
E, &empty; &xrarr; &empty;
</pre>

(An empty pattern evaluates to an empty sequence of formatted values.)

<pre>
E, p &xrarr; dv<sub>1</sub>, ... , dv<sub>n</sub>
-------------------------------------- [E-PAT-TEXT]
E, p, t &xrarr; dv<sub>1</sub>, ..., dv<sub>n</sub>, t
</pre>

(Append the text piece to the evaluated parts)

<pre>
E, p &xrarr; dv<sub>1</sub>, ... , dv<sub>n</sub>
E, e &xrarr; afv
------------------------------------- [E-PAT-EXPR]
E, p, e &xrarr; dv<sub>1</sub>, ..., dv<sub>n</sub>, FORMAT(afv)
</pre>

(Append the result of evaluating the expression piece to the evaluated parts)

### Declarations

<i>Rule type: (E, d) &rarr; E</i>

<pre>
----------------- [E-DECLS-EMPTY]
E, &empty; &xrarr; E
</pre>

<pre>
E, d &xrarr; E<sub>1</sub>
E<sub>1</sub>, e &xrarr; afv
------------------ [E-DECLS-LOCAL]
E, d, local v = e; &xrarr; E<sub>1</sub>[v &rarr; afv]
</pre>

Note: a separate semantic check enforces
the uniqueness of local variable names,
so this rule needs no special consideration
for name capture.

<pre>
E, d &xrarr; E<sub>1</sub>
A(v) = fv
E<sub>1</sub>, e &xrarr; afv
------------------------------- [E-DECLS-INPUT]
E, d, input v = e; &xrarr; E<sub>1</sub>[v &rarr; afv]
</pre>

### Messages

<i>Rule type: m &rarr; dv<sup>*</sup></i>

<pre>
&empty;, d &xrarr; E
E, p &xrarr; dv<sub>1</sub>, ... , dv<sub>n</sub>
-------------------  [E-MESSAGE-PAT]
d p &xrarr; dv<sub>1</sub>, ... , dv<sub>n</sub>
</pre>

To format a pattern message, evaluate its declarations to get an environment,
the format the pattern in that environment.

<pre>
&empty;, d &xrarr; E
E, e<sub>1</sub> &xrarr; afv<sub>1</sub>, ... , E, e<sub>n</sub> &xrarr; afv<sub>n</sub>
SELECT(&lt;afv<sub>i</sub>, <sup>i &isin; 1..n</sup>&gt; , &lt;var<sub>i</sub>, <sup>i &isin; 1..m</sup>&gt;) &xrarr; k
1 &le; k &le; m
E, p<sub>k</sub> &xrarr; dv<sub>1</sub>, ... , dv<sub>x</sub>
----------------------------------------------  [E-MESSAGE-SELECT]
d &lt;e<sub>i</sub>, <sup>i &isin; 1..n</sup>&gt; &lt;var<sub>i</sub>, <sup>i &isin; 1..m</sup>&gt; &xrarr; dv<sub>1</sub>, ... , dv<sub>x</sub>
</pre>

To format a `select` message:<ul>
  <li>Evaluate its declarations to get an environment.
  <li>Evaluate the sequence of selectors (expressions) to get a sequence of annotated formattable values.
  <li>Invoke the `SELECT` operation on this sequence of values, and the variants, to get the index _k_ to the chosen variant.
    (If _k_ is out of range, then this rule does not match.)
  <li>Evaluate the pattern in the environment, yielding a sequence of formatted values.
</ul>

## Example derivations

> TODO

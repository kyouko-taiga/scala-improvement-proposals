---
layout: sip
number: NN
permalink: /sips/:number.html
redirect_from:
  - /sips/:title.html
  - /sips/:number
stage: design
status: under review
presip-thread: https://contributors.scala-lang.org/t/6488
title: Match Expressions with Sub Cases
---

**By: Eugène Flesselle and Dimi Racordon**

## History

| Date          | Version            |
|---------------|--------------------|
| Dec 27th 2025 | Initial Draft      |

## Summary

We propose to support match expressions with sub cases.
If none of the sub cases matches, then control flow proceeds with the next outer case.
For example:

```scala
enum Version:
  case Legacy
  case Stable(major: Int, minor: Int)

case class Document(title: String, version: String)

def semanticVersion(s: String): Version = ???

def checkedVersion(d: Option[Document]) = d match
  case Some(x) if semanticVersion(x.version) match
    case Version.Stable(m, n) if m > 2 => s"$m.$n"
    case Version.Legacy => s"legacy ${x.title}"
  case _ => "unsupported"
```

## Motivation

Despite Scala's powerful pattern matching features, it remains difficult to write a single case that (1) extracts some values, (2) applies some function on these values, and (3) matches the result of this function against multiple but non-exhaustive patterns.
The simplest way to perform these operations is to duplicate either the default case or the outer pattern.
For example, given the definitions introduced in the summary:

```scala
def checkedVersion(d: Option[Document]) = d match
  case Some(x) => semanticVersion(x.version) match
    case Version.Stable(m, n) if m > 2 => s"$m.$n"
    case Version.Legacy => s"legacy ${x.title}"
    case _ => "unsupported"
  case _ => "unsupported"
```

The first case of the "outer" match extracts a document from an instance of `Option`.
Then, a "nested" match expression looks at the version of this document.
If the nested match expression is not exhaustive, its introduction causes code duplication, as illustrated above, where the `"unsupported"` string must be repeated in two case bodies.

Alternatively, one can always define a custom `unapply` method, as shown below:

```scala
object Document:
  def unapply(d: Document): Some[(String, Version)] =
    Some((d.title, semanticVersion(d.version)))

def checkedVersion(d: Option[Document]) = d match
  case Some(Document(_, Version.Stable(m, n))) if m > 2 => s"$m.$n"
  case Some(Document(t, Version.Legacy)) => s"legacy $t"
  case _ => "unsupported"
```

This approach requires duplicating the outer pattern, namely `Some(_)`.
This pattern could be much larger in more sophisticated examples and/or undesirable to evaluate more than once.

The goal of this proposal is to streamline pattern matching with multiple non-exhaustive sub cases.

## Proposed solution

### High-level overview

A case in a match expression can be followed by another sub-match expression, introduced by the `if` keyword.
For example, as shown in the summary:

```scala
def checkedVersion(d: Option[Document]) = d match
  case Some(x) if semanticVersion(x) match
    case Version.Stable(m, n) if m > 2 => s"$m.$n"
    case Version.Legacy => s"legacy ${x.title}"
  case _ => "unsupported"

assert(checkedVersion(Some(Document("...", "3.8"))) == "3.8")
assert(checkedVersion(Some(Document("...", "2.11"))) == "unsupported")
assert(checkedVersion(Some(Document("...", "beta:1.14"))) == "legacy ...")
assert(checkedVersion(None) == "unsupported")
```

The cases of a sub-match expression are tested iff the outer case matches.
The sub match scrutinee can refer to variables bound from the outer pattern.
Evaluation of sub-matches then proceeds as usual.
For example, if `checkedVersion` is applied on `Some(Document("...", "3.8"))`, the first outer pattern matches, causing the sub match expression to be evaluated with scrutinee `Version.Stable(3, 8)`, yielding `"3.8"`.

If none of the sub-cases match then control flow returns to the outer match expression and proceeds as though the current case (i.e., `Some(x)`) had not matched.
For example, `Some(Document("...", "2.11"))` matches the first pattern, but none of its sub-cases, and we therefore obtain the result `"unsupported"`.

### Specification

Sub-cases allow:
- Arbitrary nesting, e.g. sub-sub-matches are supported.
- Interleaving boolean guards, e.g. `case Some(x: Int) if x != 0 if x match ...`.
- Interleaving pattern extractors and computations for the scrutinees of sub-matches.

Sub-cases are supported for:
- match clauses
- catch clauses
- partial functions

Similarly to catch clauses, match expressions with a single case can now be written on single line (without braces),
e.g., `Some(1) match case Some(x) => x`.

Exhaustivity and reachability checking conservatively assume the sub-cases to be partial, similarly boolean guards.

A sub-match is inlined iff the outer match is inlined, with the same semantics as the usual match expressions.






A specification for the proposed changes, as precise as possible. This section should address difficult interactions with other language features, possible error conditions, and corner cases as much as the good behavior.

For example, if the syntax of the language is changed, this section should list the differences in the grammar of the language. If it affects the type system, the section should explain how the feature interacts with it.

### Compatibility

A justification of why the proposal will preserve backward binary and TASTy compatibility. Changes are backward binary compatible if the bytecode produced by a newer compiler can link against library bytecode produced by an older compiler. Changes are backward TASTy compatible if the TASTy files produced by older compilers can be read, with equivalent semantics, by the newer compilers.

If it doesn't do so "by construction", this section should present the ideas of how this could be fixed (through deserialization-time patches and/or alternative binary encodings). It is OK to say here that you don't know how binary and TASTy compatibility will be affected at the time of submitting the proposal. However, by the time it is accepted, those issues will need to be resolved.

This section should also argue to what extent backward source compatibility is preserved. In particular, it should show that it doesn't alter the semantics of existing valid programs.

### Feature Interactions

A discussion of how the proposal interacts with other language features. Think about the following questions:

- When envisioning the application of your proposal, what features come to mind as most likely to interact with it?
- Can you imagine scenarios where such interactions might go wrong?
- How would you solve such negative scenarios? Any limitations/checks/restrictions on syntax/semantics to prevent them from happening? Include such solutions in your proposal.

### Other concerns

If you think of anything else that is worth discussing about the proposal, this is where it should go. Examples include interoperability concerns, cross-platform concerns, implementation challenges.

### Open questions

If some design aspects are not settled yet, this section can present the open questions, with possible alternatives. By the time the proposal is accepted, all the open questions will have to be resolved.

## Alternatives

New keyword => Bad
`with` => clash with matt
then => was not appreciated by Odersky



This section should present alternative proposals that were considered. It should evaluate the pros and cons of each alternative, and contrast them to the main proposal above.

Having alternatives is not a strict requirement for a proposal, but having at least one with carefully exposed pros and cons gives much more weight to the proposal as a whole.

## Related work

Ultimate Syntax Lionel Parreaux


This section should list prior work related to the proposal, notably:

- A link to the Pre-SIP discussion that led to this proposal,
- Any other previous proposal (accepted or rejected) covering something similar as the current proposal,
- Whether the proposal is similar to something already existing in other languages,
- If there is already a proof-of-concept implementation, a link to it will be welcome here.

## FAQ

This section will probably initially be empty. As discussions on the proposal progress, it is likely that some questions will come repeatedly. They should be listed here, with appropriate answers.

# UCAN Delegation Specification
## Version 1.0.0-rc.1

## Editors

- [Brooklyn Zelenka], [Witchcraft Software]

## Authors

- [Brooklyn Zelenka], [Witchcraft Software]
- [Daniel Holmgren], [Bluesky]
- [Irakli Gozalishvili], [Protocol Labs]
- [Philipp Krüger], [number zero]

## Dependencies

- [UCAN]

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119].

# Abstract

This specification describes the representation and semantics for delegating attenuated authority between principals. UCAN Delegation provides a cryptographically verifiable container, batched capabilities, hierarchical authority, and a minimal syntatically-driven policy langauge.

# Introduction

UCAN Delegation is a delegable certificate capability system with runtime-extensibility, ad hoc conditions, cacheability, and focused on ease of use and interoperability. Delegations act as a proofs for [UCAN Invocation]s.

Delegation provides a way to "transfer authority without transferring cryptographic keys". As an authorization system, it is more interested in "what can be done" than a list of "who can do what". For more on how Delegation fits into UCAN, please refer to the [high level spec][UCAN].

# [UCAN Envelope] Configuration

## Type Tag

The UCAN envelope tag for UCAN Delegation MUST be set to `ucan/dlg@1.0.0-rc.1`.
 
## Delegation Payload

The Delegation payload MUST describe the authorization claims, who is involved, and its validity period.

| Field   | Type                                      | Required | Description                                                 |
|---------|-------------------------------------------|----------|-------------------------------------------------------------|
| `iss`   | `DID`                                     | Yes      | Issuer DID (sender)                                         |
| `aud`   | `DID`                                     | Yes      | Audience DID (receiver)                                     |
| `sub`   | `DID`                                     | Yes      | Principal that the chain is about (the [Subject])           |
| `cmd`   | `String`                                  | Yes      | The [Command] to eventually invoke                          |
| `pol`   | `Policy`                                  | Yes      | [Policy]                                                    |
| `nonce` | `Bytes`                                   | Yes      | Nonce                                                       |
| `meta`  | `{String : Any}`                          | No       | [Meta] (asserted, signed data) — is not delegated authority |
| `nbf`   | `Integer` (53-bits[^js-num-size])         | No       | "Not before" UTC Unix Timestamp in seconds (valid from)     |
| `exp`   | `Integer \| null` (53-bits[^js-num-size]) | Yes      | Expiration UTC Unix Timestamp in seconds (valid until)      |

# Capability

Capabilities are the semantically-relevant claims of a delegation. They MUST be presented as a map under the `cap` field as a map. This map is REQUIRED but MAY be empty. This MUST take the following form:

| Field  | Type      | Required | Description                                                                                      |
|--------|-----------|----------|--------------------------------------------------------------------------------------------------|
| `sub`  | `DID`     | Yes      | The [Subject] that this Capability is about                                                      |
| `cmd`  | `Command` | Yes      | The [Command] of this Capability                                                                 |
| `pol`  | `Policy`  | Yes      | Additional constraints on eventual Invocation arguments, expressed in the [UCAN Policy Language] |

Here is an illustrative example:

``` js
{
  // ...
  "sub": "did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp"
  "cmd": "/blog/post/create",
  "pol": [
    ["==", ".status", "draft"],
    ["every", ".reviewer", ["match", ".email", "*@example.com"]],
    ["some", ".tags", 
      ["or",
        ["==", ".", "news"], 
        ["==", ".", "press"]]]
  ]
}
```

## Subject

The Subject MUST be the DID that initiated the delegation chain.

For example:

``` js
{
  "sub": "did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp",
  // ...
}
```

## Resource

Unlike Subjects and Commands, Resources are semantic rather than syntactic. The Resource is the "what" that a capability describes.

By default, the Resource of a capability is the Subject. This makes the delegation chain self-certifying.

``` js
{
  "sub": "did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp", // Subject
  // ...
}
```

In the case where access to an [external resource] is delegated, the Subject MUST own the relationship to the Resource. The Resource SHOULD be referenced by a `uri` key in the relevant [Conditions], except where it would be clearer to do otherwise. This MUST be defined by the Subject and understood by the executor.

``` js
{
  "sub": "did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp",
  "cmd": "/crud/create",
  "pol": [
    ["==", ".url", "https://example.com/blog/"], // Resource managed by the Subject
    // ...
  ],
  // ...
}
```

# Policy Language

UCAN Delegation uses predicate logic statements extended with [jq]-style selectors as a policy language. Policies are syntactically driven, and MUST constrain the `args` field of an eventual [Invocation].

A Policy is always given as an array of predicates. This top-level array is implicitly treated as a logical `and`, where `args` MUST pass validation of every top-level predicate.

Policies are structured as trees. With the exception of subtrees under `some`, `or`, and `not`, every leaf MUST evaluate to `true`. `some`, `or`, and `not` must 

A Policy is an array of statements. Every statement MUST take the form `[operator, selector, argument]` except for negation which MUST take the form `["not", statement]`.

Below is a formal syntax for the UCAN Policy Language given in [ABNF] (for DAG-JSON):

``` abnf
policy      = "[" *1(statement *("," statement)) "]"
statement   = equality 
            / inequality 
            / match
            / connective 
            / quantifier

;; STRUCTURAL

connective  = "[" DQUOTE "not" DQUOTE ","  statement "]"                    ; Negation
            / "[" DQUOTE "and" DQUOTE ",[" statement *("," statement) "]]" ; Conjuction
            / "[" DQUOTE "or"  DQUOTE ",[" statement *("," statement) "]]" ; Disjunction

quanitifier = "[" DQUOTE "every" DQUOTE "," selector "," policy "]" ; Universal
            / "[" DQUOTE "some"  DQUOTE "," selector "," policy "]" ; Existential

;; COMPARISONS

equality    = "[" DQUOTE "==" DQUOTE "," selector "," ipld   "]" ; Equality on IPLD literals
inequality  = "[" DQUOTE ">"  DQUOTE "," selector "," number "]" ; Numeric greater-than
            / "[" DQUOTE ">=" DQUOTE "," selector "," number "]" ; Numeric greater-than-or-equal
            / "[" DQUOTE "<"  DQUOTE "," selector "," number "]" ; Numeric lesser-than
            / "[" DQUOTE "<=" DQUOTE "," selector "," number "]" ; Numeric lesser-than-or-equal

match       = "[" DQUOTE "match" DQUOTE "," selector "," pattern "]" ; String wildcard matching

;; SELECTORS

selector    = DQUOTE "." DQUOTE                     ; Identity
            / DQUOTE 1*(subselector *1("?")) DQUOTE ; Nested subselectors with optional "try"

subselector = "." CHAR string                           ; Dotted field selector
            / *1(".") "[\" DQUOTE string "\" DQUOTE "]" ; Explicit field selector
            / *1(".") "[" integer "]"                   ; Index selector
            / *1(".") "[]"                              ; Collection values // FIXME doble check code

;; SPECIAL LITERALS

pattern     = DQUOTE string DQUOTE ; Reminder: IPLD strings are UTF-8
number      = integer / float
```
 
## Comparisons

| Operator | Argument(s)                    | Example                          |
|----------|--------------------------------|----------------------------------|
| `==`     | `Selector, IPLD`               | `["==", ".a", [1, 2, {"b": 3}]]` |
| `<`      | `Selector, (integer \| float)` | `["<",  ".a", 1]`                |
| `<=`     | `Selector, (integer \| float)` | `["<=", ".a", 1]`                |
| `>`      | `Selector, (integer \| float)` | `[">",  ".a", 1]`                |
| `>=`     | `Selector, (integer \| float)` | `[">=", ".a", 1]`                |

Literal equality (`==`) MUST match the resolved selecor to entire IPLD argument. This is a "deep comparison".

Numeric inequalities MUST be agnostic to numeric type. In other words, the decimal representation is considered equivalent to an integer (`1 == 1.0 == 1.00`). Attempting to compare a non-numeric type MUST return false and MUST NOT throw an exception.

## Glob Matching

| Operator | Argument(s)         | Example                             |
|----------|---------------------|-------------------------------------|
| `match`  | `Selector, Pattern` | `["==", ".email", "*@example.com"]` |

Glob patterns MUST only include one specicial character: `*` ("wildcard"). There is no single character matcher. As many `*`s as desired MAY be used. Non-wildcard `*`-literals MUST be escaped (`"\*"`). Attempting to match on a non-string MUST return false and MUST NOT throw an exception.

The wildcard represents zero-or-more characters. The following string literals MUST pass validation for the pattern `"Alice\*, Bob*, Carol.`:

* `"Alice*, Bob, Carol."`
* `"Alice*, Bob, Dan, Erin, Carol."`
* `"Alice*, Bob  , Carol."`
* `"Alice*, Bob*, Carol."`

The following MUST NOT pass validation for that same pattern:

* `"Alice*, Bob, Carol"` (missing the final `.`)
* `"Alice*, Bob*, Carol!"` (final `.` MUST NOT be treated as a wildcard)
* `"Alice, Bob, Carol."` (missing the `*` after `Alice`)
* `"Alice Cooper, Bob, Carol."` (the `*` after `Alice` is an escaped literal in the pattern)
* `" Alice*, Bob, Carol. "` (whitespace in the pattern is significant)

## Connectives

Connectives add context to their enclosed statement(s).

| Operator | Argument(s)   | Example                                    |
|----------|---------------|--------------------------------------------|
| `not`    | `Statement`   | `["not", [">", ".a", 1]]`                  |
| `and`    | `[Statement]` | `["and", [[">", ".a", 1], [">", ".b", 2]]` |
| `or`     | `[Statement]` | `["or",  [[">", ".a", 1], [">", ".b", 2]]` | 

`not` MUST invert the truth value of the inner statement. For example, if `["==", ".a", 1]` were false (`.a` is not 1), then `["not", ["==", ".a", 1]]` would be true.

`and` MUST take an arbitrarily long array of statements, and require that every inner statement be true. An empty array MUST be treated as vacuously true.

`or` MUST take an arbitrarily long array of statements, and require that at least one inner statement be true. An empty array MUST be treated as vacuously true.

## Quantification

When a selector resolves to a collection (an array or map), quantifiers provide a way to extend `and` and `or` to their contents. Attempting to quantify over a non-collection MUST return false and MUST NOT throw an exception.

Quantifying over an array is straightforward: it MUST apply the inner statement to each array value. Quantifying over a map MUST extract the values (discarding the keys), and then MUST proceed onthe values the same as if it were an array.

| Operator | Argument(s)             | Example                          |
|----------|-------------------------|----------------------------------|
| `every`  | `Selector, [Statement]` | `["every", ".a" [">", ".b", 1]]` |
| `some`   | `Selector, [Statement]` | `["some",  ".a" [">", ".b", 1]]` |

`every` extends `and` over collections. `some` extends `or` over collections. For example:

``` js
const args = {"a": [{"b": 1}, {"b": 2}, {"z": [7, 8, 9]}]}
const statement = ["every", ".a", [">", ".b", 0]]

// Reduction
["every", [{"b": 1}, {"b": 2}, {"z": [7, 8, 9]}], [">", ".b", 0]]

["and", [
  [
    [">", 1, 0],
    [">", 2, 0],
    [">", null, 0]
  ]
]

["and",
  [ 
    true,
    true,
    false // ⬅️
  ]
]

false // ❌
```

``` js
const args = {"a": [{"b": 1}, {"b": 2}, {"z": [7, 8, 9]}]}
const statement = ["some", ".a", ["==", ".b", 2]]

// Reduction
["some", [{"b": 1}, {"b": 2}, {"z": [7, 8, 9]}], ["==", ".b", 2]]

["or", 
  [
    ["==", 1, 2],
    ["==", 2, 2],
    ["==", null, 2]
  ]
]

["or", 
  [
    false,
    true, // ⬅️
    false
  ]
]

true // ✅
```

### Nested Quantification

Quantified statements MAY be nested. For example, the below states that someone with the email `fraud@example.com` is required to be among the receipts of every newsletter.

``` js
["every", ".newsletters",
  ["some", ".recipients", 
    ["==", ".email", "fraud@example.com"]]]
```

## Selectors

Selector syntax is closely based on [jq]'s "filters". They operate on an [Invocation]'s `args` object.

Selectors MUST only include the following features:

| Selector Name             | Examples                   | Notes                                                                                           |
|---------------------------|----------------------------|-------------------------------------------------------------------------------------------------|
| Identity                  | `.`                        | Take the entire argument                                                                        |
| Dotted field name         | `.foo`, `.bar0_`           | Shorthand for selecting in a map by key (with exceptions, see below)                            |
| Unambiguous field name    | `["."]`, `["$_*"]`         | Select in a map by arbitrary key                                                                |
| Collection values         | `[]`                       | Expands out all of the children that match the remaining path FIXME double check with the code  |
| Collection index          | `[0]`, `[42]`              | The element at an index. On a map, this is decided by IPLD's key sort order. FIXME double check |
| Collection negative index | `[-1]`, `[-42]`            | The element by index from the end. `-1` is the index for the last element.                      |
| Try                       | `.foo?`, `["nope"]?` | Returns `null` on what would otherwise fail                                                     |

Any selection MAY begin and/or end with a single dot. Multiple dots (e.g. `..`, `...`) MUST NOT be used anywhere in a selector.

The try operator is idempotent, and repeated tries (`.foo???`) MUST be treated as a single one.

[jq] is a much larger language, and includes additional features like pipes, arithmatic, regexes, assignment, recursive descent, and so on which MUST NOT be supported in the UCAN Policy language.

For example, consider the following `args` from an `Invocation`:

``` json
{
  "args": {
    "from": "alice@example.com",
    "to": ["bob@example.com", "carol@not.example.com", "dan@example.com"],
    "cc": ["fraud@example.com"],
    "title": "Meeting Confirmation",
    "body": "I'll see you on Tuesday"
  }
}
```

<table>
<!-- 
NOTE: You cannot add _any_ indentation to this table if you want
      GitHub to render that big multiline code block correctly 
-->
<thead>
<tr>
<td>Selector</td>
<td>Returned Value</td>
</tr>
</thead>
  
<tbody>
<tr>
<td><pre>"."</pre></td>
<td>

```json
{
  "from": "alice@example.com",
  "to": ["bob@example.com", "carol@not.example.com", "dan@example.com"],
  "cc": ["fraud@example.com"],
  "title": "Meeting Confirmation",
  "body": "I'll see you on Tuesday"
}
```
  
</td>
</tr>
  
<tr>
<td><pre>".title"</pre></td>
  
<td>

``` json
"Meeting Confirmation"
```

</td>
</tr>

<tr>
<td><pre>".cc"</pre></td>
<td>

``` json
["fraud@example.com"]
```

</td>
</tr>
    
<tr>
<td><pre>".to[1]"</pre></td>
<td>

``` json
"carol@not.example.com"
```

</td>
</tr>

<tr>
<td><pre>".to[-1]"</pre></td>
<td>

``` json
"dan@example.com"
```

</td>
</tr>

<tr>
<td><pre>".to[99]?"</pre></td>
<td>

``` json
null
```

</td>
</tr>
</tbody>
</table>

## Validation

Validation involves substituting the values from the `args` field into the Policy, and evaluating the predicate. Since Policies are tree structured, selector substitution and predicate evaluation MAY proceed in any order.

If a selector cannot be resolved (there is no value at that path), the associated statement MUST return false, and MUST NOT throw an exception. Note that for consistent semantics, selecting a missing keys on a map MUST return `null` (but nested selectors without a try MUST then fail the predicate).

Below is a step-by-step evaluation example:

``` js
{ // Invocation
  "cmd": "/msg/send",
  "args": {
    "from": "alice@example.com",
    "to": ["bob@example.com", "carol@not.example.com"],
    "title": "Coffee",
    "body": "Still on for coffee"
  },
  // ...
}

{ // Delegation
  "cmd": "/msg",
  "pol": [
    ["==", ".from", "alice@example.com"],
    ["some", ".to", ["match", ".", "*@example.com"]]
  ],
  // ...
}
```

``` js
[ // Extract policy
  ["==", ".from", "alice@example.com"],
  ["some", ".to", ["match", ".", "*@example.com"]]
]

[ // Resolve selectors
  ["==", "alice@example.com", "alice@example.com"],
  ["some", ["bob@example.com", "carol@elsewhere.example.com"], ["match", ".", "*@example.com"]]
]

[ // Expand quantifier
  ["==", "alice@example.com", "alice@example.com"],
  ["or", [
    ["match", "bob@example.com", "*@example.com"]
    ["match", "carol@elsewhere.example.com", "*@example.com"]]
  ]
]

[ // Evaluate first predicate
  true,
  ["or", [
    ["match", "bob@example.com", "*@example.com"]
    ["match", "carol@elsewhere.example.com", "*@example.com"]]]
]

[ // Evaluate second predicate's children
  true,
  ["or", [true, false]]
]


[ // Evaluate second predicate
  true, 
  true
]

// Evaluate top-level `and`
true
```

Any arguments MUST be taken verbatim and MUST NOT be further adjusted. For more flexible validation of Arguments, use [Conditions].

Note that this also applies to arrays and objects. For example, the `to` array in this example is considered to be exact, so the Invocation fails validation in this case:

``` js
// Delegation
{
  "cmd": "/email/send",
  "pol": [
    ["==", ".from", "alice@example.com"],
    ["some", ".to", ["match", ".", "*@example.com"]]
  ]
  // ...
}

// VALID Invocation
{
  "cmd": "/email/send",
  "args": {
    "from": "alice@example.com",
    "to": ["bob@example.com", "carol@elsewhere.example.com"],
    "title": "Coffee",
    "body": "Still on for coffee"
  },
  // ...
}

// INVALID Invocation
{
  "cmd": "/email/send",
  "args": {
    "from": "alice@example.com",
    "to": ["carol@elsewhere.example.com"], // No match for `*@example.com`
    "title": "Coffee",
    "body": "Still on for coffee"
  },
  // ...
}
```

## Semantic Conditions

Other semantic conditions that are not possible to fully express syntactically (e.g. current day of week) MUST be handled as part of Invocation execution. This is considered out of scope of the UCAN Policy language. The RECOMMENDED strategy to express constrains that involve side effects (like day of week) is to include that infromation in the argument shape for that Command (i.e. have a `"day_of_week": "friday"` field).

# Validation

Validation of a UCAN chain MAY occur at any time, but MUST occur upon receipt of an [Invocation] _prior to execution_. While proof chains exist outside of a particular delegation (and are made concrete in [UCAN Invocation]s), each delegate MUST store one or more valid delegations chains for a particular claim.

Each capability has its own semantics, which needs to be interpretable by the [Executor]. Therefore, a validator MUST NOT reject all capabilities when one that is not relevant to them is not understood. For example, if a Condition fails a delegation check at execution time, but is not relevant to the invocation, it MUST be ignored.

If _any_ of the following criteria are not met, the UCAN Delegation MUST be considered invalid:

1. [Time Bounds]
2. [Principal Alignment]
3. [Signature Validation]

Additional constraints MAY be placed on Delegations by specs that use them (notably [UCAN Invocation]).

## Time Bounds

A UCAN's time bounds MUST NOT be considered valid if the current system time is before the `nbf` field or after the `exp` field. This is called the "validity period." Proofs in a chain MAY have different validity periods, but MUST all be valid at execution-time. This has the effect of making a delegation chain valid between the latest `nbf` and earliest `exp`.

``` js
// Pseudocode

const ensureTime = (delegationChain, now) => {
  delegationChain.forEach((ucan) => {
    if (!!ucan.nbf && now < can.nbf) {
      throw new Error(`Delegation is not yet valid, but will become valid at ${ucan.nbf}`)
    }

    if (ucan.exp !== null && now > ucan.exp) {
      throw new Error(`Delegation expired at ${ucan.exp}`)
    }
  })
}
```

## Principal Alignment

In delegation, the `aud` field of every proof MUST match the `iss` field of the UCAN being delegated to. This alignment MUST form a chain back to the Subject for each resource. 

This calculation MUST NOT take into account [DID fragment]s. If present, fragments are only intended to clarify which of a DID's keys was used to sign a particular UCAN, not to limit which specific key is delegated between. Use `did:key` if delegation to a specific key is desired.

``` mermaid
flowchart RL
    invoker((&nbsp&nbsp&nbsp&nbspDan&nbsp&nbsp&nbsp&nbsp))
    subject((&nbsp&nbsp&nbsp&nbspAlice&nbsp&nbsp&nbsp&nbsp))

    subject -- controls --> resource[(Storage)]
    rootCap -- references --> resource

    subgraph Delegations
        subgraph root [Root UCAN]
            subgraph rooting [Root Issuer]
                rootIss(iss: Alice)
                rootSub(sub: Alice)
            end

            rootCap("cap: (Storage, crud/*)")
            rootAud(aud: Bob)
        end

        subgraph del1 [Delegated UCAN]
            del1Iss(iss: Bob) --> rootAud
            del1Sub(sub: Alice)
            del1Aud(aud: Carol)
            del1Cap("cap: (Storage, crud/*)") --> rootCap


            del1Sub --> rootSub
        end

        subgraph del2 [Delegated UCAN]
            del2Iss(iss: Carol) --> del1Aud
            del2Sub(sub: Alice)
            del2Aud(aud: Dan)
            del2Cap("cap: (Storage, crud/*)") --> del1Cap

            del2Sub --> del1Sub
        end
    end

     subgraph inv [Invocation]
        invIss(iss: Dan)
        args("args: [Storage, crud/update, (key, value)]")
        invSub(sub: Alice)
        prf("proofs")
    end

    invIss --> del2Aud
    invoker --> invIss
    args --> del2Cap
    invSub --> del2Sub
    rootIss --> subject
    rootSub --> subject
    prf --> Delegations
```

## Signature Validation

The [Signature] field MUST validate against the `iss` DID from the [Payload].

# Acknowledgments

Thank you to [Brendan O'Brien] for real-world feedback, technical collaboration, and implementing the first Golang UCAN library.

Many thanks to [Hugo Dias], [Mikael Rogers], and the entire DAG House team for the real world feedback, and finding inventive new use cases.

Thank you [Blaine Cook] for the real-world feedback, ideas on future features, and lessons from other auth standards.

Many thanks to [Brian Ginsburg] and [Steven Vandevelde] for their many copy edits, feedback from real world usage, maintenance of the TypeScript implementation, and tools such as [ucan.xyz].

Many thanks to [Christopher Joel] for his real-world feedback, raising many pragmatic considerations, and the Rust implementation and related crates.

Many thanks to [Christine Lemmer-Webber] for her handwritten(!) feedback on the design of UCAN, spearheading the [OCapN] initiative, and her related work on [ZCAP-LD].

Thanks to [Benjamin Goering] for the many community threads and connections to [W3C] standards.

Thanks to [Juan Caballero] for the numerous questions, clarifications, and general advice on putting together a comprehensible spec.

Thank you [Dan Finlay] for being sufficiently passionate about [OCAP] that we realized that capability systems had a real chance of adoption in an ACL-dominated world.

Thanks to the entire [SPKI WG][SPKI/SDSI] for their closely related pioneering work.

Many thanks to [Alan Karp] for sharing his vast experience with capability-based authorization, patterns, and many right words for us to search for.

We want to especially recognize [Mark Miller] for his numerous contributions to the field of distributed auth, programming languages, and computer security writ large.

<!-- Footnotes -->

[^js-num-size]: JavaScript has a single numeric type ([`Number`][JS Number]) for both integers and floats. This representation is defined as a [IEEE-754] double-precision floating point number, which has a 53-bit significand.

<!-- Internal Links -->

[Ability]: #ability
[Envelope]: #delegation-envelope
[Meta]: #meta
[Payload]: #delegation-payload
[Subject]: #subject
[Policy]: #policy-language
[Policy Language]: #policy-language
 
<!-- External Links -->

[Alan Karp]: https://github.com/alanhkarp
[Benjamin Goering]: https://github.com/gobengo
[Blaine Cook]: https://github.com/blaine
[Bluesky]: https://blueskyweb.xyz/
[Brendan O'Brien]: https://github.com/b5
[Brian Ginsburg]: https://github.com/bgins
[Brooklyn Zelenka]: https://github.com/expede 
[CIDv1]: https://github.com/multiformats/cid?tab=readme-ov-file#cidv1
[Christine Lemmer-Webber]: https://github.com/cwebber
[Christopher Joel]: https://github.com/cdata
[Command]: https://github.com/ucan-wg/spec#33-command
[DAG-CBOR]: https://ipld.io/specs/codecs/dag-cbor/spec/
[DID fragment]: https://www.w3.org/TR/did-core/#terminology
[DID]: https://www.w3.org/TR/did-core/
[Dan Finlay]: https://github.com/danfinlay
[Daniel Holmgren]: https://github.com/dholms
[ES256]: https://www.rfc-editor.org/rfc/rfc7518#section-3.4
[EdDSA]: https://en.wikipedia.org/wiki/EdDSA 
[Executor]: https://github.com/ucan-wg/spec#31-roles
[Fission]: https://fission.codes
[Hugo Dias]: https://github.com/hugomrdias
[IEEE-754]: https://ieeexplore.ieee.org/document/8766229
[IPLD]: https://ipld.io/
[Ink & Switch]: https://www.inkandswitch.com/
[Invocation]: https://github.com/ucan-wg/invocation
[Irakli Gozalishvili]: https://github.com/Gozala
[JS Number]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number
[Juan Caballero]: https://github.com/bumblefudge
[Mark Miller]: https://github.com/erights
[Martin Kleppmann]: https://martin.kleppmann.com/
[Mikael Rogers]: https://github.com/mikeal/
[OCAP]: https://en.wikipedia.org/wiki/Object-capability_model
[OCapN]: https://github.com/ocapn/
[Philipp Krüger]: https://github.com/matheus23
[PoLA]: https://en.wikipedia.org/wiki/Principle_of_least_privilege 
[Protocol Labs]: https://protocol.ai/
[RFC 3339]: https://www.rfc-editor.org/rfc/rfc3339
[RFC 8037]: https://www.rfc-editor.org/rfc/rfc8037
[RS256]: https://www.rfc-editor.org/rfc/rfc7518#section-3.3
[Raw data multicodec]: https://github.com/multiformats/multicodec/blob/master/table.csv#L41
[Revocation]: https://github.com/ucan-wg/revocation
[SHA2-256]: https://github.com/multiformats/multicodec/blob/master/table.csv#L9
[SPKI/SDSI]: https://datatracker.ietf.org/wg/spki/about/
[SPKI]: https://theworld.com/~cme/html/spki.html
[Steven Vandevelde]: https://github.com/icidasset
[UCAN Envelope]: https://github.com/ucan-wg/spec/blob/high-level/README.md#envelope
[UCAN Invocation]: https://github.com/ucan-wg/invocation
[UCAN]: https://github.com/ucan-wg/spec
[Varsig]: https://github.com/ChainAgnostic/varsig
[W3C]: https://www.w3.org/
[Witchcraft Software]: https://github.com/expede
[ZCAP-LD]: https://w3c-ccg.github.io/zcap-spec/
[`did:key`]: https://w3c-ccg.github.io/did-method-key/
[`did:plc`]: https://github.com/did-method-plc/did-method-plc
[`did:web`]: https://w3c-ccg.github.io/did-method-web/
[base32]: https://github.com/multiformats/multibase/blob/master/multibase.csv#L13
[dag-json multicodec]: https://github.com/multiformats/multicodec/blob/master/table.csv#L112
[did:key ECDSA]: https://w3c-ccg.github.io/did-method-key/#p-256
[did:key EdDSA]: https://w3c-ccg.github.io/did-method-key/#ed25519-x25519
[did:key RSA]: https://w3c-ccg.github.io/did-method-key/#rsa
[external resource]: https://github.com/ucan-wg/spec#55-wrapping-existing-systems
[jq]: https://jqlang.github.io/jq/
[number zero]: https://n0.computer/
[revocation]: https://github.com/ucan-wg/revocation
[ucan.xyz]: https://ucan.xyz

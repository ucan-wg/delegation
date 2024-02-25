# delegation clause

In 0.x verison of UCANs has an open-ended `nb` field that most users use to constraints delegated capabilities. Open ended nature makes interop very difficult as how to interpert those constraints are not covered by spec. In 1.0 we would like to define standard but extensible constarints language so that implementations can interop unless user space extensions are utilized. By allowing user space extensions of the constraint language we also allow user space to innovate at the cost of interop.

## s-expression based language

### Operators

| Operator     | Function Signature                | Description                                | Example                                        |
| ------------ | --------------------------------- | ------------------------------------------ | ---------------------------------------------- |
| `/${string}` |`[] -> Any`                        | Selects value using JSON Pointer            | `["/foo/0"]`                                  |
|              |`[] -> Any`                        | Selects whole value like "" in JSON Pointer | `[]`                                          |
| `==`         |`[Exp<Any>, Exp<Any>] -> Bool`     | Equality                                    | `["==", ["/size"], 1]`                        |
| `<`          |`[Exp<Num>, Exp<Num>] -> Bool`     | Less than                                   | `["<",  ["/size"], 1024]`                     |
| `<=`         |`[Exp<Num>, Exp<Num>] -> Bool`     | Less than or equal                          | `["<=", ["/size"], 1024]`                     |
| `>`          |`[Exp<Num>, Exp<Num>] -> Bool`     | Greater than                                | `[">", ["@", "/size"], 0]`                    |
| `>=`         |`[Exp<Num>, Exp<Num>] -> Bool`     | Greater than or equal                       | `[">=", ["@", "/size"], 32]`                  |
| `not`        |`[Expr<Bool>] -> Bool`             | Negation                                    | `["not", ["<=", ["/size"], 128]`              |
| `some`       |`[Expr<Any[]>, Expr<Bool>] -> Bool`| Collection predicate                        | `["some", ["/to"], ["==", [], "hi@web.mail"]]`|
| `match`      |`[Expr<Str>, Exp<Str>] -> Bool`    | String wildcard                             | `["match", ["/to"], "*@foo.com"]`             |
| `or`         |`Exp<Bool>[] -> Bool`              | Logical **or** combinator                   | `["or", [">", ["/n"], 0], ["==", ["/n"], 0]]` |
| `and`        |`Exp<Bool>[] -> Bool`              | Logical **and** combinator                  | `["and", [">", ["/n"], 0], ["<", ["/n"], 10]]`|

## Examples

Delegates `email/send` command on `did:key:alice` with a following constraint: Email is not send to address on `gmail.com`, or someone from `fission.codes` should be cc-ed.

```js
{
  "sub": "did:key:alice",
  "iss": "did:key:alice",
  "aud": "did:key:bob",
  "cmd": "email/send",
  "cond": [
    ["or",
      // no recipient should be at `@gmail.com`
      ["every", ["/to"], ["not", ["match", [], "*@gmail.com"]]],
      // or cc should include @fission.codes
      ["some", ["/cc"], ["match", [], "*@fission.codes"]]
    ]
  ]
}
```

## logic programming language

We are also considering logic programming alternative inspired by datomic query styntax.

### concrete proposal for the inital release

Here's a sketch that shows the features

```js
{
  "iss": "did:key:alice",
  // ...
  "policy": [
     ["$ARGS", "foo..bar.[].baz", "?x"],
     [["some", "match"], "?x", "*@example.com"],
      
     ["$ARGS", "foo.quux", "?y"],
     ["?y", ">", 42],
     ["?x", "<", "?y"]
  ]
}
```

Of note:

* Selector syntax based on JSONPath / JSON Poiniter / jQ
  * `"foo..bar.[].baz"` ~ `foo.{*recursive descent*}.bar.{ALL}.baz`
* Variables look like `"?x"` for user defined, and `"$y"` for system/kernel
  * If you have a string that matches these, you have to escape them
  * these give the ability to run predicate logic in the middle of a sector
  * `[[<node>, <path>, "?x"], [<pred>, "?x", <expr>]`
* Declarative matching from `args` onwards
  * Anything of the form `[<node>, <selector>, <value or variable>]`
  * Can avoid namespace conflicts because users define variable names locally
  * Recurses if needed
    * A trivial case: `[["args", "some.path", "?x"], ["?x", "more.path", 42]]`
      * Matches `args.some.path.more.path == 42`
  * Use of `args` to start the path allows us to open up the synatx over time if we want
    * Plus it's very declarative

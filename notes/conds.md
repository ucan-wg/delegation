# delegation clause

In 0.x verison of UCANs has an open-ended `nb` field that most users use to constraints delegated capabilities. Open ended nature makes interop very difficult as how to interpert those constraints are not covered by spec. In 1.0 we would like to define standard but extensible constarints language so that implementations can interop unless user space extensions are utilized. By allowing user space extensions of the constraint language we also allow user space to innovate at the cost of interop.

## s-expression based language

### Operators

| Operator     | Function Signature           | Description                             | Example                            |
| ------------ | ---------------------------- | --------------------------------------- | ---------------------------------- |
| `@` | `String -> JSON`           | Selects invocation argument(s) using JSON Pointer | `["@", "/foo/0"]`                       |
| `==`         | `Expr -> Expr -> Bool`       | Equality                                | `["==", ["key"], 1]`               |
| `<`          | `Expr -> Expr -> Bool`        | Less than                               | `["<", ["@", "size"], 1024]`              |
| `<=`         | `Expr -> Expr -> Bool`        | Less than or equal                      | `["<=", ["@", "/size"], 1024]`             |
| `>`          | `Expr -> Expr -> Bool`        | Greater than                            | `[">", ["@", "/size"], 0]`              |
| `>=`         | `Expr -> Expr -> Bool`        | Greater than or equal                   | `[">=", ["@", "/size"], 32]`             |
| `not`        | `Expr -> Bool`               | Negation                                | `["not", ["<=", ["@", "size"], 128]`     |
| `some`        | `Expr -> Expr -> Bool` | Collection predicate                    | `["some", ["@", "/to"] ["==", ["@"], "hi@fission.codes"]]`     |
| `every`        | `Expr -> Expr -> Bool` | Collection predicate                    | ["every", ["@", "/to"] ["==", ["@"], "hi@fission.codes"]]     |
| `match`    | `Expr -> Expr -> Bool`    | String wildcard                         | `["match", ["@", "/to"], "*@foo.com"]` |

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
      ["every", ["not", ["match", ["/", "to"], "*@gmail.com"]]],
      // or cc should include @fission.codes
      ["some", ["match", ["/", "cc"], "*@fission.codes"]]
    ]
  ]
}
```

## logic programming language

We are also considering logic programming alternative inspired by datomic query styntax.

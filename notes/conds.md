# delegation clause

In 0.x verison of UCANs has an open-ended `nb` field that most users use to constraints delegated capabilities. Open ended nature makes interop very difficult as how to interpert those constraints are not covered by spec. In 1.0 we would like to define standard but extensible constarints language so that implementations can interop unless user space extensions are utilized. By allowing user space extensions of the constraint language we also allow user space to innovate at the cost of interop.

## s-expression based language

### Operators

| Operator     | Function Signature           | Description                             | Example                            |
| ------------ | ---------------------------- | --------------------------------------- | ---------------------------------- |
| `["a", "b"]` | `[String] -> Path`           | Map key (or array index) selector chain | `["key", 2]`                       |
| `==`         | `Expr -> Expr -> Bool`       | Equality                                | `["==", ["key"], 1]`               |
| `<`          | `Path -> Num -> Bool`        | Less than                               | `["<", ["key"], 2.3]`              |
| `<=`         | `Path -> Num -> Bool`        | Less than or equal                      | `["<=", ["key"], 2.3]`             |
| `>`          | `Path -> Num -> Bool`        | Greater than                            | `["<", ["key"], 2.3]`              |
| `>=`         | `Path -> Num -> Bool`        | Greater than or equal                   | `["<=", ["key"], 2.3]`             |
| `not`        | `Expr -> Bool`               | Negation                                | `["not", ["<=", ["key"], 2.3]`     |
| `some`        | `Collection -> Expr -> Bool` | Collection predicate                    | `["some", ["==", ["key"], 42]]`     |
| `every`        | `Collection -> Expr -> Bool` | Collection predicate                    | `["every", ["==", ["key"], 42]]`     |
| `match`    | `Path -> Pattern -> Bool`    | String wildcard                         | `["match", ["key"], "*@foo.com"` |

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

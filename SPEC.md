# bconf

## Table of contents

- [Introduction](#introduction)
- [Comments](#comments)
- [Key-Value Pairs](#key-value-pairs)
- [Keys](#keys)
- [Strings](#strings)
- [Numbers](#numbers)
- [Boolean](#boolean)
- [Null](#null)
- [Blocks](#blocks)
- [Arrays](#arrays)
- [Spread Expressions](#spread-expressions)
- [Statements](#statements)
- [Modifiers](#modifiers)
- [Variables](#variables)
- [Alternatives](#alternatives)
- [Built-ins](#built-ins)
    - [Reserved Keys](#reserved-keys)
        - [import](#import)
        - [export](#export)
        - [extends](#extends)
    - [Modifiers](#modifiers-1)
        - [ref()](#ref)
        - [defined()](#defined)
        - [env()](#env)
        - [string()](#string)
        - [number()](#number)
        - [int()](#int)
        - [float()](#float)
        - [bool()](#bool)
        - [eq()](#eq)
        - [lt()](#lt)
        - [lte()](#lte)
        - [gt()](#gt)
        - [gte()](#gte)

## Introduction

Every bconf document must follow these basic rules:

- Files must be UTF-8 encoded
- Newlines are either LF (`\n`) or CRLF (`\r\n`)
- Whitespace refers to spaces and/or tabs
- bconf is case-sensitive, so `key` is different from `Key`
- Values are not hoisted; variables, imports, etc., must be declared before they are used.
- The root of the document is always a [block](#block) and follows the same rules (eg. semi-colons/newlines as a delimiter). The root is not required to be wrapped in curly-braces, however, it must be the first valid token if so.
- A primitive value is a simple, single value. These are `strings`, `numbers`, `booleans`, or `null`.
- Parsers should preserve the order that key-value pairs, statements, etc. appear in text (ie. parse the file top to bottom), however, insertion order does not need to be preserved when converted to native data structures.

## Comments

Comments start with a double slash (`//`) and continues to the end of the line. Comments may contain any printable Unicode characters and tabs; other control characters are not permitted. Comments are ignored by the parser and should not alter keys or values.

```bconf
// This is a full-line comment
key = "value"  // This is a comment at the end of a line
another = "// This is not a comment because its a string"
```

C-style block comments (`/* ... */`) are not supported.

## Key-Value Pairs

The fundamental building block of a bconf document is the key-value pair. Pairs can be expressed either explicitly or implicitly.

A value must be one of the following types:

- [Strings](#strings)
- [Numbers](#numbers)
- [Boolean](#boolean)
- [Null](#null)
- [Blocks](#blocks)
- [Arrays](#arrays)
- [Modifiers](#modifiers)
- [Variables](#variables)

Every key must be assigned a value. A key declaration without a value is invalid.

```bconf
// INVALID: Open key assignment
open_key =
```

Pairs must be terminated by a newline (or EOF). Comments at the end of the line are valid.

```bconf
// INVALID: Two pairs on the same line.
invalid_key = "value" another_invalid_key = "value"

// VALID: The pair is on its own line (comment is ignored).
valid_key = "value"
```

If a key is declared multiple times in the same scope, the last one wins regardless of the operator used. Any other previously assigned value is overwritten.

```bconf
foo = "first value"
foo = "second value" // This will be the actual value of `foo` since it is the last one

block {
    foo = "third value"
    foo = "fourth value" // This is in a different block scope - only `block.foo` is affected and not `foo` in the root
}

bar = "fifth value"
bar << "sixth value" // This will override `bar` since its the last one

allow = "localhost"
allow "localhost" // This will override `allow` since its the last one
```

### Explicit

An explicit pair consists of a key, an operator (`=` or `<<`), and a value, all on the same line.

The equals operator (`=`) assigns a value to a key.

```bconf
key = "value"
```

The append operator (`<<`) adds a value to an array. If the key doesn't already hold an array, a new one is created.

```bconf
key << "value" // ["value"]
key << "another value" // ["value", "another value"]
```

### Implicit

An implicit pair is a shorthand for assigning a block to a key by omitting the `=` operator.

```bconf
// Equivalent to `block = { ... }`
block {
    key = "value"
}
```

This shorthand is only for blocks. If the operator is omitted for any other value (eg. array, string, number, etc.) it is a [statement](#statement).

A bare key without an operator or value is shorthand for assigning `true`.

```bconf
// Equivalent to `enabled = true`
enabled
port = 8080
```

## Keys

Keys are always interpreted as strings and can be bare or quoted. Keys can be chained using a dot (`.`) or an array index accessor (`[]`) to create nested structures.

Bare keys can contain any printable Unicode character excluding reserved characters (`"$'<>[]{}();/\=,.|`). A key made only of digits (eg. 1234) is still a string.

```bconf
key = "value"
bare-key = "value"
1234 = "value" // The key is the string "1234"
サーバー設定 = { ... } // Valid Unicode characters
```

`true`, `false`, and `null` are considered valid keys and will _always_ resolve to a string. Their respective types do not carry any meaning when they are used as a key. They can still be used as a value.

```bconf
true = "value" // VALID: `true` can be used as a key
false = false // VALID: can be used as a key AND as a value
null = null
```

Quoted keys are a single-line string used as a key. They follow the same rules as string values and are useful for keys containing special characters, or dynamic keys using embedded values. Multi-line strings are invalid.

```bconf
"string key" = "value"
"string key\nwith escape chars" = "value"
"${$some-variable}_value" = "value"
"127.0.0.0" = "value"
"$ref" = "value"

// INVALID: Multi-line string
"""multiline
    string key""" = "value"
```

Dotted keys are a sequence of keys joined by a dot. Any type of key can be used in a dotted key.

```bconf
// This creates a nested block structure.
a.b.c = "value"

// Any key type can be used in the chain.
a."b".c = "value"
```

Keys cannot be empty. This applies to quoted keys that resolve to an empty string.

```bconf
// INVALID: Value assignment with no key
= "value"

// INVALID: Empty quoted key
"" = "value"

$empty_string_var = ""
// INVALID: Quoted key with embedded value resolves to an empty string
"${$empty_string_var}" = "value"
```

### Array indexes

Values in an array can be accessed or assigned by appending an index accessor to a key. The syntax is a zero-based integer wrapped in square brackets (`[]`). Indexes can be positive or negative, where negative integers are used to index values relative from the end of the array. Array indexes can be chained for multi-dimensional array access.

For simplicity when parsing, index integers are expected to follow the same rules when parsing regular integer. This means the prefix `+` is allowed and indexes like `[+1]` are valid in addition to `-` for negative integers.

An index accessor must always be associated with a key; it cannot stand alone. If an index accessor is used on a key that holds a non-array value, such as a block, string, or number, it should create an array at that key.

If the key does not yet exist, a new array is created. If an index is assigned beyond the array's current bounds, the array will be padded with `null` values (or an equivalent) to accommodate the new value at the specified position. This also applies to negative integers. For example, say the key `foo` has an array with a length of 2. Indexing the 4th last element (`foo[-4]`) would effectively insert two elements at the beginning of the array - the first one at index 0 is the actual value and the value at index 1 would be `null` for padding.

```bconf
// Create a new array and assign a value at index 1
// new_list becomes [null, "world"]
new_list[1] = "world"

// Overwrite a value in an existing array
new_list[1] = "bconf" // new_list is now [null, "bconf"]

// Use with dotted keys to create nested structures
data.users[0] = "Alice" // data.users becomes ["Alice"]

// Chain array indexes for multi-dimensional array access
multi_dimensional_index[0][1] = "nested"

// VALID: Negative indexes are ok - this will assign the last value in the array
data.users[-1] = "Bob"

// INVALID: Index accessor must be attached to a key
[0] = "value"

// VALID: Index numbers follow the same rules as regular integers, so `+` is allowed
data.users[+1] = "John"

// Given this non-array value:
not_an_array = "hello"

// VALID: the previous value will be replaced with an array that has "H" at index 0
not_an_array[0] = "H"
```

## Strings

A string can either be single-line or multi-line.

Single-line strings are wrapped in one double quote (`"`). They can contain any Unicode character except for control characters and characters that require escaping (`\`, `"`, `$`).

```bconf
key = "A single-line string with \"escaped quotes\" and a newline\n."
```

Multi-line strings are wrapped in three double quotes (`"""`). They follow the same rules but also permit literal newlines and tabs.

```bconf
key = """
    This is a multi-line string!
    Indentation and newlines are preserved.

    You can also use \"escaped characters\" as well as\nescaped control characters
"""
```

The following escape sequences are reserved. Using any other escape sequence (eg. `\a`) is invalid.

```
\"          - quotation mark
\$          - dollar sign
\\          - backslash
\b          - backspace
\f          - form feed
\n          - new line
\r          - carriage return
\t          - tab
\uXXXX      - U+XXXX
\UXXXXXXXX  - U+XXXXXXXX
```

Any Unicode character may be escaped with the `\uHHHH` or `\UHHHHHHHH` forms and must be Unicode scalar values.

You can embed values in a string using the `${...}` syntax. The fully resolved value must be a primitive such that the values can be converted into a string. Resolved values that are not a primitive value (like blocks and arrays) are invalid. Variables, modifiers and alternative expressions must also resolve to a primitive value.

```bconf
$variable = "embedded value"

// Resolves to "This is a string using an embedded value!"
key = "This is a string using an ${$variable}!"
```

If there is a dollar sign (`$`) not followed immediately by a curly brace (`{`) within a string, the dollar sign should be treated as a regular value (eg. `"the total is $10.99`)

## Numbers

Numbers can be integers or floats. Negative numbers are prefixed with `-` and positive numbers can be prefixed with `+`. If there is no prefix, the number is positive by default. Leading zeros are not allowed (eg. `07` is invalid).

```bconf
int1 = 42
int2 = 0
int3 = -17
int4 = +17

float1 = -1.0
float2 = +1.0
float3 = 3.14159
```

Underscores (`_`) can be used as separators for readability. They cannot be leading, trailing, or appear next to another underscore.

```bconf
int_readable = 1_000_000
float_readable = 5_349.123_456

// INVALID: consecutive underscores
invalid = 1__000
// INVALID: leading underscore
invalid = _1000
// INVALID: trailing underscore
invalid = 1000_
```

Floats support scientific notation using `e` or `E`, followed by an integer exponent. If both a fraction and exponent are used, the exponent must be after the fractional.

```bconf
exponent1 = 1.2e10
exponent2 = 1.2E10
negative_exponent = -2e-2
positive_explicit_exponent = 2e+2
fraction_and_exponent = -5.43e2

// INVALID: Trailing exponent identifier without an integer following
invalid = 4e
```

Leading and trailing decimal points are unsupported and must be surrounded by at least one digit on either side.

```bconf
// INVALID: Leading decimal point
invalid = .4

// INVALID: Trailing decimal point
invalid_float_2 = 4.

// INVALID: Trailing decimal point with exponent
invalid_float_3 = 4.e10
```

Float values `-0.0` and `+0.0` are valid and should map according to IEEE 754. Special float values like NaN and Infinity are not supported.

## Boolean

Booleans are the lowercase tokens `true` and `false`.

```bconf
bool_true = true
bool_false = false
```

## Null

The `null` value represents the absence of a value and must be lowercase.

```bconf
null1 = null
```

For implementations where a direct null equivalent is absent or discouraged (eg. Go's `nil` with non-pointer types), parsers may omit keys with null values from the final output.

## Blocks

A block is a collection of key-value pairs and statements wrapped in curly braces (`{}`). Values can be separated by newlines or semi-colons. Trailing semi-colons are allowed.

```bconf
config {
    enabled;
    host = "localhost";
    port = 8080

    hooks ondeploy {
        channel = "#deployments"
    }
}

// Blocks can also be defined inline.
inline_block = { enabled; port = 8080; }
```

### Scoping

Defining a block also creates a new scope. Scopes only affect the availability of [variables](#variables), other dynamic features such as statements and modifiers are free to ignore scopes. Although dotted keys and blocks may affect the same values, dotted keys **do not** create scopes. Instead, they use the current scope in which they are being defined.

Scopes inherit the scope of the parent, allowing anything that was defined in the parent block to also be accessible the child. Once the parser reaches the end of the block, all variables defined inside that scope will no longer be accessible. Parsers do not need to hang onto the scope created once a block has finished parsing.

## Arrays

An array is an ordered list of values wrapped in square brackets (`[]`). Values _must_ be separated by a comma, and trailing commas are allowed. Arrays can contain a mix of value types.

Arrays can only contain primitive values, blocks, arrays, variables and modifiers. Any other value is invalid.

```bconf
// An array of strings
colors = ["red", "yellow", "green"]

// An array with mixed types
mixed_array = [
    1.2,
    "hello",
    true,
    null,
    ["a", "nested", "array"],
    { foo = "bar" }
]
```

## Spread Expressions

A spread expression inserts the contents of an existing value inline into a surrounding array or block. The syntax is three dots (`...`) immediately followed by a value called the "spread source". A spread source must be one of the following. Any other value is invalid:

- A variable or modifier that resolves to an array or block
- An array literal (`[...]`)
- An block literal (`{ ... }`)

```bconf
$ports = [8080, 8443]
server.hosts = ["localhost", "example.com"]

// VALID: variable as source
all_ports = [...$ports, 9000]

// VALID: modifier as source (assuming getPorts() returns an array)
dynamic = [...getPorts(), 9000]

// VALID: inline array literal as source
all_ports = [...[8080, 8443], 9000]

// VALID: inline block literal as source
server {
    ...{ host = "localhost"; timeout = 30 }
    port = 8080
}
```

> Important: This syntax is designed for composition of blocks and arrays from locally available values and _not_ for directly merging configuration files. See the [extends](#extends) built-in statement instead.

The type of value being spread must match the context it is being spread into. These rules are strict and violations must be rejected at parse time:

- A spread value that resolves to an **array** may only be used inside an **array**. Spreading an array into a block is invalid.
- A spread value that resolves to a **block** may only be used inside a **block**. Spreading a block into an array is invalid.
- A spread value that resolves to any other type is always invalid regardless of context, there is no implicit coercion. For example, a spread value that is a primitive, modifier result that is not an array or block, etc.

```bconf
$ports = [8080, 8443]
$config = { host = "localhost" }

// VALID: array spread into array
all_ports = [...$ports, 9000]

// VALID: block spread into block
server {
    ...$config
    port = 8080
}

// INVALID: array spread into block
server {
    ...$ports
}

// INVALID: block spread into array
all = [...$config, "extra"]

// INVALID: primitive spread
$label = "main"
invalid = [...$label]
```

Spread expressions are not valid as statement arguments. They may only appear inside array literals or blocks.

```bconf
$hosts = ["localhost", "example.com"]

// INVALID: spread cannot be used as a statement argument
allow from ...$hosts

// VALID: construct the array first, then use it
$allowed = [...$hosts, "extra.com"]
allow from $allowed
```

### Ordering

Spread expressions are evaluated in the order they appear. The "last write wins" rule continue to apply, meaning a key or value written after a spread overrides anything introduced by the spread, and a key or value written before a spread is overridden by it if the spread introduces the same key.

```bconf
$base = { host = "localhost"; port = 8080 }

// `port` from $base is overridden by the explicit assignment after the spread
server {
    ...$base
    port = 9000 // wins - port is 9000
}

// `port` set before the spread is overridden by $base
server {
    port = 7000 // loses - overridden by spread
    ...$base // port becomes 8080
}
```

The same applies for arrays. Spread elements are inserted at the position of the expression, preserving the order of elements within the spread value:

```bconf
$extras = [4, 5, 6]
result = [1, 2, 3, ...$extras, 7]  // [1, 2, 3, 4, 5, 6, 7]
```

### Multiple Spreads

Multiple spread expressions are allowed in the same array or block. Each is evaluated in order. For blocks, the same "last write wins" rule applies across all spreads and explicit assignments together.

```bconf
$a = { host = "localhost" }
$b = { port = 8080 }
$c = { timeout = 30; port = 9000 }

// host = "localhost", port = 9000, timeout = 30
// $b sets port to 8080, then $c overrides it to 9000
server {
    ...$a
    ...$b
    ...$c
}
```

```bconf
$first = [1, 2, 3]
$second = [4, 5, 6]

result = [...$first, ...$second]    // [1, 2, 3, 4, 5, 6]
```

## Statements

Statements provide a special syntax for creating configurations that read like a sentence or command. A statement consists of a key followed by a series of space-separated values. Statements exist independently from key-value pairs and do not interact with them — a statement key and a key-value pair can share the same name in the same scope without conflict. Statements are stripped from the resolved document and are purely processed at parse time.

Defining a statement with the same key multiple times is valid.

```bconf
allow from "192.168.1.1"
allow from "10.0.0.0/8"
allow from server.host[0]
```

The values following the key in a statement can be any of the following types:

- Primitives
- Blocks
- Arrays
- Modifiers
- Variables
- Bare keys. Dotted keys and array index accessors are allowed (eg. `foo.bar[0]`)

To avoid ambiguity, implementations must prioritize matching standard value types first. For instance, `true` will always be parsed as a boolean, and `123.1` as a number BEFORE being parsed as a bare key. Only if a value does not match any other type will it be treated as an unquoted string.

Parsers and language servers are encouraged to emit a warning when a key is used as both a statement and a key-value pair within the same scope, as this may indicate unintentional overlap.

Implementations must allow users to register custom statement handlers with the parser. If a statement is encountered and no handler is registered for its key, it is invalid.

Important: This syntax is only considered a statement if the value immediately after the key is not a block (`{`). A key followed directly by a block is an [implicit key-value](#implicit) pair.

## Modifiers

Modifiers act like functions that process or generate a value during parsing. Every modifier must resolve to a value — if a modifier is unrecognized or cannot be resolved, it is invalid. Implementations must allow users to register custom modifiers with the parser.

The syntax is a modifier name followed by zero or more arguments enclosed in parentheses, separated by commas like `modifier_name(argument, 123)`. Trailing commas are allowed. The argument can be any valid value, including a key path with dots and array indexers (eg. `server.ports[0]`).

```bconf
// ref() resolves to the value at the specified path.
default_port = ref(server.port)

// Multiple values can be passed to a modifier. Trailing commas are allowed
timestamp = date("2025-10-09", "UTC",)

// It's valid to use pass no values to a modifier. This may be useful
// for cases where values are reliant on runtime specific information
// and don't require any static values to return a result
active_connections = getNumActiveConnections()
```

Important: This syntax is only considered a modifier if there is an identifier before the opening parenthesis (`(`). Parentheses without a preceding identifier are an [alternatives](#alternatives) expression.

## Variables

Variables let you define a value once and reuse it. They follow the same rules as a standard key-value pair, however, a variable name must start with a dollar sign (`$`) and must be defined before it is used. Variable definitions are not included in the final parsed output.

```bconf
$default_port = 8080
server.port = $default_port // Value becomes 8080.

// INVALID: $hostname is used before it is defined.
server.host = $hostname
$hostname = "localhost"

// VALID: You can redefine a variable. Any use of the variable will now have the value be 443
$default_port = 443

// VALID: Append operator can also be used
$allowed_origins << "test.com"
origins = $allowed_origins // Value becomes ["test.com"]
```

Variables are scoped. A variable defined inside a block is only accessible within that block and its descendants.

```bconf
server.features {
    $apiV2Enabled = true
}

app {
    $port = 3000

    // VALID: $port is in a parent scope.
    server.port = $port

    // INVALID: this may have been defined before for `server.features`, but it went
    // out of scope once that block was finished parsing
    server.features.apiV2 = $apiV2Enabled
}

// INVALID: $port is not accessible in the root scope.
default_port = $port
```

Since only blocks can define new scopes, variables cannot be defined as a segment within a dotted key. For example, `app.$port` would be invalid because dotted keys do not create a scope, so there is no scope for `$port` to be defined in. However, something like `$port.app` would be valid since `$port` is the key being defined in the current scope, with `.app` being a nested extension of it.

## Alternatives

An alternatives expression provides a syntax for expressing a set of potential values called [branches](#branches). It is wrapped in parentheses without a leading identifier and consists of one or more branches separated by a pipe (`|`). A name followed by parentheses is always a [modifier](#modifiers).

A leading pipe is allowed so branches can be formatted across multiple lines.

```bconf
inline_alternative = (branch_a | branch_b | branch_c)
multiline_alternative = (
    | branch_a
    | branch_b
    | branch_c
)
```

Branches can be conditional, where a boolean-producing value (modifier, variable, alternative expression, boolean literal) is followed by an arrow (`=>`) and value. The boolean-producing value is called a `condition` and must _always_ be a boolean. Conditions of other types such as numbers, strings or null are invalid.

When an alternatives expression is used as a condition, it must itself resolve to a boolean. If it resolves to any other type, parsing must fail.

```bconf
$is_prod = eq($env, "prod")

// VALID: $is_prod is a boolean variable
port1 = ($is_prod => 443 | 8080)

// VALID: an alternatives expression used as a condition must resolve to a boolean
port2 = ((eq($env, "prod") => true | false) => 443 | 8080)

// INVALID: the alternatives expression resolves to a number, not a boolean
port3 = ((eq($env, "prod") => 443 | 8080) => 9000 | 3000)
```

Branches are always evaluated strictly left to right, top to bottom. Evaluation stops as soon as a branch produces a value — remaining branches are never considered.

Values in conditional branches must only be used if the condition evaluates to `true`. If it evaluates to `false`, evaluation moves to the next branch.

```bconf
// eq() is a conditional branch because it is followed by =>
port4 = (eq($env, "prod") => 443 | 8080)

// ref() is a regular branch because it is not followed by =>
port5 = (eq($env, "prod") => 443 | ref(server.default_port))

$is_enabled = true

// VALID: $is_enabled is a boolean variable
port6 = ($is_enabled => 443 | 8080)

$portToUse = 8080

// INVALID: $portToUse is not a boolean
port7 = ($portToUse => 443 | 8080)
```

An alternatives expression _must_ always produce a value, including `null`. If no branches produce a value, it is invalid and parsing must fail. For example, an alternatives expression consisting of only conditional branches may not produce a result if no conditions are met, and thus parsing will fail.

```bconf
// VALID: The last branch acts as a fallback/default value if no conditions are met for
// the conditional branches
port1 = (
    | eq($env, "prod") => 443
    | eq($env, "staging") => 3000
    | 8080
)

// INVALID: No fallback is defined, so if `$env` is neither `"prod"` nor `"staging`",
// then no branch produces a value and parsing fails
port2 = (
    | eq($env, "prod") => 443
    | eq($env, "staging") => 3000
)
```

### Nested Alternatives

A branch result can itself be an alternatives expression, allowing conditions to be chained to arbitrary depth. Each nested expression is entirely self-contained — it has no awareness of its parent and does not interact with it in any direction.

This has two important consequences:

1. A nested expression that fails does not fall through to the parent
2. A resolved nested expression immediately produces the final value

If a nested expression has no fallback and no condition is met, parsing fails immediately. The parent expression does not continue evaluating its remaining branches.

In the following example, if `$env` is `"prod"` but `$region` is not `"us-east"`, parsing fails and does not fall back to `"localhost"`. The moment `eq($env, "prod")` is true, the parent is committed and the nested expression is responsible for producing a value.

```bconf
outer = (
    | eq($env, "prod") => (eq($region, "us-east") => "us-east.example.com")
    | "localhost"
)
```

Once a value is returned at any depth, evaluation stops entirely. The parent does not re-evaluate its remaining branches, and no other nested expressions in the chain are considered.

In this example, if `$env` is `"prod"` and `$region` is `"us-east"`, the result is `"us-east.example.com"`. The `eq($env, "staging")` branch and the `"localhost"` fallback are never evaluated.

```bconf
result = (
    | eq($env, "prod") => (eq($region, "us-east") => "us-east.example.com" | "prod.example.com")
    | eq($env, "staging") => "staging.example.com"
    | "localhost"
)
```

Parsers may expose a configurable depth limit for nested alternatives expressions. No hard limit is defined by the spec, but implementations are encouraged to provide a sensible default.

## Built-ins

Parsers are expected to implement the following built-in functionality. These are a mix of reserved keys and modifiers.

### Reserved Keys

#### import

Syntax: `import from "path/to/file.bconf" { $var1; $var2; ... }`

The `import` statement allows for importing variables defined in other bconf files for use within the current file. Only local file paths are supported (relative or absolute). `import` statements must be defined before their variables can be used.

Paths to files must either be absolute or relative. URI schemas such as `file://` or `https://` are unsupported.

```bconf
// INVALID: Cannot use $app_name before it has been imported
name = $app_name

import from "./common.bconf" { $app_name }
name = $app_name
```

It's important to understand the difference between a variable's actual value (the data to be imported and what should actually be used) and the import instruction (the values assigned to the variable inside the import statement block).

- Actual Value: This is the value defined for the variable in the source file. It's the value that will be made available in your current file.
- Import Instruction: This is the value assigned to the variable inside the import statement block. This defines what/how a variable should be imported

```bconf
// common.bconf
// The ACTUAL VALUE of $app_name is the string "My Awesome App".
$app_name = "My Awesome App"

// base.conf
import from "./common.bconf" {
    // This is an IMPORT INSTRUCTION.
    // The shorthand `$app_name` is the same as writing `$app_name = true`.
    $app_name
}

// Now you can use the variable, and it holds its ACTUAL VALUE.
app.name = $app_name // The value here is "My Awesome App"
```

An import instruction must be `true`, `false`, or an `alias statement`. Any other value or statement is invalid. The following is the expected logic for each valid value:

- `true` (shorthand or explicit): Imports the actual value of the variable under its original name.
- `false`: Does not import the variable.
- `$variable as $aliased` (alias statement): Imports the actual value of the variable under a new alias specified after `as`.

```bconf
import from "path/to/file.bconf" {
    // VALID: Imports $shorthand using its original name.
    $shorthand

    // VALID: Explicitly imports $explicit_true using its original name.
    $explicit_true = true

    // VALID: Imports the actual value of $original to be used as $new_alias.
    $original as $new_alias

    // VALID: The value `false` is used to skip imports. Although valid,
    // it is encouraged to simply omit the key entirely.
    $skipped_var = false

    // INVALID: The instruction is a string.
    $invalid_string = "some value"

    // INVALID: The instruction is an block.
    $invalid_block = { a = 1 }
}
```

Variables with the same name cannot be imported more than once or conflict with a variable previously defined with the same name. However, it is valid to redefine a variable with the same name as one that has previously been imported.

```bconf
$foo = "foo"

import from "path/to/file" {
    $variable

    // INVALID: $variable is already being imported above
    $variable

    // VALID: $variable is being aliased as $aliased
    $variable as $aliased

    // INVALID: conflicts with the previously defined $foo variable
    $foo
}

// VALID: $aliased is being redefined after it has been imported (this is discouraged though)
$aliased = "aliased"
```

#### export

Syntax: `export vars { $var1; $var2; ... }`

The `export` statement makes variables from the current file available for other files to `import`.

Inside the block, variable key names can either be a reference to a variable already defined in the file, or an inline definition just for export. Much like the `import` statement, there is the actual value and export instruction.

An export instruction is `true` or an `alias statement`. Any other value can immediately be considered as an inline definition. Any other statement is invalid. The following is the expected logic for each valid export instruction:

- `true` (shorthand or explicit): If there is a variable defined before the export statement with the same name in the document, it is considered a reference. Otherwise, if there is no matching name, it is an inline definition where the value is `true`.
- `$variable as $alias` (alias statement): Exports the actual value of the variable under a new alias specified after `as`.

```bconf
$app_name = "My App"
$port = 8080

export vars {
    // REFERENCE: Exports the $app_name variable ("My App") defined above.
    $app_name

    // REFERENCE: Also exports the $port variable (8080) defined above.
    $port = true

    // INLINE DEFINITION: Defines and exports a new variable, $env.
    // This $env cannot be used elsewhere in this file.
    $env = "production"

    // ALIAS: Export the value for $app_name under the $aliased_name name
    $app_name as $aliased_name

    // This is another way to alias since its assigning the value of $app_name under the name $name.
    $name = $app_name

    // ----------------------------------------------------
    // WARNING: BE CAREFUL WITH ORDER
    // ----------------------------------------------------
    // This is an INLINE DEFINITION, not a reference. Because $is_enabled
    // is only defined below, this creates and exports a new variable
    // named $is_enabled with the value `true`.
    $is_enabled = true
}

$is_enabled = false // This variable is separate from the one exported above.
```

The export block must only contain variable keys. Non-variable keys or duplicate exports of the same name are invalid.

```bconf
export vars {
    // INVALID: `api_key` is not a variable key.
    api_key = "secret"

    $version = "1.0"
    // INVALID: You cannot export the same variable name more than once.
    $version = "2.0"
}
```

#### extends

Syntax: `extends "path/to/base/file.bconf"`

The `extends` statement inserts the resolved contents of the extended file at its location. This means all variables, modifiers, and other built-ins in the extended file must be processed first, leaving only the final key-value structure to be inserted. The "last key wins" rule still applies.

Paths to files must either be absolute or relative. URI schemas such as `file://` or `https://` are unsupported.

```bconf
// base.bconf
env = "development"

// prod.bconf
extends "./base.bconf" // Inserts `env = "development"` here.
env = "production" // Overwrites the value of `env`.

// staging.bconf
env = "staging"

// Since this is extended after the above `env` key-value pair, the contents of base.bconf
// will override it with `env = development`
extends "./base.bconf"
```

### Modifiers

#### ref()

References a value at a specified key path within the document. The argument must be a key path. References to an undefined key or value is invalid. Variables should always be preferred, however, this is useful for situations where the data is not accessible through a variable. For example, referencing a value from an extended document where a variable is not exported.

```bconf
server.port = 8080
default_port = ref(server.port) // Resolves to 8080.

// INVALID: Key has not previously been defined
app_name = ref(app.name)

cors.allowed_origins << "test.com"

// INVALID: Referencing a value that does not exist. Only index 0 has a value
host = ref(cors.allowed_origins[1])
```

Circular dependencies should be rejected and immediately invalidate the document. For example, this is invalid:

```bconf
foo = ref(bar)
bar = ref(foo)
```

Key paths that start with a variable are not expected to work as variables should be resolved first (eg. `ref($foo.bar)`). So the resulting value provided to the `ref()` modifier will not be a key path and is therefore invalid.

#### defined()

Returns `true` if a value has been assigned to the given key path, including `null`. Returns `false` if the key has not been defined at the time the modifier is evaluated. The argument must be a key path.

```bconf
server.port = 8080
defined_modifier1 = defined(server.port) // true

explicit_null = null
defined_modifier2 = defined(explicit_null) // true - null is still a defined value

// INVALID: argument must be a key path
defined_modifier3 = defined("some string")
defined_modifier4 = defined(123)

// false - `undefined_key` has not been assigned a value at this point
defined_modifier5 = defined(undefined_key)
```

#### env()

Reads the value of an operating system environment variable. The argument must be a string. It is invalid if the environment variable does not exist when parsing.

```bconf
environment = env("APP_ENV")
```

#### string()

Converts a value to its string representation.. The following are valid values which can be converted to a string - any other value is invalid:

- `variable` / `modifier`: These should be resolved first and then follow the rules below
- `number`: The value should be quoted (eg. `"123"`, `"123.45"`, `"123.45e6"`)
- `boolean`: The value should be quoted (eg. `"true"`, `"false"`)
- `null`: The value should be quoted (eg. `"null"`)
- `string`: There is nothing needed for converting a string to a string. It should resolve to the same value

```bconf
$variable = 321

string_modifier1 = string(123) // "123"
string_modifier2 = string(true) // "true"
string_modifier3 = string(false) // "false"
string_modifier4 = string(null) // "null"
string_modifier5 = string("some string") // "some string"
string_modifier6 = string($variable) // "321"
```

#### number()

Converts a value to a number, inferring an integer or float type. The following are valid values which can be converted to a number - any other value is invalid:

- `variable` / `modifier`: These should be resolved first and then follow the rules below
- `true`: Always resolves to `1`
- `false`: Always resolves to `0`
- `null`: Always resolves to `0`
- `string`: The value of a string must strictly follow the integer/float syntax. If there is any other character is encountered, it is invalid
- `number`: There is nothing needed for converting a number to a number. It should resolve to the same value

```bconf
$variable = "some string"

number_modifier1 = number(123) // 123
number_modifier2 = number(true) // 1
number_modifier3 = number(false) // 0
number_modifier4 = number(null) // 0
number_modifier5 = number($variable) // Invalid since variable is `"some string"` and an invalid number
number_modifier6 = number("123") // 123
number_modifier7 = number("123.321") // 123.321
number_modifier8 = number("123.321e10") // 123.321e10
number_modifier9 = number("-123_456") // -123456
```

To convert specifically to an integer or float, see [int()](#int) and [float()](#float).

#### int()

Converts a value to an integer. The following are valid values which can be converted to a integer - any other value is invalid:

- `variable` / `modifier`: These should be resolved first and then follow the rules below
- `true`: Always resolves to `1`
- `false`: Always resolves to `0`
- `null`: Always resolves to `0`
- `string`: A string must follow a number syntax. If there is any other character, it is invalid. It should first be converted to its correct number type (float or integer) and then follow the rules below
- `float`: The value is truncated. Exponents must be evaluated first before truncating the value
- `integer`: There is nothing needed for converting an int to an int. It should resolve to the same value

```bconf
$variable = "invalid number"

int_modifier1 = int(3.7) // 3
int_modifier2 = int(true) // 1
int_modifier3 = int(false) // 0
int_modifier4 = int(null) // 0
int_modifier5 = int($variable) // Invalid since variable is `"invalid number"` and an invalid integer
int_modifier6 = int("123") // 123
int_modifier7 = int("123.321") // 123 since the string is first evaluated as a float, so it should be truncated
int_modifier8 = int(456.321e2) // 45632 as the exponent is evaluated and then the result is truncated
int_modifier9 = int("-123_456") // -123456
```

#### float()

Converts a value to a float. The following are valid values which can be converted to a float - any other value is invalid:

- `variable` / `modifier`: These should be resolved first and then follow the rules below
- `true`: Always resolves to `1.0`
- `false`: Always resolves to `0.0`
- `null`: Always resolves to `0.0`
- `string`: A string must follow a number syntax. If there is any other character, it is invalid. It should first be converted to it correct number type (float or integer) and then follow the rules below.
- `integer`: Values resolve to their exact floating point representation (eg. `5` resolves to `5.0`)
- `float`: There is nothing needed for converting a float to a float. It should resolve to the same value

```bconf
$variable = "false"

float_modifier1 = float(3) // 3.0
float_modifier2 = float(true) // 1.0
float_modifier3 = float(false) // 0.0
float_modifier4 = float(null) // 0.0
float_modifier5 = float($variable) // Invalid since variable is `"false"` and an invalid integer
float_modifier6 = float("123") // 123.0 since the string is first evaluated as an int
float_modifier7 = float("123.321") // 123.321
float_modifier8 = float("456.321e10") // 456.321e10
float_modifier9 = float("-123_456") // -123456.0
```

#### bool()

Converts a value to a boolean. The following are valid values which can be converted to a boolean - any other value is invalid:

- `variable` / `modifier`: These should be resolved first and then follow the rules below
- `null`: Always resolves to `false`
- `string`: Non-empty strings always resolve to `true`, while empty strings are `false`
- `number`: Any non-zero number always resolved to `true` (including negatives). Only `0`, `0.0` and `-0.0` resolve to `false`
- `boolean`: There is nothing needed for converting a boolean to a boolean. It should resolve to the same value

```bconf
$variable = "non-empty string!"

bool_modifier1 = bool(-3) // true
bool_modifier1 = bool(3) // true
bool_modifier2 = bool(0) // false
bool_modifier4 = bool(null) // false
bool_modifier5 = bool($variable) // true - since it resolves to a non-empty string
bool_modifier6 = bool("") // false
```

#### eq()

Compare two values for equality, returning a boolean. If the two values are of different types (ie. comparing a string to a number), the values will never be equal and must return `false`. Variables and modifiers must be resolved before the comparison is made.

Blocks and arrays are valid values but are not comparable. Any comparison that involving a block or an array must always return false.

```bconf
eq_modifier1 = eq(1, 1)           // true
eq_modifier2 = eq(1, 2)           // false
eq_modifier3 = eq("foo", "foo")   // true
eq_modifier4 = eq("foo", "bar")   // false
eq_modifier5 = eq(true, true)     // true
eq_modifier6 = eq(true, false)    // false
eq_modifier7 = eq(null, null)     // true
eq_modifier8 = eq(1, "1") // false: comparing different types
eq_modifier9 = eq(true, 1) // false: comparing different types
eq_modifier10 = eq([], []) // false: comparing arrays
eq_modifier11 = eq({}, 1) // false: comparison includes a block
```

#### lt()

Returns a boolean for if the first argument is less than the second. If any value is not a number, it is invalid. Variables and modifiers must be resolved before the comparison is made.

```bconf
lt_modifier1 = lt(1, 2) // true
lt_modifier2 = lt(2, 1) // false
lt_modifier3 = lt(1, 1) // false

// INVALID: arguments must be numbers
lt_modifier4 = lt("a", "b")
```

#### lte()

Returns a boolean for if the first argument is less than or equal to the second. If any value is not a number, it is invalid. Variables and modifiers must be resolved before the comparison is made.

```bconf
lte_modifier1 = lte(1, 2)   // true
lte_modifier2 = lte(1, 1)   // true
lte_modifier3 = lte(2, 1)   // false

// INVALID: arguments must be numbers
lte_modifier4 = lte("a", "b")
```

#### gt()

Returns a boolean for if the first argument is greater than the second. If any value is not a number, it is invalid. Variables and modifiers must be resolved before the comparison is made.

```bconf
gt_modifier1 = gt(2, 1)     // true
gt_modifier2 = gt(1, 2)     // false
gt_modifier3 = gt(1, 1)     // false

// INVALID: arguments must be numbers
gt_modifier4 = gt("b", "a")
```

#### gte()

Returns a boolean for if the first argument is greater than or equal to the second. If any value is not a number, it is invalid. Variables and modifiers must be resolved before the comparison is made.

```bconf
gte_modifier1 = gte(2, 1)   // true
gte_modifier2 = gte(1, 1)   // true
gte_modifier3 = gte(1, 2)   // false

// INVALID: arguments must be numbers
gte_modifier4 = gte("b", "a")
```

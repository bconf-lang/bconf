# bconf

For better configuration files

## Table of contents

-   [Introduction](#introduction)
-   [Comments](#comments)
-   [Key-Value Pairs](#key-value-pairs)
-   [Keys](#keys)
-   [Strings](#strings)
-   [Numbers](#numbers)
-   [Boolean](#boolean)
-   [Null](#null)
-   [Blocks](#blocks)
-   [Arrays](#arrays)
-   [Statements](#statements)
-   [Modifiers](#modifiers)
-   [Variables](#variables)
-   [Built-ins](#built-ins)
    -   [Reserved Keys](#reserved-keys)
        -   [import](#import)
        -   [export](#export)
        -   [extends](#extends)
    -   [Modifiers](#modifiers)
        -   [ref()](#ref)
        -   [env()](#env)
        -   [string()](#string)
        -   [number()](#number)
        -   [int()](#int)
        -   [float()](#float)
        -   [bool()](#bool)

## Introduction

Every bconf document must follow these basic rules:

-   Files must be UTF-8 encoded
-   Newlines are either LF (`\n`) or CRLF (`\r\n`)
-   Whitespace refers to spaces and/or tabs
-   bconf is case-sensitive, so `key` is different from `Key`
-   Values are not hoisted; variables, imports, etc., must be declared before they are used.
-   The root of the document is always a [block](#block) and follows the same rules (eg. semi-colons/newlines as a delimiter). The root is not required to be wrapped in curly-braces, however, it must be the first valid token if so.
-   A primitive value is a simple, single value. These are `strings`, `numbers`, `booleans`, or `null`.
-   Parsers should preserve the order that key-value pairs, statements, etc. appear in text (ie. parse the file top to bottom), however, insertion order does not need to be preserved when converted to native data structures.

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

-   [Strings](#strings)
-   [Numbers](#numbers)
-   [Boolean](#boolean)
-   [Null](#null)
-   [Blocks](#blocks)
-   [Arrays](#arrays)
-   [Modifiers](#modifiers)
-   [Variables](#variables)

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

Values in an array can be accessed or assigned by appending an index accessor to a key. The syntax is a non-negative, zero-based integer wrapped in square brackets (`[]`). Array indexes can be chained for multi-dimensional array access.

An index accessor must always be associated with a key; it cannot stand alone. It is invalid to use an index accessor on a key that holds a non-array value, such as a block, string, or number.

If the key does not yet exist, a new array is created. If an index is assigned beyond the array's current bounds, the array will be padded with `null` values (or an equivalent) to accommodate the new value at the specified position.

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

// INVALID: Index cannot be negative
data.users[-1] = "Bob"

// INVALID: Index accessor must be attached to a key
[0] = "value"

// Given this non-array value:
not_an_array = "hello"

// INVALID: Cannot use an index accessor on a string
not_an_array[0] = "H"
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
\\          - backslash
\$          - dollar sign
\b          - backspace
\f          - form feed
\n          - new line
\r          - carriage return
\t          - tab
\uXXXX      - U+XXXX
\UXXXXXXXX  - U+XXXXXXXX
```

Any Unicode character may be escaped with the `\uHHHH` or `\UHHHHHHHH` forms and must be Unicode scalar values.

You can embed values in a string using the `${...}` syntax. The fully resolved value must be a primitive such that the values can be converted into a string. Resolved values that are not a primitive value (like blocks and arrays) are invalid. Variables and modifiers must also resolve to a primitive value.

```bconf
$variable = "embedded value"

// Resolves to "This is a string using an embedded value!"
key = "This is a string using an ${$variable}!"
```

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

## Statements

Statements provide a special syntax for creating configurations that read like a sentence or command. A statement consists of a key followed by a series of space-separated values.

Defining a statement with the same key multiple times appends a new list of values to a 2D array, rather than overwriting the previous entry.

For example, this:

```bconf
allow from "192.168.1.1"
allow from "10.0.0.0/8"
allow from server.host[0]
```

Is parsed into a structure like this (represented as JSON):

```json
{
	"allow": [
		["from", "192.168.1.1"],
		["from", "10.0.0.0/8"],
		["from", "server.host[0]"]
	]
}
```

The values following the key in a statement can be any of the following types:

-   Primitives
-   Blocks
-   Arrays
-   Modifiers
-   Variables
-   Bare keys
    - These are parsed as a string. Dotted keys and array index accessors are allowed, with the full key being resolved to a string (eg. `foo.bar[0]`) 

To avoid ambiguity, implementations must prioritize matching standard value types first. For instance, `true` will always be parsed as a boolean, and `123.1` as a number BEFORE being parsed as a bare key. Only if a value does not match any other type will it be treated as an unquoted string.

Important: This syntax is only considered a statement if the value immediately after the key is not a block (`{`). A key followed directly by a block is an [implicit key-value](#implicit) pair.

## Modifiers

Modifiers act like functions that process or generate a value during parsing.

The syntax is a modifier name followed by a single argument enclosed in parentheses, like `modifier_name(argument)`. The argument can be any valid value, including a key path with dots and array indexers (eg. `server.ports[0]`).

If the parser recognizes the modifier name (eg. a built-in like `ref()`), it replaces the modifier with the resolved value.

```bconf
// ref() resolves to the value at the specified path.
default_port = ref(server.port)
```

If the parser encounters an unrecognized modifier, it treats it as a custom modifier. Instead of resolving it, the parser serializes it as a tuple: `[modifier_name, argument]`. When the argument is a key path, it's serialized as a string. This allows the application itself to implement custom logic after the file has been parsed.

For example:

```bconf
// This allows an application to implement its own date parsing
// It would be parsed to: ["date", "2025-10-09"]
last_login = date("2025-10-09")

// A custom modifier using a key path as an argument
// It would be parsed to: ["to_upper", "app.name"]
capitalized_name = to_upper(app.name)
```

Implementations may optionally allow users to register their own custom modifiers with the parser, enabling them to be resolved at parse-time. However, this is not required.

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

Since only blocks can define new scopes, variables cannot be defined inside a dotted key. For example, `app.$port` would be invalid since no new scope is created and any scope previously created for `app` would no longer available. However, something like `$port.app` would be valid since it is defining a variable in the current scope.

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

-   Actual Value: This is the value defined for the variable in the source file. It's the value that will be made available in your current file.
-   Import Instruction: This is the value assigned to the variable inside the import statement block. This defines what/how a variable should be imported

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

-   `true` (shorthand or explicit): Imports the actual value of the variable under its original name.
-   `false`: Does not import the variable.
-   `$variable as $aliased` (alias statement): Imports the actual value of the variable under a new alias specified after `as`.

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

-   `true` (shorthand or explicit): If there is a variable defined before the export statement with the same name in the document, it is considered a reference. Otherwise, if there is no matching name, it is an inline definition where the value is `true`.
-   `$variable as $alias` (alias statement): Exports the actual value of the variable under a new alias specified after `as`.

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

#### env()

Reads the value of an operating system environment variable. The argument must be a string. It is invalid if the environment variable does not exist when parsing.

```bconf
environment = env("APP_ENV")
```

#### string()

Converts a value to its string representation.. The following are valid values which can be converted to a string - any other value is invalid:

-   `variable` / `modifier`: These should be resolved first and then follow the rules below
-   `number`: The value should be quoted (eg. `"123"`, `"123.45"`, `"123.45e6"`)
-   `boolean`: The value should be quoted (eg. `"true"`, `"false"`)
-   `null`: The value should be quoted (eg. `"null"`)
-   `string`: There is nothing needed for converting a string to a string. It should resolve to the same value

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

-   `variable` / `modifier`: These should be resolved first and then follow the rules below
-   `true`: Always resolves to `1`
-   `false`: Always resolves to `0`
-   `null`: Always resolves to `0`
-   `string`: The value of a string must strictly follow the integer/float syntax. If there is any other character is encountered, it is invalid
-   `number`: There is nothing needed for converting a number to a number. It should resolve to the same value

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

-   `variable` / `modifier`: These should be resolved first and then follow the rules below
-   `true`: Always resolves to `1`
-   `false`: Always resolves to `0`
-   `null`: Always resolves to `0`
-   `string`: A string must follow a number syntax. If there is any other character, it is invalid. It should first be converted to its correct number type (float or integer) and then follow the rules below
-   `float`: The value is truncated. Exponents must be evaluated first before truncating the value
-   `integer`: There is nothing needed for converting an int to an int. It should resolve to the same value

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

-   `variable` / `modifier`: These should be resolved first and then follow the rules below
-   `true`: Always resolves to `1.0`
-   `false`: Always resolves to `0.0`
-   `null`: Always resolves to `0.0`
-   `string`: A string must follow a number syntax. If there is any other character, it is invalid. It should first be converted to it correct number type (float or integer) and then follow the rules below.
-   `integer`: Values resolve to their exact floating point representation (eg. `5` resolves to `5.0`)
-   `float`: There is nothing needed for converting a float to a float. It should resolve to the same value

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

-   `variable` / `modifier`: These should be resolved first and then follow the rules below
-   `null`: Always resolves to `false`
-   `string`: Non-empty strings always resolve to `true`, while empty strings are `false`
-   `number`: Any non-zero number always resolved to `true` (including negatives). Only `0`, `0.0` and `-0.0` resolve to `false`
-   `boolean`: There is nothing needed for converting a boolean to a boolean. It should resolve to the same value

```bconf
$variable = "non-empty string!"

bool_modifier1 = bool(-3) // true
bool_modifier1 = bool(3) // true
bool_modifier2 = bool(0) // false
bool_modifier4 = bool(null) // false
bool_modifier5 = bool($variable) // true - since it resolves to a non-empty string
bool_modifier6 = bool("") // false
```

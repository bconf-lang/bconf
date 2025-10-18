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
-   [Objects](#objects)
-   [Arrays](#arrays)
-   [Nodes](#nodes)
-   [Tagged Values](#tagged-values)
-   [Variables](#variables)
-   [Built-ins](#built-ins)
    -   [Reserved Keys](#reserved-keys)
        -   [import](#import)
        -   [export](#export)
        -   [extends](#extends)
    -   [Tags](#tags)
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

Key Terms:

-   A primitive value is a simple, single value. These are `strings`, `numbers`, `booleans`, or `null`.
-   A block value is a container for other values. These are `objects` and `arrays`

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
-   [Objects](#objects)
-   [Arrays](#arrays)
-   [Tagged Values](#tagged-values)
-   [Variables](#variables)

Every key be assigned a value. A key declaration without a value is invalid

```bconf
// INVALID: No value is assigned to the key.
open_key =
```

Pairs must be terminated by a newline (or EOF). Comments at the end of the line are valid

```bconf
// INVALID: Two pairs on the same line.
invalid_key = "value" another_invalid_key = "value"

// VALID: The pair is on its own line (comment is ignored).
valid_key = "value"
```

If a key is declared multiple times in the same scope, the last one wins. Any other previously assigned value is overwritten.

```bconf
foo = "first value"
foo = "second value" // This will be the actual value of `foo` since it is the last one

object {
    foo = "third value"
    foo = "fourth value" // This is in a different object scope - only `object.foo` is affected and not `foo` in the root
}
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

An implicit pair is a shorthand for assigning a block value (an object or array) to a key by omitting the `=` operator.

```bconf
// Equivalent to `object = { ... }`
object {
    key = "value"
}

// Equivalent to `array = [ ... ]`
array [
    "value"
]
```

This shorthand is only for block values. If the operator is omitted for any other value (eg. string, number, etc.) it is a [node](#nodes).

## Keys

Keys are always interpreted as strings and can be alphanumeric, quoted, or dynamic. Keys can be chained using a dot (`.`) to create nested objects.

Alphanumeric keys can contain ASCII letters, ASCII numbers, underscores, and dashes (`A-Za-z0-9_-`). A key made only of digits (e.g., 1234) is still a string.

```bconf
key = "value"
alphanumeric-key = "value"
1234 = "value" // The key is the string "1234"
```

Quoted keys are string literals (single or multi-line) used as keys. They follow the same rules as string values and are useful for keys containing special characters. Although multiline strings are allowed, it is discouraged as it can be harder to read and should only be used when necessary.

```bconf
"string key" = "value"
"string key\nwith escape chars" = "value"
"${$some-variable}_value" = "value"
"127.0.0.0" = "value"
"$ref" = "value"
"""multiline
    string key""" = "value"
```

Dynamic keys are wrapped in square brackets (`[]`) and use a variable or tagged value that resolves to a string. If the variable or tagged value cannot be resolved to a string, it is invalid.

```bconf
$variable = "bar"
$object_variable = {
    foo = "bar"
}

foo {
    // The key becomes "bar", making this equivalent to `foo.bar = "baz"`
    [$variable] = "baz"

    // The key becomes the string "123", making this `foo.123 = "value"`
    [string(123)] = "value"

    // INVALID: Variable used for the dynamic key is an object, which cannot resolve to a valid key
    [$object_variable] = "object variable"
}
```

Dotted keys are a sequence of keys joined by a dot. Any type of key can be used in a dotted key.

```bconf
// This creates a nested object structure.
a.b.c = "value"

// Any key type can be used in the chain.
a."b".[$c] = "value"
```

Keys cannot be empty. This applies to quoted strings and dynamic keys that resolve to an empty string.

```bconf
// INVALID: Value assignment with no key
= "value"

// INVALID: Empty single-line string
"" = "value"

// INVALID: Empty multi-line string
"""""" = "value"

$empty_string_var = ""
// INVALID: Variable resolves to an empty string
[$empty_string_var] = "value"
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

The following escape sequences are reserved. Using any other escape sequence (e.g., `\a`) is invalid.

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

You can embed values in a string using the `${...}` syntax. The resolved value must be a string or a type that can be converted to one (like a number or boolean). Resolved values that cannot be converted to a string (like objects and arrays) are invalid.

```bconf
$variable = "substitution"

// Resolves to "This is a string using value substitution!"
key = "This is a string using value ${$variable}!"
```

## Numbers

Numbers can be integers or floats. Negative numbers are prefixed with `-`. Leading zeros are not allowed (e.g., `07` is invalid).

```bconf
int1 = 42
int2 = 0
int3 = -17

float1 = -1.0
float2 = 3.14159
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

Special float values like NaN and Infinity are not supported.

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

For implementations where a direct null equivalent is absent or discouraged (e.g., Go's `nil` with non-pointer types), parsers may omit keys with null values from the final output.

## Objects

An object is a collection of key-value pairs wrapped in curly braces (`{}`). Pairs can be separated by newlines or commas. Trailing commas are allowed. A key inside an object without a value is shorthand for `key = true`.

```bconf
config {
    enabled  // Shorthand for `enabled = true`
    host = "localhost",
    port = 8080
}

// Objects can also be defined inline.
inline_object = { enabled, port = 8080 }
```

## Arrays

An array is an ordered list of values wrapped in square brackets (`[]`). Like objects, values can be separated by newlines or commas, and trailing commas are allowed. Arrays can contain a mix of value types.

Arrays can only contain primitive values, block values, variables and tagged values. Any other value is invalid.

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

## Nodes

Nodes are a special syntax for creating configurations that read like sentences. A node consists of a key followed by a series of values separated by whitespace.

Defining a node with the same key multiple times appends a new list of values, resulting in a 2D array, rather than overwriting.

For example, this:

```bconf
allow from "192.168.1.1"
allow from "10.0.0.0/8"
```

Is parsed into a structure like this (represented as JSON):

```json
{
	"allow": [
		["from", "192.168.1.1"],
		["from", "10.0.0.0/8"]
	]
}
```

The following are valid values for nodes:

-   [Keys](#keys)
    -   **NOTE** as a value, these are considered unquoted strings and not actual keys. The same character rules for keys still apply.
-   [Strings](#strings)
-   [Numbers](#numbers)
-   [Boolean](#boolean)
-   [Null](#null)
-   [Objects](#objects)
-   [Arrays](#arrays)
-   [Tagged Values](#tagged-values)
-   [Variables](#variables)

Important: This syntax is only considered a node if the value immediately after the key is not an object (`{`) or array (`[`). A key followed directly by a block is an [implicit key-value](#implicit) pair.

## Tagged Values

Tagged values act like functions that process or generate a value. The syntax is a tag followed by a value in parentheses, like `tag_name(value)`.

If the parser recognizes the tag name (e.g., a built-in like `ref()`), it replaces the tag with the resolved value.

```bconf
// ref() resolves to the value at the specified path.
default_port = ref(server.port)
```

If the parser encounters an unrecognized tag, it serializes it as a tuple: `[tag_name, value]`. This allows applications to handle custom logic.

For example, a custom date tag:

```bconf
last_login = date("2025-10-09")
```

Would be parsed into a structure like:

```json
{
	"last_login": ["date", "2025-10-09"]
}
```

Implementations may optionally provide an option to extend a lookup table to resolve custom tags during parsing, but this is not a required.

## Variables

Variables let you define a value once and reuse it. A variable name must start with a dollar sign (`$`) and must be defined before it is used. Variable definitions are not included in the final parsed output.

```bconf
$default_port = 8080
server.port = $default_port // Value becomes 8080.

// INVALID: $hostname is used before it is defined.
server.host = $hostname
$hostname = "localhost"
```

Variables are scoped. A variable defined inside an object is only accessible within that object and its descendants.

```bconf
app {
    $port = 3000

    // VALID: $port is in a parent scope.
    server.port = $port
}

// INVALID: $port is not accessible in the root scope.
default_port = $port
```

## Built-ins

Implementations are expected to implement the following built-in functionality. These are a mix of reserved keys and tags.

### Reserved Keys

#### import

Syntax: `import from "path/to/file.bconf" { $var1, $var2, ... }`

The `import` node allows for imports variables defined in other bconf files for use within the current file. Only local file paths are supported (relative or absolute). `import` nodes must defined before their variables can be used.

```bconf
// INVALID: Cannot use $app_name before it has been imported
name = $app_name

import from "./common.bconf" { $app_name }
name = $app_name
```

It's important to understand the difference between a variable's actual value (the data to be imported and what should actually be used) and the import instruction (the values assigned to the variable inside the object).

-   Actual Value: This is the value defined for the variable in the source file. It's the value that will be made available in your current file.
-   Import Instruction: This is the value assigned to the variable inside the import node object.

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

An import instruction must be `true`, `false`, or an `as()` tagged value. Any other value is invalid. The following is the expected logic for each valid value:

-   `true` (shorthand or explicit): Imports the actual value of the variable under its original name.
-   `false`: Does not import the variable.
-   `as($variable_key)`: Imports the actual value of the variable under a new alias specified as the value inside the parentheses.

```bconf
import from "path/to/file.bconf" {
    // VALID: Imports $shorthand using its original name.
    $shorthand

    // VALID: Explicitly imports $explicit_true using its original name.
    $explicit_true = true

    // VALID: Imports $original and renames it to $new_alias.
    $original = as($new_alias)

    // VALID: The value `false` is used to skip imports. Although valid,
    // it is encouraged to simply omit the key entirely.
    $skipped_var = false

    // INVALID: The instruction is a string.
    $invalid_string = "some value"

    // INVALID: The instruction is an object.
    $invalid_object = { a = 1 }
}
```

#### export

Syntax: `export vars { $var1, $var2, ... }`

The `export` node makes variables from the current file available for other files to `import`.

Inside the object, variable key names can either be a reference to a variable already defined in the file, or an inline definition just for export. Much like the `import` node, there is the actual value and export instruction. The only valid export instruction is `true`. Any other value can immediately be considered an inline definition. Parsers must implement the following logic when working with the `true` export instruction:

-   If there is a variable defined with the same name in the document, it is considered a reference.
-   Otherwise, if there is no matching name, it is an inline definition where the value is `true`.

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

    // ALIAS: Exports the value of $app_name under the name $name.
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

The export object must only contain variable keys. Non-variable keys or duplicate exports of the same name are invalid.

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

It inserts the resolved contents of the extended file at its location. This means all variables, tags, and other built-ins in the extended file must be processed first, leaving only the final key-value structure to be inserted. The "last key wins" rule still applies.

```bconf
// -- base.bconf
env = "development"

// -- prod.bconf
extends "./base.bconf" // Inserts `env = "development"` here.
env = "production" // Overwrites the value of `env`.

// staging.bconf
env = "staging"

// Since this is extended after the above `env` key-value pair, the contents of base.bconf
// will override it with `env = development`
extends "./base.bconf"
```

### Tags

#### ref()

References a value at a specified key path within the document. Referencing a non-existent key is invalid. Variables should always be preferred, however, this is useful for situations where the data is not accessible through a variable. For example, referencing a value from an extended document where a variable is not exported.

```bconf
server.port = 8080
default_port = ref(server.port) // Resolves to 8080.
```

#### env()

Reads the value of an operating system environment variable. The variable name must be a string. It is invalid if the environment variable does not exist.

```bconf
environment = env("APP_ENV")
```

#### string()

Converts a value to its string representation.. The following are valid values which can be converted to a string - any other value is invalid:

-   `variable`: These should be resolved first and then follow the rules below
-   `number`: The value should be quoted (eg. `"123"`, `"123.45"`, `"123.45e6"`)
-   `boolean`: The value should be quoted (eg. `"true"`, `"false"`)
-   `null`: The value should be quoted (eg. `"null"`)
-   `string`: There is nothing needed for converting a string to a string. It should resolve to the same value

```bconf
$variable = 321

string_tag1 = string(123) // "123"
string_tag2 = string(true) // "true"
string_tag3 = string(false) // "false"
string_tag4 = string(null) // "null"
string_tag5 = string("some string") // "some string"
string_tag6 = string($variable) // "321"
```

#### number()

Converts a value to a number, inferring integer or float type. The following are valid values which can be converted to a number - any other value is invalid:

-   `variable`: These should be resolved first and then follow the rules below
-   `true`: Always resolves to `1`
-   `false`: Always resolves to `0`
-   `null`: Always resolves to `0`
-   `string`: The value of a string must strictly follow the integer/float syntax. If there is any other character is encountered, it is invalid
-   `number`: There is nothing needed for converting a number to a number. It should resolve to the same value

```bconf
$variable = "some string"

number_tag1 = number(123) // 123
number_tag2 = number(true) // 1
number_tag3 = number(false) // 0
number_tag4 = number(null) // 0
number_tag5 = number($variable) // Invalid since variable is `"some string"` and an invalid number
number_tag6 = number("123") // 123
number_tag7 = number("123.321") // 123.321
number_tag8 = number("123.321e10") // 123.321e10
number_tag9 = number("-123_456") // -123456
```

To convert specifically to an integer or float, see [int()](#int) and [float()](#float).

#### int()

Converts a value to an integer. The following are valid values which can be converted to a integer - any other value is invalid:

-   `variable`: These should be resolved first and then follow the rules below
-   `true`: Always resolves to `1`
-   `false`: Always resolves to `0`
-   `null`: Always resolves to `0`
-   `string`: A string must follow a number syntax. If there is any other character, it is invalid. It should first be converted to it correct number type (float or integer) and then follow the rules below.
-   `float`: The value is truncated. Exponents must be evaluated first before truncating the value.
-   `integer`: There is nothing needed for converting an int to an int. It should resolve to the same value

```bconf
$variable = "invalid number"

int_tag1 = int(3.7) // 3
int_tag2 = int(true) // 1
int_tag3 = int(false) // 0
int_tag4 = int(null) // 0
int_tag5 = int($variable) // Invalid since variable is `"invalid number"` and an invalid integer
int_tag6 = int("123") // 123
int_tag7 = int("123.321") // 123 since the string is first evaluated as a float, so it should be truncated
int_tag8 = int(456.321e2) // 45632 as the exponent is evaluated and then the result is truncated
int_tag9 = int("-123_456") // -123456
```

#### float()

Converts a value to a floating-point number. The following are valid values which can be converted to a float - any other value is invalid:

-   `variable`: These should be resolved first and then follow the rules below
-   `true`: Always resolves to `1.0`
-   `false`: Always resolves to `0.0`
-   `null`: Always resolves to `0.0`
-   `string`: A string must follow a number syntax. If there is any other character, it is invalid. It should first be converted to it correct number type (float or integer) and then follow the rules below.
-   `integer`: Values resolve to their exact floating point representation (eg. `5` resolves to `5.0`)
-   `float`: There is nothing needed for converting a float to a float. It should resolve to the same value

```bconf
$variable = "false"

float_tag1 = float(3) // 3.0
float_tag2 = float(true) // 1.0
float_tag3 = float(false) // 0.0
float_tag4 = float(null) // 0.0
float_tag5 = float($variable) // Invalid since variable is `"false"` and an invalid integer
float_tag6 = float("123") // 123.0 since the string is first evaluated as an int
float_tag7 = float("123.321") // 123.321
float_tag8 = float("456.321e10") // 456.321e10
float_tag9 = float("-123_456") // -123456.0
```

#### bool()

Converts a value to a boolean. The following are valid values which can be converted to a boolean - any other value is invalid:

-   `variable`: These should be resolved first and then follow the rules below
-   `null`: Always resolves to `false`
-   `string`: Non-empty strings always resolve to `true`, while empty strings are `false`
-   `number`: Any non-zero number always resolved to `true` (including negatives). Only `0` resolves to `false`
-   `boolean`: There is nothing needed for converting a boolean to a boolean. It should resolve to the same value

```bconf
$variable = "non-empty string!"

bool_tag1 = bool(-3) // true
bool_tag1 = bool(3) // true
bool_tag2 = bool(0) // false
bool_tag4 = bool(null) // false
bool_tag5 = bool($variable) // true - since it resolves to a non-empty string
bool_tag6 = bool("") // false
```

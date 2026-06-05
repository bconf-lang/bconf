# bconf

## Table of Contents

- [Introduction](#introduction)
- [Comments](#comments)
- [Identifiers](#identifiers)
- [Key-Value Pairs](#key-value-pairs)
    - [Keys](#keys)
    - [Assignment Operators](#assignment-operators)
- [Values](#values)
    - [Blocks](#blocks)
    - [Arrays](#arrays)
    - [Strings](#strings)
    - [Numbers](#numbers)
    - [Booleans](#booleans)
    - [Null](#null)
    - [Variables](#variables)
    - [Modifiers](#modifiers)
    - [Directives](#directives)
    - [Resolvers](#resolvers)
- [Spreads](#spreads)
- [Built-ins](#built-ins)
    - [Directives](#built-in-directives)
        - [@import](#import)
        - [@export](#export)
        - [@extends](#extends)
    - [Modifiers](#built-in-modifiers)
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
        - [gt()](#gt)
        - [not()](#not)
        - [all()](#all)
        - [some()](#some)
        - [coalesce()](#coalesce)
        - [contains()](#contains)
        - [len()](#len)
        - [keys()](#keys-1)
        - [values()](#values-1)
        - [entries()](#entries)
        - [trim()](#trim)
        - [lower()](#lower)
        - [upper()](#upper)
        - [min()](#min)
        - [max()](#max)
        - [clamp()](#clamp)

---

## Introduction

A bconf document resolves unambiguously to a hash map. The following rules apply to all documents:

- Files must be UTF-8 encoded.
- Newlines may be either LF (`\n`) or CRLF (`\r\n`).
- The term "whitespace" refers only to spaces and tabs, not newlines.
- bconf is case-sensitive: `key` and `Key` are distinct identifiers.
- Documents are parsed and evaluated strictly top to bottom. No values are hoisted. Key-value pairs preserve the order in which they appear in the document, though implementations are not required to preserve insertion order when converting to native data structures.

The root of every document is implicitly a [block](#blocks). Wrapping the root in curly braces is optional. If curly braces are used, the opening brace must be the first valid token in the document and it must always have a matching closing brace. Only [comments](#comments) may appear before the opening brace or after the closing brace.

```bconf
// VALID: Curly braces are optional. The root is parsed as a block.
foo = "bar"
bar = "baz"
```

```bconf
// VALID: Comments may appear outside the braces.
{
    foo = "bar"
}
// This comment is fine here.
```

```bconf
foo = "bar"

// INVALID: A key-value pair appears before the opening brace.
{
    bar = "baz"
}
```

```bconf
// INVALID: No closing brace.
{
    foo = "bar"
```

```bconf
foo = "bar"
} // INVALID: Unexpected closing brace with no matching opening brace.
```

---

## Comments

A comment begins with `//` and extends to the end of the line. Comments may contain any printable Unicode character and tabs. Other control characters are not permitted. Comments are ignored during evaluation and have no effect on the resolved output.

```bconf
// This is a full-line comment.
key = "value"  // This is an inline comment.
another = "// This is not a comment because it is inside a string."
```

C-style block comments (`/* ... */`) are not supported.

---

## Identifiers

An identifier is a sequence of one or more printable Unicode characters excluding the following reserved characters:

```
"  $  '  (  )  ,  . /  ;  <  =  >  ?  @  [  ]  `  {  |  }
```

Identifiers are always interpreted as strings. An identifier made entirely of digits (e.g. `1234`) is the string `"1234"`, not the integer `1234`.

```bconf
key = "value"
bare-key = "value"       // The key is the string "bare-key"
1234 = "value"           // The key is the string "1234", not an integer
サーバー設定 = "value"    // Unicode characters are valid in identifiers
```

---

## Key-Value Pairs

Key-value pairs are the fundamental building blocks of a bconf document and are the **only** statements that appear in the resolved output. A key-value pair consists of a [key](#keys) on the left, an [operator](#assignment-operators) in the middle, and a [value](#values) on the right.

```bconf
key = "value"
```

There are two shorthands. The assignment operator may be omitted when the value is a [block](#blocks) — the block follows the key directly. A standalone key with no operator at all is shorthand for assigning `true` to that key.

```bconf
// Shorthand for: enabled = true
enabled

// Shorthand for: config = { host = "localhost"; port = 8080 }
config {
    host = "localhost"
    port = 8080
}
```

Assigning a key that already has a value is valid. The new value replaces the old one. This is referred to as the **last-assign-wins** rule.

```bconf
foo = "bar"       // foo is "bar"
foo = "replaced"  // foo is now "replaced"
```

### Keys

A key is one of the following:

- A **bare key**: any [identifier](#identifiers).
- A **quoted key**: a [single-line string](#strings). A quoted key must not resolve to an empty string. Multi-line strings are not valid as keys.
- A **variable key**: a [variable identifier](#variables), prefixed with `$` or `$$`.

```bconf
key = "value"                  // Bare key
"another key" = "value"        // Quoted key with a space
"${$prefix}_name" = "value"    // Quoted key with an embedded value

// INVALID: Empty string as a key.
"" = "value"

// INVALID: Quoted key resolves to an empty string.
$empty = ""
"${$empty}" = "value"
```

Bare and quoted keys are equivalent in the resolved output. `example` and `"example"` refer to the same key and follow the last-assign-wins rule. Variable keys and directive identifiers are never equivalent to quoted keys because neither appears in the resolved output.

```bconf
// Equivalent keys — second overwrites the first.
example = "first"
"example" = "second"

// No conflict. $env is a variable and is not in the output.
// The quoted key "$env" is a regular key and is included in the output.
$env = "prod"
"$env" = "staging"
```

The identifiers `true`, `false`, and `null` are valid bare keys. When used as a key they are always treated as an identifier, never as a value.

```bconf
true = "value"   // VALID: `true` used as a key
false = false    // VALID: `false` as a key and as its value
null = null      // VALID: `null` as a key and as its value
```

#### Dotted Keys

Bare and quoted keys can be chained with dots (`.`) to create **dotted keys**. Each segment separated by a dot creates a nested block at that level if one does not already exist. The value is assigned at the depth of the final key.

```bconf
// Equivalent to: a = { b = { c = "value" } }
a.b.c = "value"

// Any key type can be used in the chain.
a."b".c = "value"
```

#### Array Indexes

An **array index** can follow any key segment to assign a value at a specific position in an array. The syntax is the key followed by an integer wrapped in square brackets (`[0]`). Indexes can be chained for multi-dimensional access.

Indexes are zero-based. Positive indexes count from the start of the array; negative indexes count from the end. The `+` prefix is allowed on positive indexes. If the key does not already hold an array, one is created. If the index is beyond the current length of the array, the array is extended and padded with `null` as needed. This applies to both positive and negative out-of-bounds indexes.

```bconf
// Creates an array and assigns at index 0: ["first"]
foo[0] = "first"

// Appends to the existing array: ["first", "second"]
foo[1] = "second"
foo[+1] = "second"  // The + prefix is fine.

// Index 4 is out of bounds, so the array is padded:
// ["first", "second", null, null, "third"]
foo[4] = "third"

// Negative index: counts from the end, inserting at position 3.
// ["first", "second", null, "fourth", "third"]
foo[-2] = "fourth"

// Negative index out of bounds: array is padded at the front.
// ["fifth", null, "first", "second", null, "fourth", "third"]
foo[-7] = "fifth"

// Multi-dimensional access: [null, [null, "nested"]]
multi[1][1] = "nested"
```

### Assignment Operators

There are three assignment operators.

The **assignment** operator (`=`) assigns the value to the key, replacing any existing value.

```bconf
key = "value"
```

The **push** operator (`<<`) appends a value to the end of an array held at the key. If the key does not already hold an array, a new one is created first, replacing any existing value if needed.

```bconf
key << "first"   // key is ["first"]
key << "second"  // key is ["first", "second"]
```

The **conditional assignment** operator (`?=`) assigns the value to the key only if the key has not already been assigned a value. If the key already has a value, the statement is a no-op.

```bconf
// key has no prior value, so this assigns "default".
key ?= "default"

// assigned already has a value, so this is ignored.
assigned = "original"
assigned ?= "ignored"  // assigned remains "original"
```

---

## Values

A value is one of the following: a [primitive](#strings), a [block](#blocks), an [array](#arrays), or a [dynamic value](#variables). Dynamic values are [variables](#variables), [modifiers](#modifiers), [directives](#directives), and [resolvers](#resolvers).

### Blocks

A **block** is a collection of statements enclosed in curly braces (`{}`). Valid statements inside a block are [key-value pairs](#key-value-pairs), [variable declarations](#variables), [directives](#directives), [spread expressions](#spreads), and [comments](#comments). Only key-value pairs appear in the resolved output; all other statement types are excluded.

Each statement is terminated by either a newline or a semicolon. Semicolons can also be used to place multiple statements on the same line.

```bconf
config {
    host = "localhost"
    port = 8080          // newline-delimited
    tls = true; debug = false  // semicolon-delimited on one line
}
```

A semicolon may appear on its own with no preceding content, forming an **empty statement**. Empty statements are valid anywhere a statement is allowed and produce no output. This means any number of consecutive semicolons is valid and simply ignored by the parser.

```bconf
config {
    host = "localhost";;;  // trailing semicolons after a statement are fine
    ;;;;                   // standalone semicolons with no content are also fine
    port = 8080
}
```

#### Scopes

Every block creates a new **scope**. Variables declared inside a block are accessible only within that block and any blocks nested inside it. They are not visible to the parent block. Key-value assignments and directive side effects are also local to the block's scope.

A block inherits the scope of its parent, so variables declared in an outer block are accessible in inner blocks.

```bconf
$env = "prod"

server {
    $timeout = 30

    // VALID: $env is accessible from the parent scope.
    active = (eq($env, "prod") => true | false)

    // VALID: $timeout is declared in this block.
    timeout = $timeout
}

// INVALID: $timeout is not in scope here.
default_timeout = $timeout
```

### Arrays

An **array** is an ordered list of values enclosed in square brackets (`[]`). Items must be separated by commas. Trailing commas are allowed. An array may contain any mix of value types, including nested arrays and blocks.

Arrays do not create a new scope.

```bconf
// An array of strings.
colors = ["red", "yellow", "green"]

// A mixed-type array spanning multiple lines.
mixed = [
    1.2,
    "hello",
    true,
    null,
    ["a", "nested", "array"],
    { foo = "bar" },
]
```

### Strings

There are two string types: **single-line** and **multi-line**.

A **single-line string** is enclosed in double quotes (`"`). It may contain any Unicode character except literal control characters, a standalone backslash (`\`), double quote (`"`), or an unescaped dollar sign (`$`) followed immediately by `{` (which begins an [embedded value](#embedded-values)).

```bconf
greeting = "Hello, world!"
escaped = "A quote: \" and a backslash: \\"
```

A **multi-line string** is enclosed in three double quotes (`"""`). It follows the same rules as a single-line string, but literal newlines and tab characters are also permitted.

```bconf
message = """
    This is a multi-line string.
    Indentation and newlines are preserved as written.

    Escape sequences like \n and \" still work here too.
"""
```

#### Escape Sequences

The following escape sequences are valid in both single-line and multi-line strings. Any other escape sequence is invalid.

```
\"          quotation mark
\$          dollar sign
\\          backslash
\b          backspace
\f          form feed
\n          newline
\r          carriage return
\t          tab
\uXXXX      Unicode scalar value (U+XXXX)
\UXXXXXXXX  Unicode scalar value (U+XXXXXXXX)
```

The codepoint provided to `\uXXXX` or `\UXXXXXXXX` must be a valid [Unicode scalar value](https://unicode.org/glossary/#unicode_scalar_value).

#### Embedded Values

A value can be embedded directly inside a string using the syntax `${...}`. The sequence begins with `${` and ends with `}`. Dynamic values can be used inside the embedded expression, however, the value must resolve to a [string](#strings), [number](#numbers), [boolean](#booleans), or [null](#null). Blocks and arrays are not valid inside an embedded sequence.

A bare `$` not immediately followed by `{` is treated as a literal dollar sign character and does not require escaping.

```bconf
$name = "world"

// Resolves to: "Hello, world!"
greeting = "Hello, ${$name}!"

// The $ here is just a literal character — no embedded sequence.
price = "The total is $10.99"

// INVALID: $config resolves to a block, which is not allowed in an embed.
$config = { host = "localhost" }
invalid = "Value: ${$config}"
```

### Numbers

Numbers are either **integers** or **floats**. A leading `-` indicates a negative number. A leading `+` is also accepted for positive numbers. If neither prefix is present, the number is positive. Leading zeros are not allowed (e.g. `07` is invalid).

```bconf
int1 = 42
int2 = 0
int3 = -17
int4 = +17

float1 = 3.14
float2 = -1.0
float3 = +1.0
```

Underscores (`_`) may be used as digit separators for readability. An underscore may not appear at the start or end of a number, and consecutive underscores are not allowed.

```bconf
readable_int   = 1_000_000
readable_float = 5_349.123_456

// INVALID: Consecutive underscores.
invalid1 = 1__000

// INVALID: Leading underscore (this would be parsed as an identifier, not a number).
invalid2 = _1000

// INVALID: Trailing underscore.
invalid3 = 1000_
```

Floats support scientific notation using `e` or `E` followed by an integer exponent. When both a fractional part and an exponent are present, the exponent must come after the fraction.

```bconf
sci1 = 1.2e10
sci2 = 1.2E10
sci3 = -2e-2
sci4 = 2e+2
sci5 = -5.43e2

// INVALID: Exponent with no following integer.
invalid1 = 4e

// INVALID: Fraction after the exponent.
invalid2 = 4e1.2
```

Both the integer and fractional parts of a float must have at least one digit on their respective sides of the decimal point. Leading and trailing decimal points are not allowed.

```bconf
// INVALID: Leading decimal point.
invalid3 = .4

// INVALID: Trailing decimal point.
invalid4 = 4.

// INVALID: Trailing decimal point before an exponent.
invalid5 = 4.e10
```

The values `-0.0` and `+0.0` are valid floats and map according to IEEE 754. The values `+0` and `-0` are valid integers; their sign prefix may be dropped. Special float values such as `NaN` and `Infinity` are not supported.

### Booleans

Boolean values are the lowercase tokens `true` and `false`.

```bconf
flag_on  = true
flag_off = false
```

### Null

`null` is a valid value used to represent the intentional absence of a value. It must be lowercase.

```bconf
nothing = null
```

For implementations where `null` has no natural equivalent, implementations may choose their own representation. For example, keys with a `null` value may be omitted from the output, or a zero-value may be substituted.

### Variables

A **variable** is an [identifier](#identifiers) prefixed with `$` or `$$`. Variables are declared using the same syntax as [key-value pairs](#key-value-pairs) but are **not** included in the resolved output. Using a variable as a value is equivalent to writing its value inline at that point. Variables always produce a deep copy of their value.

A variable must be declared before it is used. Using a variable before its declaration is invalid.

There are two kinds of variables:

A **mutable variable** is prefixed with a single `$`. Its value may be reassigned any number of times after the initial declaration.

A **constant** is prefixed with `$$`. Once declared, its value may not be reassigned. This restriction applies to the variable itself as well as any nested values accessed via dotted keys or array indexes.

```bconf
// Declaring and using a constant.
$$CONFIG = { port = 8080 }
server.port = $$CONFIG.port  // 8080

// INVALID: Cannot reassign a constant.
$$CONFIG = { port = 443 }

// INVALID: Cannot modify a constant's nested values.
$$CONFIG.port = 443

// Declaring and reassigning a mutable variable.
$domain = "localhost"
$domain = "example.com"  // VALID: mutable variables can be reassigned.

// INVALID: $host is used before it is declared.
server.host = $host
$host = "localhost"

// The push and conditional assignment operators also work with variable declarations.
$ports << 8080    // $ports is [8080]
$ports << 443     // $ports is [8080, 443]

$env ?= "dev"     // $env is "dev" if not already set

// Variables can be assigned to other variables.
$mutable_config = $$CONFIG
$mutable_config.port = 443 // VALID: $$CONFIG was deeply copied to a mutable variable
```

Variables follow the same scoping rules as all other block declarations — they are accessible within the block they are declared in and any nested blocks, but not in the parent block. See [Scopes](#scopes).

When using a variable as a value, the path must always begin with a variable identifier. A path that starts with a bare key is always invalid as a value even if it later references a variable.

```bconf
$config = { $subkey = 123; host = "localhost" }

// VALID: Path begins with a variable.
host = $config.host

// INVALID: Path begins with a bare key, not a variable.
host = config.$subkey
```

### Modifiers

A **modifier** is a callable expression that produces a value at evaluation time. Modifiers must resolve to a [string](#strings), [number](#numbers), [boolean](#booleans), [null](#null), [block](#blocks), or [array](#arrays). If a modifier is unrecognized or fails to resolve, it is invalid. Implementations should allow users to register custom modifiers.

The syntax is a modifier name (an [identifier](#identifiers)) followed immediately by parentheses enclosing zero or more comma-separated arguments. Trailing commas are allowed. Arguments may be any [value](#values) or a [spread expression](#spreads). Dynamic values used as arguments must be resolved first before the modifier is evaluated.

If a spread is used as an argument, it must resolve to an array before the modifier is called. The elements of that array are then passed as individual arguments at the position of the spread, as if they had been written inline.

```bconf
// A modifier with a single argument.
default_port = ref("server.port")

// A modifier with multiple arguments. Trailing comma is fine.
timestamp = date("2025-10-09", "UTC",)

// A modifier with no arguments.
connections = getNumActiveConnections()

// Using a spread to expand an array into arguments.
$hosts = ["localhost", "test.com"]
// Equivalent to: getHost("localhost", "test.com", "example.com")
host = getHost(...$hosts, "example.com")

// $hosts is evaluated first, so the argument provided
// is the array and not the variable identifier
default_hosts = getHost($hosts)
```

A modifier is only recognized as such when an identifier appears immediately before the opening parenthesis. Parentheses without a preceding identifier are a [resolver](#resolvers), not a modifier.

### Directives

A **directive** is an [identifier](#identifiers) prefixed with `@`, followed by any number of whitespace separated arguments. Unlike modifier arguments, directive arguments are not limited to values and can include bare [identifiers](#identifiers) as well.

```bconf
@allow "192.168.1.1" "192.168.1.2"
@deny ssh
```

Directive arguments are passed to the directive implementation as unevaluated syntax — they are not resolved at the point of the call. This means variables, modifiers, resolvers, and spreads are handed to the directive as-is. Implementations should expose an API that allows a directive to resolve argument values on demand, evaluated against the scope in which the directive was called.

```bconf
$ports = [80, 443]

// @allow receives a spread expression referencing $ports.
// It does not automatically resolve $ports.
@allow $ports
```

A directive may optionally produce a value. A directive that produces no value is distinct from one that produces `null` — the former has no output at all, while the latter produces an explicit [null](#null) value.

When a directive is used as a statement and produces a **block**, the statements from that block are merged into the parent block at the point of the directive call, as if they had been written inline. If the directive produces no value, evaluation continues normally.

```bconf
// --- base.bconf
$bar = "bar"
foo = $bar

// --- main.bconf
// Assuming @insert parses the file and returns it as a block,
// its statements (including variable declarations) are merged here.
@insert "./base.bconf"

// VALID: $bar is now in scope, merged in from base.bconf.
baz = $bar

// This would overwrite "bar" previously at `foo` since `foo` was defined in base.bconf
// and had that declaration merged into the document.
foo = "baz"
```

When a directive produces any value other than a block (including `null`), the value is treated as if it were written inline at the point of the directive call. If the inline value would be invalid syntax in that context, it is an error.

```bconf
// Assuming @array returns [123, "test", 321], this is invalid because
// an array literal is not a valid statement in a block.
foo {
    @array 123 "test" 321
}
```

When a directive is used as a **value** in a key-value pair, the produced value is used directly. If the directive produces no value, it is treated as `null`.

```bconf
// Assuming @exec returns the stdout of the command.
$working_dir = @exec pwd

// Assuming @emit produces no value, this is the same as: result = null
result = @emit "ok"
```

### Resolvers

A **resolver** is an expression containing one or more **branches** that are evaluated in order. The resolver produces the value of the first branch whose condition is met. Resolvers are enclosed in parentheses (`()`), with branches separated by a pipe (`|`). A leading pipe is allowed.

```bconf
// A resolver with a single branch — always resolves to "value".
result = ("value")

// A resolver with multiple branches.
result = ("branch_a" | "branch_b" | "branch_c")

// A leading pipe is allowed for multi-line formatting.
result = (
    | "branch_a"
    | "branch_b"
    | "branch_c"
)

// Multi-line formatting is still possible without a leading pipe
result = ("branch_a"
    | "branch_b"
    | "branch_c"
)
```

#### Conditional Branches

A branch can be made conditional by prefixing it with a **condition** followed by `=>`. The condition must be a boolean-producing expression — which are a [boolean literal](#booleans), a [variable](#variables) holding a boolean, a [modifier](#modifiers), [directive](#directive), or another [resolver](#resolvers) that produces a boolean. Conditions that resolve to any other type (string, number, null, etc.) are invalid. Conditions are never truthy or falsy.

A branch with no condition is always evaluated as if the condition has been met.

```bconf
$env = "prod"
$is_prod = eq($env, "prod")

// A conditional branch with a boolean variable as the condition.
port = ($is_prod => 443 | 8080)

// An unconditional branch — always resolves to 80.
port = (80)

// eq($env, "staging") => 3000 is unreachable because the unconditional branch above it always wins.
port = (
    | $is_prod => 443
    | 8080
    | eq($env, "staging") => 3000
)

// INVALID: $env contains a string, not a boolean
port = ($env => 3000 | 8080)
```

Branches are evaluated strictly left to right. Evaluation stops as soon as a branch produces a value; remaining branches are never evaluated.

```bconf
// The last branch is an unconditional fallback.
port = (
    | eq($env, "prod")     => 443
    | eq($env, "staging")  => 3000
    | 8080
)

// Without a fallback, this resolver may fail to produce a value.
// If $env is neither "prod" nor "staging", parsing fails.
port = (
    | eq($env, "prod")     => 443
    | eq($env, "staging")  => 3000
)
```

A resolver used as a condition must itself resolve to a boolean.

```bconf
// VALID: The inner resolver produces a boolean.
port = (
    | (eq($env, "prod") => true | false) => 443
    | 8080
)

// INVALID: The inner resolver produces a number, not a boolean.
port = (
    | (eq($env, "prod") => 443 | 8080) => 9000
    | 3000
)
```

#### Nested Resolvers

A branch value can itself be a resolver, allowing conditions to be chained to any depth. Each resolver is self-contained — it has no awareness of its parent. If a nested resolver has no fallback and no condition is met, evaluation fails immediately. The parent resolver does not continue to its remaining branches.

```bconf
// If $env is "prod" but $region is not "us-east", parsing fails.
// The "localhost" fallback in the outer resolver is never reached.
result = (
    | eq($env, "prod") => (eq($region, "us-east") => "us-east.example.com")
    | "localhost"
)
```

Once any resolver at any depth produces a value, evaluation stops entirely.

```bconf
// If $env is "prod" and $region is "us-east", the result is "us-east.example.com".
// The "staging" branch and "localhost" fallback are never evaluated.
result = (
    | eq($env, "prod") => (
        | eq($region, "us-east") => "us-east.example.com"
        | "prod.example.com"
    )
    | eq($env, "staging") => "staging.example.com"
    | "localhost"
)
```

Implementations may expose a configurable depth limit for nested resolvers. No hard limit is defined by this spec, but a sensible default is encouraged.

---

## Spreads

A **spread expression** inserts the contents of a value into the array or block that contains it. The syntax is three dots (`...`) followed immediately by a **spread source**. A spread source must be one of the following:

- A [variable](#variables), [modifier](#modifiers), [directive](#directives), or [resolver](#resolvers) that produces an array or block.
- An inline [array](#arrays) literal (`[...]`).
- An inline [block](#blocks) literal (`{ ... }`).

```bconf
$ports = [8080, 8443]

// Spreading a variable into an array.
all_ports = [...$ports, 9000]           // [8080, 8443, 9000]

// Spreading the result of a modifier into an array.
dynamic = [...getPorts(), 9000]

// Spreading an inline array literal.
more_ports = [...[8080, 8443], 9000]    // [8080, 8443, 9000]

// Spreading an inline block literal into a block.
server {
    ...{ host = "localhost"; timeout = 30 }
    port = 8080
}
```

The type of the spread source must match the context it is spread into. Spreading an array into a block, or a block into an array, is invalid. Spreading any other type (a string, a number, etc.) is always invalid regardless of context. There is no implicit coercion.

```bconf
$ports  = [8080, 8443]
$config = { host = "localhost" }

// VALID: Array spread into an array.
all_ports = [...$ports, 9000]

// VALID: Block spread into a block.
server {
    ...$config
    port = 8080
}

// INVALID: Array spread into a block.
server {
    ...$ports
}

// INVALID: Block spread into an array.
all = [...$config, "extra"]

// INVALID: A string is not a valid spread source.
$label = "main"
invalid = [...$label]
```

Spreads are not valid as an argument to a [directive](#directives).

```bconf
$hosts = ["localhost", "example.com"]

// INVALID: Spread directly as a directive argument.
@allow ...$hosts

// VALID: Spread inside an array, which is the argument.
@allow [...$hosts, "extra.com"]
```

### Ordering

Spread expressions are evaluated in the order they appear. Inside a block, the [last-assign-wins](#key-value-pairs) rule applies across all spreads and explicit assignments together, so position relative to other statements determines which value wins.

```bconf
$base = { host = "localhost"; port = 8080 }

// The explicit `port` after the spread overwrites the one from $base.
server {
    ...$base
    port = 9000  // port is 9000
}

// The explicit `port` before the spread is overwritten by $base.
server {
    port = 7000  // overridden
    ...$base     // port becomes 8080
}
```

For arrays, spread elements are inserted at the position of the expression, preserving their internal order.

```bconf
$extras = [4, 5, 6]
result = [1, 2, 3, ...$extras, 7]  // [1, 2, 3, 4, 5, 6, 7]
```

### Multiple Spreads

Multiple spread expressions are allowed in the same array or block. Each is evaluated in order, with the last-assign-wins rule continuing to apply across all of them.

```bconf
$a = { host = "localhost" }
$b = { port = 8080 }
$c = { timeout = 30; port = 9000 }

// $b sets port to 8080, then $c overrides it to 9000.
// Result: { host = "localhost", port = 9000, timeout = 30 }
server {
    ...$a
    ...$b
    ...$c
}
```

```bconf
$first  = [1, 2, 3]
$second = [4, 5, 6]

result = [...$first, ...$second]  // [1, 2, 3, 4, 5, 6]
```

---

## Built-ins

### Built-in Directives

#### @import

Syntax: `@import <path: string> [variables: block]`
Returns: `block`

```@bconf
@import "./config.bconf" { $host }
```

Imports exported variables from another bconf file into the current scope. The `path` argument must resolve to a string for a file path (relative or absolute), meaning if a dynamic value is used for the argument, it must be resolved first. The `variables` argument must be a block and destructures which variables are imported. If the `variables` argument is omitted, all exported variables from the file are imported. `@import` returns a block containing the variable declarations needed to bring the imported variables into scope.

The `variables` block only processes variable assignments and ignores any other statement inside the block. A variable can be aliased by writing `$alias = $original`, where the left side is the new name in the current scope and the right side is the name as exported from the file. If no alias is given, the variable is imported under its original name.

Importing variables follows the same [last-assign-wins](#key-value-pairs) rule as every other key-value pair, meaning it is valid to import a variable which conflicts with one already defined in the document. It is also valid to reassign an imported variable once it has been imported. While there may be cases where this is intentional, implementations should raise a warning if either of these scenarios are detected.

```bconf
// --- config.bconf (exports $host and $port)
@export { $host = "localhost"; $port = 8080 }

// --- main.bconf

// Import all exported variables into scope.
@import "./config.bconf"
server.host = $host   // "localhost"
server.port = $port   // 8080

// Import only $host, aliased as $server_host.
// No warning is raised since it only imports $host as $server_host instead.
@import "./config.bconf" { $server_host = $host }
server.host = $server_host  // "localhost"

// Use as a value to namespace the imports under a key.
$cfg = @import "./config.bconf"
server.host = $cfg.host   // "localhost"
server.port = $cfg.port   // 8080

// WARNING: $host was an imported variable and is being reassigned
$host = "example.com"

// WARNING: when importing "./config.bconf" again since it would reassign
// $host and $port which are already declared in scope.
@import "./config.bconf"
```

If no file exists at the path provided, parsing must fail.

```bconf
// INVALID: Assuming `./invalid.bconf` does not exist, importing will fail
@import "./invalid.bconf" { $host }
server.port = $host
```

---

#### @export

Syntax: `@export <variables: block>`
Returns: `void`

```bconf
@export { $host; $port }
```

Exports variables from the current file. The block lists the variables to be exported so they can be imported. Inside the `variables` block, only shorthand boolean statements are used to reference variables defined in scope. If the variable being referenced does not exist, it is invalid. Inline variable assignments are allowed inside the block. Any other statement inside the block is ignored. `@export` does not return a value.

All variables exported must have a unique name, it is invalid otherwise.

```bconf
$host = "localhost"
$port = 8080

// Export variables that are already declared.
@export { $host; $port }

// Declare and export inline.
@export {
    $env = "prod"
    $localhost = $host    // basically aliasing $host since its just assigning it to another name
    foo = "bar"           // ignored — not a variable assignment
}

// INVALID: $host is already exported
@export {
    $host = "localhost"
}
```

---

#### @extends

Syntax: `@extends <path: string>`
Returns: `block`

```
@extends "./base.bconf"
```

Inserts the fully resolved output of another bconf file into the current scope at the point of the directive call. The referenced file is evaluated in complete isolation — its variables, directives, and other dynamic expressions are resolved within that file's own scope and have no effect on the current document. It also does not inherit anything from the scope the `@extends` directive is called in.

Returns a block containing only the key-value assignments from the resolved output of the file.

```bconf
// --- base.bconf
$env = "prod"
host = (eq($env, "prod") => "example.com" | "localhost")
port = 8080

// --- extended.bconf
host = "test.com"

// --- main.bconf
// Inserts the resolved output of "./base.bconf".
// Variables like $env are not visible here — only the resolved key-value pairs are inserted.
@extends "./base.bconf"

// Overrides port from the extended file.
port = 9000

extended = {
    // Only inserts the resolved output of "./extended.bconf" into this scope.
    // This means `extended.host` would be "test.com" while `host` in the root
    // remains "example.com" (coming from the base.bconf file)
    @extends "./extended.bconf"
}
```

If no file exists at the path provided, parsing must fail.

```bconf
// INVALID: Assuming `./invalid.bconf` does not exist, extending will fail
@extends "./invalid.bconf"
server.port = 8080
```

---

### Built-in Modifiers

#### ref()

Syntax: `ref(key: string)`
Returns `any`

```
ref("foo.bar[0]")
```

Looks up a value in the current document by key path and returns a deep copy of it. The argument must be a string formatted as a valid bconf key path (including dotted keys and array indexes). `null` is considered a defined value and is returned as-is.

Fails if no value has been assigned at the specified key path at the time of the call.

```bconf
server.port = 8080

default_port = ref("server.port")   // 8080

// Dotted paths and array indexes are supported.
hosts[0] = "localhost"
first_host = ref("hosts[0]")        // "localhost"

// INVALID: No value has been assigned at this path.
missing = ref("server.timeout")
```

---

#### defined()

Syntax: `defined(key: string)`
Returns: `boolean`

```
defined("server.timeout")
```

Checks whether a value has been assigned at the given key path at the time of the call. The argument must be a string formatted as a valid bconf key path. Returns `true` if any value — including `null` — has been assigned at that path, and `false` if the key does not exist.

```bconf
server.port = 8080
server.timeout = null

has_port    = defined("server.port")     // true
has_timeout = defined("server.timeout")  // true — null is a valid value
has_host    = defined("server.host")     // false — key does not exist
```

---

#### env()

Syntax: `env(name: string)`
Returns `string`

```
env("APP_ENV")
```

Reads an environment variable by name and returns its value as a string. Fails if the environment variable is not set.

```bconf
// Reads the APP_ENV environment variable.
environment = env("APP_ENV")

// INVALID: Assuming `INVALID_NAME` does not exist
$env = env("INVALID_NAME")
```

---

#### string()

Syntax: `string(arg: boolean | number | null | string)`
Returns: `string`

```
string(123)
```

Converts the argument to its string representation. The argument must resolve to a primitive value — blocks and arrays are always invalid.

The conversion rules are:

- `string`: returned as-is.
- `integer`: the base-10 string representation, including any sign prefix (e.g. `-17` --> `"-17"`).
- `float`: the base-10 string representation, including decimal point and any exponent (e.g. `3.14` --> `"3.14"`).
- `true` --> `"true"`, `false` --> `"false"`.
- `null` --> `"null"`.

```bconf
$world = "world"
a = string(42)      // "42"
b = string(-1.5)    // "-1.5"
c = string(true)    // "true"
d = string(null)    // "null"
e = string("hello") // "hello"
f = string($world)  // "world"

// INVALID: Blocks and arrays cannot be converted to strings.
$cfg = { host = "localhost" }
invalid = string($cfg)
```

---

#### number()

Syntax: `number(arg: boolean | number | null | string)`
Returns: `number`

```
number("123")
```

Converts the argument to a number. Blocks and arrays are always considered invalid. The conversion rules are:

- `number`: returned as-is.
- `true` --> `1`, `false` --> `0`.
- `null` --> `0`.
- `string`: must strictly follow [integer](#numbers) or [float](#numbers) syntax. Any character that is not valid in a bconf number literal is invalid.

```bconf
$num = "123"
a = number(true)    // 1
b = number(false)   // 0
c = number(null)    // 0
d = number("3.14")  // 3.14
e = number(42)      // 42
f = number($num)    // 123

// INVALID: String does not follow number syntax.
invalid = number("12px")
```

---

#### int()

Syntax: `int(arg: boolean | number | null | string)`
Returns: `integer`

```
int("3.14")
```

Converts the argument to an integer. Blocks and arrays are always considered invalid. The conversion rules are:

- `integer`: returned as-is.
- `true` --> `1`, `false` --> `0`.
- `null` --> `0`.
- `string`: must strictly follow integer or float syntax. The string is first converted to its appropriate number type, then the rules below are applied.
- `float`: the value is truncated toward zero. Exponents are evaluated before truncation (e.g. `1.9e1` becomes `19.0`, which truncates to `19`).

```bconf
$pi = 3.14
a = int(true)     // 1
b = int(false)    // 0
c = int(null)     // 0
d = int(3.9)      // 3  — truncated, not rounded
e = int(-3.9)     // -3 — truncated toward zero
f = int("1.9e1")  // 19 — exponent evaluated first, then truncated
g = int(42)       // 42
h = int($pi)     // 3

// INVALID: String does not follow number syntax.
invalid = int("12px")
```

---

#### float()

Syntax: `float(arg: boolean | number | null | string)`
Returns: `float`

```
float(123)
```

Converts the argument to a float. Blocks and arrays are always considered invalid. The conversion rules are:

- `float`: returned as-is.
- `true` --> `1.0`, `false` --> `0.0`.
- `null` --> `0.0`.
- `string`: must strictly follow integer or float syntax. The string is first converted to its appropriate number type, then the rules below are applied.
- `integer`: converted to its exact floating-point representation (e.g. `5` becomes `5.0`).

```bconf
$one = 1
a = float(true)    // 1.0
b = float(false)   // 0.0
c = float(null)    // 0.0
d = float(5)       // 5.0
e = float("3.14")  // 3.14
f = float(2.71)    // 2.71
g = float($one)    // 1.0

// INVALID: String does not follow number syntax.
invalid = float("3.14abc")
```

---

#### bool()

Syntax: `bool(arg: boolean | number | null | string)`
Returns: `boolean`

```
bool("string")
```

Converts the argument to a boolean. Blocks and arrays are always considered invalid. The conversion rules are:

- `boolean`: returned as-is.
- `null` --> `false`.
- `string`: an empty string --> `false`; any non-empty string --> `true`.
- `number`: `0`, `0.0`, and `-0.0` --> `false`; any other number (including negative numbers) --> `true`.

```bconf
a = bool(true)     // true
b = bool(false)    // false
c = bool(null)     // false
d = bool("")       // false
e = bool("hello")  // true
f = bool(0)        // false
g = bool(0.0)      // false
h = bool(-1)       // true
i = bool(42)       // true
```

---

#### eq()

Syntax: `eq(a: boolean | number | null | string, b: boolean | number | null | string)`
Returns: `boolean`

```
eq(1, 2)
```

Returns `true` if the two arguments are equal, `false` otherwise.

If the two values are of different types, they are never equal and `false` is always returned — there is no implicit coercion. Blocks and arrays are not comparable; any comparison involving a block or array always returns `false`.

```bconf
a = eq("hello", "hello")   // true
b = eq(1, 1)               // true
c = eq(true, true)         // true
d = eq(null, null)         // true
e = eq("1", 1)             // false — different types
f = eq(1, 2)               // false

$cfg = { port = 8080 }
g = eq($cfg, $cfg)         // false — blocks are not comparable
```

---

#### lt()

Syntax: `lt(a: number, b: number)`
Returns: `boolean`

```
lt(1, 2)
```

Returns `true` if `a` is less than `b`. Both arguments must be numbers. Fails if either argument is not a number - there is no implicit coercion.

```bconf
a = lt(1, 2)    // true
b = lt(2, 2)    // false
c = lt(3, 2)    // false

// INVALID: Arguments must be numbers.
invalid = lt("1", 2)
```

---

#### gt()

Syntax: `gt(a: number, b: number)`
Returns: `boolean`

```
gt(1, 2)
```

Returns `true` if `a` is greater than `b`. Both arguments must be numbers. Fails if either argument is not a number - there is no implicit coercion.

```bconf
a = gt(3, 2)    // true
b = gt(2, 2)    // false
c = gt(1, 2)    // false

// INVALID: Arguments must be numbers.
invalid = gt(1, "2")
```

---

#### not()

Syntax: `not(arg: boolean)`
Returns: `boolean`

```
not(true)
```

Returns the logical negation of the argument. Fails if the argument is not a boolean.

```bconf
a = not(true)   // false
b = not(false)  // true

// INVALID: Argument must be a boolean.
invalid = not(1)
```

---

#### all()

Syntax: `all(...args: boolean)`
Returns: `boolean`

```
all(true, true, false)
```

Takes one or more boolean arguments and returns `true` if every argument is `true`. Fails if any argument is not a boolean.

```bconf
a = all(true, true, true)    // true
b = all(true, false, true)   // false

$flags = [true, true]
c = all(...$flags, true)     // true

// INVALID: All arguments must be booleans.
invalid = all(true, 1)
```

---

#### some()

Syntax: `some(...args: boolean)`
Returns: `boolean`

```
some(false, false, true)
```

Takes one or more boolean arguments and returns `true` if at least one argument is `true`. Fails if any argument is not a boolean.

```bconf
a = some(false, true, false)   // true
b = some(false, false, false)  // false

$flags = [false, false]
c = some(...$flags, true)      // true

// INVALID: All arguments must be booleans.
invalid = some(false, 0)
```

---

#### coalesce()

Syntax: `coalesce(...args: any)`
Returns: `any`

```
coalesce(null, 123, null)
```

Returns the first argument that does not resolve to `null`. Fails if all arguments resolve to `null`.

```bconf
a = coalesce(null, null, "fallback")  // "fallback"
b = coalesce("first", "second")       // "first"
c = coalesce(null, 42)                // 42

// INVALID: All arguments resolve to null.
invalid = coalesce(null, null)
```

---

#### contains()

Syntax: `contains(source: string, value: string) | contains(source: array, value: boolean | number | null | string)`
Returns: `boolean`

```
contains("hello world", "hello")
contains([123, "foo"], 123)
```

Returns `true` if `source` contains `value`.

If `source` is a string, `value` must also be a string. Returns `true` if `value` is a substring of `source`. Fails if `value` is not a string.

If `source` is an array, `value` may be a boolean, number, null or string. Returns `true` if any element in the array is equal to `value`, using the same equality rules as [eq()](#eq). Because blocks and arrays are not comparable, passing a block or array as `value` when `source` is an array always returns `false`.

Fails if `source` is not a string or array.

```bconf
a = contains("hello world", "world")   // true
b = contains("hello world", "xyz")     // false

c = contains([1, 2, 3], 2)             // true
d = contains([1, 2, 3], 4)             // false
e = contains(["a", "b"], "a")          // true

$cfg = { port = 8080 }
f = contains([1, 2], $cfg)             // false — blocks are not comparable

// INVALID: source is a string but value is not.
invalid1 = contains("hello", 1)

// INVALID: source must be a string or array.
invalid2 = contains(42, "hello")
```

---

#### len()

Syntax: `len(source: string | array)`
Returns: `integer`

```
len("hello world")
```

Returns the number of characters in a string or the number of elements in an array. Fails if the argument is not a string or array.

```bconf
a = len("hello")           // 5
b = len("")                // 0
c = len([1, 2, 3])         // 3
d = len([])                // 0

// INVALID: Argument must be a string or array.
invalid = len(42)
```

---

#### keys()

Syntax: `keys(arg: block)`
Returns: `string []`

```
keys({ host = "localhost"; port = 8080 })
```

Returns an array of strings containing the top-level keys of the block. Only the immediate keys of the block are included — keys from nested blocks are not traversed. Returns an empty array if the block has no keys. Fails if the argument is not a block.

```bconf
$cfg = { host = "localhost"; port = 8080; tls { enabled = true } }

k = keys($cfg)   // ["host", "port", "tls"]

// INVALID: Argument must be a block.
invalid = keys([1, 2, 3])
```

---

#### values()

Syntax: `values(arg: block)`
Returns: `any []`

```
values({ host = "localhost"; port = 8080 })
```

Returns an array of the values from the block, in the order they appear. Only the immediate values are included - nested blocks are included but not traversed. All returned values are deeply copied. Returns an empty array if the block has no keys. Fails if the argument is not a block.

```bconf
$cfg = { host = "localhost"; port = 8080; tls = { enabled } }

v = values($cfg)   // ["localhost", 8080, { enabled = true }]

// INVALID: Argument must be a block.
invalid = values([1, 2, 3])
```

---

#### entries()

Syntax: `entries(arg: block)`
Returns: `[key: string, value: any] []`

```
entries({ host = "localhost"; port = 8080 })
```

Returns a two-dimensional array where each element is a two-element array of `[key, value]` for each top-level entry in the block, in the order they appear. Only immediate entries are included — nested blocks are included but not traversed. Returns an empty array if the block has no keys. Fails if the argument is not a block.

```bconf
$cfg = { host = "localhost"; port = 8080; tls { enabled } }

e = entries($cfg)
// [["host", "localhost"], ["port", 8080], ["tls", { enabled = true }]]

// INVALID: Argument must be a block.
invalid = entries([1, 2, 3])
```

---

#### trim()

Syntax: `entries(arg: string, side?: "start" | "end" | "both")`
Returns: `string`

```
trim("    trimmed       ")
```

Removes leading and/or trailing whitespace from the string. The optional second argument specifies which side to trim: `"start"` trims only the start, `"end"` trims only the end, and `"both"` trims both sides. If the second argument is omitted, `"both"` is used. Fails if the first argument is not a string, or if the second argument is present but does not exactly match one of the three allowed strings.

```bconf
a = trim("  hello  ")           // "hello"
b = trim("  hello  ", "start")   // "hello  "
c = trim("  hello  ", "end")  // "  hello"
d = trim("  hello  ", "both")   // "hello"

// INVALID: Second argument must be "left", "right", or "both".
invalid1 = trim("  hello  ", "all")

// INVALID: First argument must be a string.
invalid2 = trim(42)
```

---

#### lower()

Syntax: `lower(arg: string)`
Returns: `string`

```
lower("YELLING")
```

Returns the string with all characters converted to lowercase. Fails if the argument is not a string.

```bconf
a = lower("Hello World")  // "hello world"
b = lower("ABC")          // "abc"
```

---

#### upper()

Syntax: `upper(arg: string)`
Returns: `string`

```
upper("whispering")
```

Returns the string with all characters converted to uppercase. Fails if the argument is not a string.

```bconf
a = upper("hello world")  // "HELLO WORLD"
b = upper("abc")          // "ABC"
```

---

#### min()

Syntax: `min(...args: number)`
Returns: `number`

```
min(10, 1, 3, 4)
```

Returns the lowest value among the arguments. At least one argument is required. Fails if any argument is not a number.

```bconf
a = min(3, 1, 2)      // 1
b = min(-5, 0, 5)     // -5
c = min(3.14, 2.71)   // 2.71

// INVALID: All arguments must be numbers.
invalid = min("1", 2)
```

---

#### max()

Syntax: `max(...args: number)`
Returns: `number`

```
max(15, 32, 2, 12)
```

Returns the highest value among the arguments. At least one argument is required. Fails if any argument is not a number.

```bconf
a = max(3, 1, 2)      // 3
b = max(-5, 0, 5)     // 5
c = max(3.14, 2.71)   // 3.14

// INVALID: All arguments must be numbers.
invalid = max(1, "2")
```

---

#### clamp()

Syntax: `clamp(value: number, min: number, max: number)`
Returns: `number`

```
clamp(10, 1, 5)
```

Constrains `value` to be no less than `min` and no greater than `max`. All three arguments are required. If `value` is below `min`, `min` is returned. If `value` is above `max`, `max` is returned. Otherwise `value` is returned unchanged. Fails if any argument is not a number.

```bconf
a = clamp(5, 1, 10)    // 5 — within range
b = clamp(0, 1, 10)    // 1 — below min
c = clamp(15, 1, 10)   // 10 — above max
d = clamp(1, 1, 10)    // 1 — equal to min
e = clamp(10, 1, 10)   // 10 — equal to max

// INVALID: All arguments must be numbers.
invalid = clamp("5", 1, 10)
```

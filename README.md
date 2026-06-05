# bconf

<img align="right" src="logos/bconf-100.png" alt="bconf logo">

Better configuration files

> This repository contains the working documents of the bconf specification.
> You can find the released versions at https://bconf-lang.org

## Design Principles

bconf is designed with a single objective in mind: make it easy to compose large or complex configurations. Syntax and concepts should be familiar, but flexible enough that its easy for applications to enhance values directly without the need for magic strings. Configuration values should remain predictable with no hidden default values, and a standard library of sorts should be expected for implementations so developers have a baseline for portability.

## Example

```bconf
// This is a bconf file

@extends "./base.bconf"
@import "./secrets.bconf" { $db_user; $db_pass = $database_password }

$$ENV = env("APP_ENV")
$app_name = "An awesome app"
$common_domains = ["https://app.example.com", "https://admin.example.com"]

app {
    name = $app_name
    environment = $$ENV
    features {
        auth
    }
}

server.http.host = "0.0.0.0"
server.http.port = int(env("PORT"))

server.tls {
    enabled
    cert_file = "/etc/ssl/certs/app.crt"
    key_file = "/etc/ssl/private/app.key"
}

database.primary {
    host = "primary.db.internal"
    user = $db_user
    pass = $database_password
    pool_size = int(env("DB_POOL_SIZE"))
}

database.replicas = [
    { host = "replica1.db.internal"; user = $db_user },
    { host = "replica2.db.internal"; user = $db_user }
]

database.replicas[0].read_only = true
database.replicas[1].read_only = false

plugins << {
    name = "authentication"
    config {
        jwt_secret = env("JWT_SECRET")
        token_expiry = (
            | eq($$ENV, "prod") => "1hr"
            | "6hr"
        )
    }
}

plugins << {
    name = "cors"
    config {
        allowed_origins = [...$common_domains, "staging.example.com"]
    }
}

api_docs_header = """
    Welcome to the API for ${$app_name}.
    Host: ${ref(server.http.host)}:${ref(server.http.port)}
    Environment: ${$$ENV}
"""

@export {
    $$ENV,
    $name = $app_name
}
```

## Compared to Other Languages

bconf sits somewhere between a data-serialization format and a programming language. At its core, it is a data-serialization format since only a hash-map is produced from a document. Unlike other formats though, there is an emphasis on features that are more programming language like, such as variables, directives and modifiers. However, basically everything else is missing from what you'd expect a full-featured programming language to have, such as arithmetic and equality operators.

Unlike other data-serialization formats, bconf does not require a strict hierarchy of values to define deeply nested values. Values can be assigned using nested keys and array indexes. Keys can have values conditionally assigned if no value is present at the key, and arrays can also have values pushed to it with an append operator.

Features often used in tandem when creating configuration with data-serialization formats, such as extending files and referencing other values, are typically implementation specific and non-portable. However, features like this are standardized in bconf, ensuring the same set of features can be used regardless of parser or language.

As bconf is explicitly designed for configuration files, it is not intended to support serializing arbitrary data structures. The root of any valid bconf document is always a hash-map which excludes some data from being serialized.

## Contributions

Any ideas, fixes, documentation, etc., are welcome! If you have any ideas, feedback or suggestions, open an issue.

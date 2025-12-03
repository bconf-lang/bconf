# bconf

<img align="right" src="logos/bconf-100.png" alt="bconf logo">

For better configuration files

> This repository contains the latest draft version of the bconf specification.
> You can find the released versions at https://bconf-lang.org

## Design Principles

bconf is designed with a the following key principles in mind: configuration files should be human readable, easy to maintain, flexible and predictable. The syntax and concepts should be familiar and easy to learn, leveraging constructs found in other data-serialization and programming languages.

Dynamic elements like variables, tags and extending files should be expressed with existing syntax. All syntax should be deserializable into native data structures that make sense _without_ introducing "invisible" keys/values. This ensures all valid bconf files remain 100% statically parsable with predictable data structures.

## Example

```bconf
// This is a bconf file

extends "./base.bconf"
import from "./secrets.bconf" { $db_user; $db_pass as $database_password }

$app_name = "An awesome app"
$env = env("APP_ENV")

app {
    name = $app_name
    environment = $env
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
        token_expiry = "1h"
    }
}

plugins << {
    name = "cors"
    config {
        allowed_origins = ["https://app.example.com", "https://admin.example.com"]
    }
}

api_docs_header = """
    Welcome to the API for \${$app_name}.
    Host: \${ref(server.http.host)}:\${ref(server.http.port)}
    Environment: \${$env}
"""

export vars {
    $env
    $app_name as $name
}
```

## Compared to Other Languages

bconf is a data-serialization format designed to be easy to enhance. Compared to other formats, like JSON, YAML and TOML, bconf is minimal and easy to write. Where it differs is an emphasis on scalability for writing large files and expressive ways to write dynamic features. Data types like tags and statements make methods and commands easy to write without it needing to be expressed awkwardly through other data structures. This syntax remains 100% statically deserializable into native data structures.

Unlike other data-serialization formats, bconf does not require a strict hierarchy of values to define deeply nested values. Values can be assigned using deeply nested keys and array indexes. Arrays can also have values pushed to it with an append operator.

Features often used in tandem when creating configuration with data-serialization formats, such as extending files and referencing other values, are typically implementation specific and non-portable. However, features like this are standardized in bconf, ensuring the same set of features can be used regardless of parser or language.

As bconf is explicitly designed for configuration files, it is not intended to support serializing arbitrary data structures. The root of any valid bconf document is always a hash-map which excludes some data from being serialized.

## Pitching in

Ideas, pull requests, documentation, bug fixes, and all other contributions are welcome!

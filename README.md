# bconf

<img align="right" src="logos/bconf-100.png" alt="bconf logo">

For better configuration files

> This repository contains the latest draft version of the bconf specification.
> You can find the released versions at https://bconf-lang.org

## Design Principles

bconf is designed with a few key principles in mind: configuration files should be human readable, easy to maintain, flexible and predictable. The syntax and concepts should be familiar and easy to learn, leveraging constructs found in other data-serialization and programming languages.

Dynamic elements like variables, schemas and extending files should be expressed with existing syntax. All syntax should be deserializable into native data structures that make sense _without_ introducing "invisible" keys/values. This ensures all valid bconf files remain 100% statically parsable with predictable data structures.

## Example

```bconf
// This is an example bconf document mimicking a config file for a CI/CD pipeline

import from "../common.bconf" { $docker_registry, $alert_channels }

name = "Build & Deploy"

// Variables are keys that start with $
$project_name = "phoenix"
$CI_COMMIT_SHA = env("CI_COMMIT_SHA")

job_stages = ["build", "test", "deploy"]

default_settings {
	// Custom tagged values can be declared to be handled in the application layer
	timeout = duration("10m")
	max_retries = 0

	// Arrays can be set/accessed with an index
	runs_on[0] = "linux"
}

// Nested keys are allowed
jobs.build {
	name = "Build image"

	// Bare keys in objects is shorthand for the value being true
	cancel_in_progress

	// This variable is scoped to the `jobs.build` block
	$image_name = "${$docker_registry}/${$project_name}:${$CI_COMMIT_SHA}"

	steps {
		// Nodes can be set inside objects
		run "echo 'Building Docker image...'"
		run "docker build -t ${$image_name} ."
	}
}

jobs.test {
	name = "Run tests"
	max_retries = 3

	// Arrays can have values pushed using `<<`
	needs << "build"
	needs << "test"

	steps {
		run "echo 'Running unit tests...'"
		run "npm test"
	}
}

jobs.deploy {
	$deploy_target = "production"

	// Custom nodes can be defined
	hooks onfailure {
		channels = $alert_channels
		// Multiline strings are defined with triple quotes
		message = """
			❌ Could not deploy to ${$deploy_target}
			Image: ${ref(jobs.build.$image_name)}

			This may block future deployments if unaddressed - check that everything is working correctly
		"""
	}

	// Use a dynamic key to define environment-specific settings.
	// This block will be evaluated as a 'production' key.
	[$deploy_target] {
		replicas = int("3")

		steps {
			// Variables are just special keys, so their values can be referenced as regular data as well
			run "echo 'Deploying image ${ref(jobs.build.$image_name)} to ${$deploy_target}...'"
			run "kubectl apply -f deployment.yml"
			run "echo 'Deployment successful.'"
		}
	}
}
```

## Compared to Other Languages

bconf is a data-serialization format designed to be easy to enhance. Compared to other formats, like JSON, YAML and TOML, bconf is minimal and easy to write. Where it differs is an emphasis on scalability for writing large files and expressive ways to write dynamic features. Features like tagged values and nodes make methods and statements easy to write without it needing to be expressed awkwardly through other data structures. This syntax remains 100% statically deserializable into native data structures.

Unlike other data-serialization formats, bconf does not require a strict hierarchy of values to define deeply nested values. Values can be assigned using deeply nested keys and array indexes. Arrays can also have values pushed to it.

Features often used in tandem when creating configuration with data-serialization formats, such as extending files and schemas, are typically implementation specific and non-portable. However, features like this are standardized in bconf, ensuring the same set of features can be used regardless of parser or language.

As bconf is explicitly designed for configuration files, it is not intended to support serializing arbitrary data structures. The root of any valid bconf document is always a hash-map which excludes some data from being serialized.

## Pitching in

Ideas, pull requests, documentation, bug fixes, and all other contributions are welcome!

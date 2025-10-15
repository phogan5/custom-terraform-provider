## Testing steps
When testing updates to the provider for the hashicups app lab, follow these steps before running a terraform operation (plan, apply, destroy, etc).

1) Set GOBIN environment variable
`export GOBIN="/home/phogan/go/bin`

2) Create ~.terraformrc with the dev_overrides block
```hcl
provider_installation {

  dev_overrides {
      "hashicorp.com/edu/hashicups" = "<PATH>"
  }

  # For all other providers, install them directly from their origin provider
  # registries as normal. If you omit this, Terraform will _only_ use
  # the dev_overrides block, and so no other providers will be available.
  direct {}
}
```

3) Start the docker app using docker-compose
`cd docker_compose && docker-compose up`

4) Create an API user and retrieve an authentication token
```shell
response = $(curl -X POST localhost:19090/signup -d '{"username":"education", "password":"test123"}')

export HASHICUPS_TOKEN=$(echo $token | jq .token)
```

5) Save changes to the provider
`go install .`
 
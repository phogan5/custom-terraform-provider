# Provider client configuration
The provider client config defines how the provider should interact with your API via provider config or environment variables.

1) Define the provider schema
Allows the provider to accept client authentication and host information.

2) Define the provider data model
Defines a model that represents the provider scheme as a go `type` so the data is accessible for other go code.

3) Define the provider `configure` method
Defines a method that reads terraform configuration through the data model or environment variables and creates a client to be used by data sources and resources. Raises errors if data is wrong or missing.

4) Testing changes

## Define provider schema
The provider schema is implemented as a method implementation of the provider struct. It uses the schema package to define attributes as a map of `strings` and `schema.StringAttribute`. In this example we are using three configuration attributes: host, username, and password.

```go
// Schema defines the provider-level schema for configuration data.
func (p *hashicupsProvider) Schema(_ context.Context, _ provider.SchemaRequest, resp *provider.SchemaResponse) {
    resp.Schema = schema.Schema{
        Attributes: map[string]schema.Attribute{
            "host": schema.StringAttribute{
                Optional: true,
            },
            "username": schema.StringAttribute{
                Optional: true,
            },
            "password": schema.StringAttribute{
                Optional:  true,
                Sensitive: true,
            },
        },
    }
}
```

## Implement provider data model
A data model is created that maps the passed values to string values that can be used throughout the rest of the application using the `tfsdk` struct fields.

```go
// hashicupsProviderModel maps provider schema data to a Go type.
type hashicupsProviderModel struct {
    Host     types.String `tfsdk:"host"`
    Username types.String `tfsdk:"username"`
    Password types.String `tfsdk:"password"`
}
```

## Implement client configuration
The provider enabled client configuration using the `Configure` method of the provider struct. This method is used to retrieve the required data from the provider configuration or environment variables and pass it to the provider itself. It also defines several error messages for missing or incorrect configurations.

## Testing changes
Once the provider configuration is finished, you can test the changes by creating a dummy datasource and running `terraform plan`. By default, you will recieve errors (defined in the `Configure` method) stating you do not have the required configuration data.

```txt
$ terraform plan
##...
╷
│ Error: Missing HashiCups API Host
│ 
│   with provider["hashicorp.com/edu/hashicups"],
│   on main.tf line 9, in provider "hashicups":
│    9: provider "hashicups" {}
│ 
│ The provider cannot create the HashiCups API client as there is a missing or
│ empty value for the HashiCups API host. Set the host value in the
│ configuration or use the HASHICUPS_HOST environment variable. If either is
│ already set, ensure the value is not empty.
╵
╷
│ Error: Missing HashiCups API Username
│ 
│   with provider["hashicorp.com/edu/hashicups"],
│   on main.tf line 9, in provider "hashicups":
│    9: provider "hashicups" {}
│ 
│ The provider cannot create the HashiCups API client as there is a missing or
│ empty value for the HashiCups API username. Set the username value in the
│ configuration or use the HASHICUPS_USERNAME environment variable. If either
│ is already set, ensure the value is not empty.
╵
╷
│ Error: Missing HashiCups API Password
│ 
│   with provider["hashicorp.com/edu/hashicups"],
│   on main.tf line 9, in provider "hashicups":
│    9: provider "hashicups" {}
│ 
│ The provider cannot create the HashiCups API client as there is a missing or
│ empty value for the HashiCups API password. Set the password value in the
│ configuration or use the HASHICUPS_PASSWORD environment variable. If either
│ is already set, ensure the value is not empty.
╵
```

To fix this, export your environment variables and try again. This shows that the provider configuration is being read correctly.

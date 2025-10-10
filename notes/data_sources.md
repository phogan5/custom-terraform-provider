# Implementing Data Sources
Data sources are a type of "resource" that can be defined in terraform configuration that retrieve data from the provider. In AWS, this can be retrieving an existing instance id, or iam role. In other API applications, this can be retrieving arbritrary data and using it as an output or to inform other resources.

To do this, you must:

1) Define the data source type (struct)
This prepares the data source object to be added to the provider

2) Add data source to the provider
This enables the data source

3) Implement the client in the data source
This allows the already-configured provider client to be accessible from the data source.

4) Define the data source schema
This defines the attributes in the data source to be set in terraform state

5) Define the data source data model
This defines the schema as a go type so that it is accessible from other go code

6) Define the data source read method logic
This expands on the `read` method to implement reading the provider API and retrieving information for that data source.

7) Verify data source behavior

## Define the data source struct
The new data source should be created as a struct in your <data_source>.go file. Its methods are defined below as `Metadata`, `Schema`, and `Read`

```go
// coffeesDataSource is the data source implementation.
type coffeesDataSource struct{}

// Metadata returns the data source type name.
func (d *coffeesDataSource) Metadata(_ context.Context, req datasource.MetadataRequest, resp *datasource.MetadataResponse) {
  resp.TypeName = req.ProviderTypeName + "_coffees"
}

// Schema defines the schema for the data source.

// Read refreshes the Terraform state with the latest data.
```

## Add data source to provider
Once the data source is defined, it can be added as a data source to the provider via the `DataSources` provider method

```go
// DataSources defines the data sources implemented in the provider.
func (p *hashicupsProvider) DataSources(_ context.Context) []func() datasource.DataSource {
  return []func() datasource.DataSource {
    NewCoffeesDataSource,
  }
}
```

## Implement the client in the data source
You can enable access to the provider client in the data source by adding the `Configure` method to the data source struct and using the `datasource.ConfigureRequest` and `datasource.ConfigureResponse` arguments

## Implement the data source schema
The schema is the collection of attributes that apply to the data source. These attributes define the resource and can be used to filter or query data, if that functionality is implemented.

The data source schema is defined in the `Schema` method.

```go
// Schema defines the schema for the data source.
func (d *coffeesDataSource) Schema(_ context.Context, _ datasource.SchemaRequest, resp *datasource.SchemaResponse) {
  resp.Schema = schema.Schema{
    Attributes: map[string]schema.Attribute{
      "coffees": schema.ListNestedAttribute{
        Computed: true,
        NestedObject: schema.NestedAttributeObject{
          Attributes: map[string]schema.Attribute{
            "id": schema.Int64Attribute{
              Computed: true,
            },
            "name": schema.StringAttribute{
              Computed: true,
            },
            "teaser": schema.StringAttribute{
              Computed: true,
            },
          }
        }
      }
    }
  }
}
```

## Implement data source data models
Data models map the schema data to tfsdk data types

```go
// coffeesDataSourceModel maps the data source schema data.
type coffeesDataSourceModel struct {
    Coffees []coffeesModel `tfsdk:"coffees"`
}

// coffeesModel maps coffees schema data.
type coffeesModel struct {
    ID          types.Int64               `tfsdk:"id"`
    Name        types.String              `tfsdk:"name"`
    Teaser      types.String              `tfsdk:"teaser"`
}
```

## Implement read functionality
The data source `Read` method defines logic for reading the data from the provider api based on the data source schema. The method uses the configured api client to connect to the api and read the resource.

## Verify data source 
Once finished, you should be able to add a `data{}` block and `output{}` block to `main.tf`. When you run `terraform plan`, the provider will read the API and return the requested data as an output in the plan

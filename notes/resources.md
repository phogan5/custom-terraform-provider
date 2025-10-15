# Implementing resources
Resources are the foundation of your terraform provider, and correspond with REST API calls in the backend API to build resources using terraform. Resources are created much the same as [data sources](data_sources.md), with many of the same struct methods.

## Provider implementation
Once configured, the resource itself needs to be added to the `provider.Resources` method in order to be recognized as a resource by the provider.

```go
// Resources defines the resources implemented in the provider.
func (p *hashicupsProvider) Resources(_ context.Context) []func() resource.Resource {
	return []func() resource.Resource{
		NewOrderResource,
	}
}
```

## Resource struct
The resource itself is represented as a struct with only one attribute: a `*hashicups.Client` pointer that enables the resource to access the API with the authentication methods supplied in the provider itself.

The struct should be supported by additional vars and a function that provides type checking and makes implementation easier

```go
// Resource implementation
type orderResource struct {
    client *hashicups.Client
}

// Ensure the implementation satisfies the expected interfaces.
var (
	_ resource.Resource              = &orderResource{}
	_ resource.ResourceWithConfigure = &orderResource{}
)

// NewOrderResource is a helper function to simplify the provider implementation.
func NewOrderResource() resource.Resource {
	return &orderResource{}
}
```

## Resource data model
The data model is, as a whole, responsible for laying out the various attributes of the resources and defining their data types and mappings. Like JSON implementations in Go, this data representation consists of multiple structs nested in one another. There should be a top-level `orderResourceModel` struct that represents terraform resources in general using the ID, Items, and LastUpdated fields. Other structs represent the data schema of the partiular resource.

In the below example, the `orderResourceModel` struct defines an arbritrary terraform resource, `orderItemModel` defines an order, and `orderItemCoffeeModel` defines a particular coffee. Each of these structs role up to the top level `orderResourceModel`.

```go
// orderResourceModel maps the resource schema data.
type orderResourceModel struct {
	ID          types.String     `tfsdk:"id"`
	Items       []orderItemModel `tfsdk:"items"`
	LastUpdated types.String     `tfsdk:"last_updated"`
}

// orderItemModel maps order item data.
type orderItemModel struct {
	Coffee   orderItemCoffeeModel `tfsdk:"coffee"`
	Quantity types.Int64          `tfsdk:"quantity"`
}

// orderItemCoffeeModel maps cofee order item data.
type orderItemCoffeeModel struct {
	ID          types.Int64   `tfsdk:"id"`
	Name        types.String  `tfsdk:"name"`
	Teaser      types.String  `tfsdk:"teaser"`
	Description types.String  `tfsdk:"description"`
	Price       types.Float64 `tfsdk:"price"`
	Image       types.String  `tfsdk:"image"`
}
```

When planning these resources in a `.tf` file, they need these attributes defined. For example, this terraform configuration file can be used to create the above resource. See [resource schema](./resources.md#schema)

## Resource methods
Various methods are used to perform different resource functions, such as resource creation, deletion, updating, schema definition, and client configuration

### Metadata
Metadata returns metadata about the resource implementation. It can be used to return the name of the provider or resource, or other information.

### Schema
Schema defines the resource schema. While the resource data models define what data types can be used for different attributes, the resource schema defines what attributes must be passed with the resource, their data types, whether they are computed, required, nested, and other terraform-specific details. This information is used to store resource information in the terraform state file.

The following example defines the resource schema and its attributes:

```go
func (r *orderResource) Schema(_ context.Context, _ resource.SchemaRequest, resp *resource.SchemaResponse) {
	resp.Schema = schema.Schema{
		Attributes: map[string]schema.Attribute{
			"id": schema.StringAttribute{
				Computed: true,
				PlanModifiers: []planmodifier.String{
					stringplanmodifier.UseStateForUnknown(),
				},
			},
			"last_updated": schema.StringAttribute{
				Computed: true,
			},
			"items": schema.ListNestedAttribute{
				Required: true,
				NestedObject: schema.NestedAttributeObject{
					Attributes: map[string]schema.Attribute{
						"quantity": schema.Int64Attribute{
							Required: true,
						},
						"coffee": schema.SingleNestedAttribute{
							Required: true,
							Attributes: map[string]schema.Attribute{
								"id": schema.Int64Attribute{
									Required: true,
								},
								"name": schema.StringAttribute{
									Computed: true,
								},
                            },
                        },
                    },
                },
            },
        },
    }
}
```

Using the above resource schema, the resource can be planned with a resource configuration file (`.tf` file) like this (note the difference between required attributes and 'known after configuration' attributes):

```tf
provider "hashicups" {
  username = "education"
  password = "test123"
  host     = "http://localhost:19090"
}

resource "hashicups_order" "edu" {
  items = [{
    coffee = {
      id = 3
    }
    quantity = 2
    }, {
    coffee = {
      id = 2
    }
    quantity = 4
    }
  ]
}
```

### Configure
Configure is used to add the provider-configured client to the resource definition. This sets the `*resource.client` variable to be used for other resource methods. Without the client configured, the resource will be unable to interact with the backend API to create, update, delete, or read anything.

### Create
Create creates the resource and updates the terraform state with that new resource. To create resources, the method retrieves the plan, and iterates through the plan, calling a predefined API endpoint with the required attributes/parameters:

```go
// Retrieve values from plan
var plan orderResourceModel
diags := req.Plan.Get(ctx, &plan) // Terrafrom plan lives in ctx
resp.Diagnostics.Append(diags...)
if resp.Diagnostics.HasError() {
    return
}

// Generate API request body from plan
var items []hashicups.OrderItem // OrderItem api
for _, item := range plan.Items {
    items = append(items, hashicups.OrderItem{
        Coffee: hashicups.Coffee{
            ID: int(item.Coffee.ID.ValueInt64()),
        },
        Quantity: int(item.Quantity.ValueInt64()),
    })
}

// Create new order
order, err := r.client.CreateOrder(items) // CreateOrder api
if err != nil {
    resp.Diagnostics.AddError(
        "Error creating order",
        "Could not create order, unexpected error: "+err.Error(),
    )
    return
	}
```

Once the resource is created, the resource attributes are mapped to the [resource data schema](./resources.md#schema) using the [resource data models](./resources.md#resource-data-model) and used to update the plan file with the newly created resource.

```go
// Map response body to schema and populate Computed attribute values
plan.ID = types.StringValue(strconv.Itoa(order.ID))
for orderItemIndex, orderItem := range order.Items {
    plan.Items[orderItemIndex] = orderItemModel{
        Coffee: orderItemCoffeeModel{
            ID:          types.Int64Value(int64(orderItem.Coffee.ID)),
            Name:        types.StringValue(orderItem.Coffee.Name),
            Teaser:      types.StringValue(orderItem.Coffee.Teaser),
            Description: types.StringValue(orderItem.Coffee.Description),
            Price:       types.Float64Value(orderItem.Coffee.Price),
            Image:       types.StringValue(orderItem.Coffee.Image),
        },
        Quantity: types.Int64Value(int64(orderItem.Quantity)),
    }
}
plan.LastUpdated = types.StringValue(time.Now().Format(time.RFC850))

// Set state to fully populated data
diags = resp.State.Set(ctx, plan)
resp.Diagnostics.Append(diags...)
if resp.Diagnostics.HasError() {
    return
}
```

### Read
Read calls a seperate API to get an existing resource and return information associated with that resource. That information can then be used to update the terraform state with the most up-to-date information.

### Update
Update works similarly to the [create](./resources.md#create) and [read](./resources.md#read) methods to update a resource using a specific API endpoint and update the terraform state with the new configuration.

### Delete
Delete retrieves the current state and calls an API to delete that resource from the terraform state. This can be much simpler than other CRUD methods because there is no data mapping that has to occur to update state.

```go
// Delete deletes the resource and removes the terraform state on success.
func (r *orderResource) Delete(ctx context.Context, req resource.DeleteRequest, resp *resource.DeleteResponse) {
	// Retrieve values from state
	var state orderResourceModel
	diags := req.State.Get(ctx, &state)
	resp.Diagnostics.Append(diags...)
	if resp.Diagnostics.HasError() {
		return
	}

	// Delete existing order resource
	err := r.client.DeleteOrder(state.ID.ValueString())
	if err != nil {
		resp.Diagnostics.AddError(
			"Error deleting Hashicups order",
			"Could not delete order, unexpected error: "+err.Error(),
		)
		return
	}
}
```

# Logging

## Adding logging to the provider
Log messages can be added using the [tflog]("github.com/hashicorp/terraform-plugin-log/tflog") library. 

## Viewing logs
Logs can be viewed by setting the `TF_LOG=TRACE | INFO | WARNING` environment variable (log level) and, optionally, setting the `TF_LOG_PATH` environment variable (output log file location)

## Examples

### Basic log levels
Log levels work similarly to logging in other languages. The `ctx` is required because the logs are saved to the context
```go
tflog.Info(ctx, "Configuring Hashicups client")
tflog.Debug(ctx, "Debug message")
```

### Structured log fields
Key-value pairs can be added to the context to be referenced later by other log messages
```go
ctx = tflog.SetField(ctx, "hashicups_host", host)
ctx = tflog.SetField(ctx, "hashicups_username", username)
ctx = tflog.SetField(ctx, "hashicups_password", password)
```

### Log filtering
Log filtering/masking can be applied to sensitive values. These will be masked in the output logs
```go
ctx = tflog.SetField(ctx, "hashicups_password", password)
ctx = tflog.MaskFieldValuesWithFieldKeys(ctx, "hashicups_password")
```


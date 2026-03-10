# IConfiguration — How It Works and Advanced Usage

## Why IConfiguration "just works"

> **Q:** How can the `IConfiguration` be injected without even having to call something like `ConfigureAppConfiguration` with the `CreateDefaultBuilder`? Is it something magic with the standard name `appsettings.json`? How could I add another JSON configuration file (how to configure it, how to inject its `IConfiguration`, and so on)?

`Host.CreateDefaultBuilder()` (or `Host.CreateApplicationBuilder()`) automatically calls `ConfigureAppConfiguration` under the hood with these sources, in order:

1. `appsettings.json` (optional, reloadOnChange: true)
2. `appsettings.{Environment}.json` (optional, reloadOnChange: true)
3. Environment variables
4. Command-line args
5. User secrets (in Development environment only)

So yes, `appsettings.json` is "magic" — but only because `CreateDefaultBuilder` registers it for you. If you used `new HostBuilder()` directly, you'd get nothing automatically.

## How to add another JSON config file

In the host builder setup, chain `ConfigureAppConfiguration`:

```csharp
var builder = Host.CreateDefaultBuilder(args)
    .ConfigureAppConfiguration((context, config) =>
    {
        // The defaults (appsettings.json, env vars, etc.) are already registered.
        // Just add your extra file:
        config.AddJsonFile("mysettings.json", optional: true, reloadOnChange: true);
    });
```

Later sources override earlier ones for the same keys, so your custom file will win over `appsettings.json` for any duplicate keys.

## How to inject / consume it

**`IConfiguration`** — You don't need to register it; the host does that. Just request it via constructor injection:

```csharp
public class MyService(IConfiguration configuration)
{
    var value = configuration["MySection:MyKey"];
}
```

**Strongly-typed options (preferred)** — Bind a section to a POCO class:

```csharp
// In your service registration:
services.Configure<MyOptions>(configuration.GetSection("MySection"));

// POCO:
public class MyOptions
{
    public string MyKey { get; set; }
}

// Then inject:
public class MyService(IOptions<MyOptions> options)
{
    var value = options.Value.MyKey;
}
```

### Key points

- There's **one** `IConfiguration` instance — it's a merged view of all sources. There's no separate `IConfiguration` per file.
- Source order matters: last writer wins for duplicate keys.
- `IOptionsSnapshot<T>` gives you reloaded values per-request (in scoped scenarios); `IOptionsMonitor<T>` gives you a change callback.

---

## IOptions\<T\> Showcase — Dates, Lists, Deep Nesting, and More

> **Q:** With builder configuration and `IOptions<T>`, can you showcase how to work with dates, list of items, and other stuff from settings? Also show how to use sections with many levels.

### appsettings.json

```json
{
  "App": {
    "LaunchDate": "2026-03-15T09:00:00Z",
    "MaintenanceWindow": {
      "Start": "02:00:00",
      "End": "04:00:00"
    },
    "Tags": [ "production", "v2", "stable" ],
    "AllowedOrigins": [ "https://example.com", "https://api.example.com" ],
    "Limits": {
      "MaxRetries": 5,
      "TimeoutSeconds": 30.5,
      "EnableCircuitBreaker": true
    },
    "Database": {
      "Primary": {
        "Connection": {
          "Host": "db.example.com",
          "Port": 5432,
          "Credentials": {
            "Username": "admin",
            "UseSsl": true
          }
        }
      }
    },
    "FeatureFlags": {
      "DarkMode": true,
      "ExperimentalApi": false,
      "BetaUsers": [ "alice", "bob" ]
    },
    "Schedule": {
      "Jobs": [
        {
          "Name": "Cleanup",
          "CronExpression": "0 0 * * *",
          "Enabled": true,
          "RetryPolicy": {
            "MaxAttempts": 3,
            "DelaySeconds": 60
          }
        },
        {
          "Name": "Report",
          "CronExpression": "0 8 * * 1",
          "Enabled": false,
          "RetryPolicy": {
            "MaxAttempts": 1,
            "DelaySeconds": 120
          }
        }
      ]
    },
    "Metadata": {
      "Version": "2.1.0",
      "BuildDate": "2026-02-19",
      "Environment": "staging"
    }
  }
}
```

### C# Options classes

```csharp
public class AppOptions
{
    public const string SectionName = "App";

    public DateTime LaunchDate { get; set; }
    public TimeWindow MaintenanceWindow { get; set; } = new();
    public List<string> Tags { get; set; } = [];
    public string[] AllowedOrigins { get; set; } = [];
    public LimitOptions Limits { get; set; } = new();
    public DatabaseOptions Database { get; set; } = new();
    public FeatureFlagOptions FeatureFlags { get; set; } = new();
    public ScheduleOptions Schedule { get; set; } = new();
    public Dictionary<string, string> Metadata { get; set; } = new();
}

public class TimeWindow
{
    public TimeSpan Start { get; set; }    // "02:00:00" binds to TimeSpan
    public TimeSpan End { get; set; }
}

public class LimitOptions
{
    public int MaxRetries { get; set; }
    public double TimeoutSeconds { get; set; }
    public bool EnableCircuitBreaker { get; set; }
}

// Deep nesting: App -> Database -> Primary -> Connection -> Credentials
public class DatabaseOptions
{
    public DatabaseInstanceOptions Primary { get; set; } = new();
}

public class DatabaseInstanceOptions
{
    public ConnectionOptions Connection { get; set; } = new();
}

public class ConnectionOptions
{
    public string Host { get; set; } = "";
    public int Port { get; set; }
    public CredentialOptions Credentials { get; set; } = new();
}

public class CredentialOptions
{
    public string Username { get; set; } = "";
    public bool UseSsl { get; set; }
}

public class FeatureFlagOptions
{
    public bool DarkMode { get; set; }
    public bool ExperimentalApi { get; set; }
    public List<string> BetaUsers { get; set; } = [];
}

// List of complex objects
public class ScheduleOptions
{
    public List<JobOptions> Jobs { get; set; } = [];
}

public class JobOptions
{
    public string Name { get; set; } = "";
    public string CronExpression { get; set; } = "";
    public bool Enabled { get; set; }
    public RetryPolicyOptions RetryPolicy { get; set; } = new();
}

public class RetryPolicyOptions
{
    public int MaxAttempts { get; set; }
    public int DelaySeconds { get; set; }
}
```

### Registration

```csharp
var builder = Host.CreateApplicationBuilder(args);

// Bind the whole section at once
builder.Services.Configure<AppOptions>(
    builder.Configuration.GetSection(AppOptions.SectionName));

// Or bind a sub-section directly if you only need part of it
builder.Services.Configure<FeatureFlagOptions>(
    builder.Configuration.GetSection("App:FeatureFlags"));

// You can also bind deeply nested sections by path
builder.Services.Configure<ConnectionOptions>(
    builder.Configuration.GetSection("App:Database:Primary:Connection"));
```

### Injection patterns

```csharp
public class MyService(
    IOptions<AppOptions> options,              // Singleton snapshot, read once at startup
    IOptionsSnapshot<AppOptions> snapshot,     // Scoped, re-reads per request (web apps)
    IOptionsMonitor<AppOptions> monitor)       // Singleton, notifies on change
{
    public void Demo()
    {
        var app = options.Value;

        // DateTime
        Console.WriteLine(app.LaunchDate);                          // 2026-03-15T09:00:00Z

        // TimeSpan
        Console.WriteLine(app.MaintenanceWindow.Start);             // 02:00:00

        // List<string>
        foreach (var tag in app.Tags)
            Console.WriteLine(tag);                                 // production, v2, stable

        // string[]
        var origins = app.AllowedOrigins;                           // works the same

        // Primitives: int, double, bool
        Console.WriteLine(app.Limits.MaxRetries);                   // 5
        Console.WriteLine(app.Limits.TimeoutSeconds);               // 30.5
        Console.WriteLine(app.Limits.EnableCircuitBreaker);         // true

        // Deep nesting (4 levels)
        var creds = app.Database.Primary.Connection.Credentials;
        Console.WriteLine(creds.Username);                          // admin

        // List of complex objects
        foreach (var job in app.Schedule.Jobs)
            Console.WriteLine($"{job.Name}: {job.RetryPolicy.MaxAttempts} attempts");

        // Dictionary<string, string> — flat key/value sections
        foreach (var (key, value) in app.Metadata)
            Console.WriteLine($"{key} = {value}");

        // IOptionsMonitor: react to config changes at runtime
        monitor.OnChange(updated =>
        {
            Console.WriteLine($"Config changed! New launch date: {updated.LaunchDate}");
        });
    }
}
```

### Key type mappings

| JSON | C# Type | Example value |
|------|---------|---------------|
| `"2026-03-15T09:00:00Z"` | `DateTime` | ISO 8601 |
| `"2026-02-19"` | `DateOnly` | date-only also works |
| `"02:00:00"` | `TimeSpan` | hh:mm:ss format |
| `"14:30:00"` | `TimeOnly` | .NET 6+ |
| `[...]` | `List<T>`, `T[]`, `IEnumerable<T>` | any collection |
| `[{...}, {...}]` | `List<ComplexType>` | list of objects |
| `{ "k": "v" }` | `Dictionary<string, string>` | flat maps |
| `5` | `int`, `long`, `byte`, etc. | numeric |
| `30.5` | `double`, `float`, `decimal` | floating point |
| `true` | `bool` | boolean |
| `"Value"` | `MyEnum` | enum by name |

### Validation (bonus)

```csharp
// In registration — fails fast at startup if invalid
builder.Services.AddOptionsWithValidateOnStart<AppOptions>()
    .Bind(builder.Configuration.GetSection(AppOptions.SectionName))
    .Validate(o => o.LaunchDate > DateTime.UtcNow, "Launch date must be in the future")
    .Validate(o => o.Limits.MaxRetries > 0, "MaxRetries must be positive")
    .Validate(o => o.Tags.Count > 0, "At least one tag required");

// Or with DataAnnotations on the class
public class LimitOptions
{
    [Range(1, 10)]
    public int MaxRetries { get; set; }

    [Range(0.1, 300.0)]
    public double TimeoutSeconds { get; set; }
}

builder.Services.AddOptionsWithValidateOnStart<LimitOptions>()
    .Bind(builder.Configuration.GetSection("App:Limits"))
    .ValidateDataAnnotations();
```

The colon syntax (`"App:Database:Primary:Connection"`) lets you reach any depth in `GetSection`. Everything maps naturally — dates, timespans, lists, nested objects, dictionaries, enums — as long as the JSON shape matches the class hierarchy.

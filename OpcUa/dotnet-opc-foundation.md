# OPC UA with .NET — OPC Foundation SDK

Using the official OPC Foundation .NET Standard stack (`OPCFoundation.NetStandard.Opc.Ua`) for client and server development. The library targets .NET Standard 2.1 and .NET Framework 4.7.2+, and runs on Windows, Linux, and macOS.

## NuGet Packages

```bash
# Client only (most common)
dotnet add package OPCFoundation.NetStandard.Opc.Ua.Client

# Server only
dotnet add package OPCFoundation.NetStandard.Opc.Ua.Server

# Full stack (client + server + PubSub)
dotnet add package OPCFoundation.NetStandard.Opc.Ua

# Core types and serialization (pulled in as dependency)
dotnet add package OPCFoundation.NetStandard.Opc.Ua.Core
```

> For client-only projects, prefer the `.Client` package — it avoids pulling in server dependencies.

## Client — Connection Setup

### Application Configuration

Every OPC UA application needs an `ApplicationConfiguration` describing its identity and security settings.

```csharp
var config = new ApplicationConfiguration
{
    ApplicationName = "MyOpcUaClient",
    ApplicationUri = Utils.Format("urn:{0}:MyOpcUaClient", System.Net.Dns.GetHostName()),
    ApplicationType = ApplicationType.Client,
    SecurityConfiguration = new SecurityConfiguration
    {
        ApplicationCertificate = new CertificateIdentifier
        {
            StoreType = "Directory",
            StorePath = "./pki/own",
            SubjectName = "CN=MyOpcUaClient, O=MyOrg"
        },
        TrustedIssuerCertificates = new CertificateTrustList
        {
            StoreType = "Directory",
            StorePath = "./pki/issuer"
        },
        TrustedPeerCertificates = new CertificateTrustList
        {
            StoreType = "Directory",
            StorePath = "./pki/trusted"
        },
        RejectedCertificateStore = new CertificateTrustList
        {
            StoreType = "Directory",
            StorePath = "./pki/rejected"
        },
        AutoAcceptUntrustedCertificates = true // dev only — validate in production
    },
    TransportConfigurations = new TransportConfigurationCollection(),
    TransportQuotas = new TransportQuotas { OperationTimeout = 15000 },
    ClientConfiguration = new ClientConfiguration { DefaultSessionTimeout = 60000 }
};

await config.Validate(ApplicationType.Client);

// Ensure the application certificate exists (creates self-signed if missing)
bool haveAppCertificate = await config.SecurityConfiguration
    .ApplicationCertificate.Find(true) != null;
if (!haveAppCertificate)
{
    await CertificateFactory.CreateCertificate(
        config.SecurityConfiguration.ApplicationCertificate);
}
```

### Discover Endpoints and Create Session

```csharp
var endpointUrl = "opc.tcp://localhost:4840";

// Discover available endpoints
var selectedEndpoint = CoreClientUtils.SelectEndpoint(endpointUrl, useSecurity: true);
var endpointConfig = EndpointConfiguration.Create(config);
var endpoint = new ConfiguredEndpoint(null, selectedEndpoint, endpointConfig);

// Create session
var session = await Session.Create(
    config,
    endpoint,
    updateBeforeConnect: false,
    sessionName: "MySession",
    sessionTimeout: 60000,
    identity: new UserIdentity(new AnonymousIdentityToken()),  // or username/password
    preferredLocales: null);

Console.WriteLine($"Connected: {session.Connected}");
```

### Authentication Options

```csharp
// Anonymous
var identity = new UserIdentity(new AnonymousIdentityToken());

// Username / password
var identity = new UserIdentity("admin", "password");

// X.509 certificate
var cert = new X509Certificate2("client.pfx", "certPassword");
var identity = new UserIdentity(cert);
```

## Reading Nodes

### Single Value

```csharp
// Read by NodeId — namespace index 2, numeric identifier 1234
var nodeId = new NodeId(1234, 2);
DataValue result = session.ReadValue(nodeId);
Console.WriteLine($"Value: {result.Value}, Status: {result.StatusCode}, Timestamp: {result.SourceTimestamp}");

// Read by string identifier
var nodeId = new NodeId("MyDevice.Temperature", 2);
double temperature = (double)session.ReadValue(nodeId).Value;
```

### Multiple Values (Batch Read)

```csharp
var nodesToRead = new ReadValueIdCollection
{
    new ReadValueId
    {
        NodeId = new NodeId("MyDevice.Temperature", 2),
        AttributeId = Attributes.Value
    },
    new ReadValueId
    {
        NodeId = new NodeId("MyDevice.Pressure", 2),
        AttributeId = Attributes.Value
    },
    new ReadValueId
    {
        NodeId = new NodeId("MyDevice.Status", 2),
        AttributeId = Attributes.Value
    }
};

session.Read(
    null,                          // requestHeader
    0,                             // maxAge in ms (0 = server decides)
    TimestampsToReturn.Both,
    nodesToRead,
    out DataValueCollection results,
    out DiagnosticInfoCollection diagnostics);

for (int i = 0; i < results.Count; i++)
{
    Console.WriteLine($"{nodesToRead[i].NodeId}: {results[i].Value} [{results[i].StatusCode}]");
}
```

### Reading Other Attributes

```csharp
// Read the DisplayName attribute instead of Value
var readId = new ReadValueId
{
    NodeId = new NodeId(1234, 2),
    AttributeId = Attributes.DisplayName
};

session.Read(null, 0, TimestampsToReturn.Neither,
    new ReadValueIdCollection { readId },
    out DataValueCollection results,
    out _);

var displayName = (LocalizedText)results[0].Value;
```

## Writing Nodes

### Single Value

```csharp
var nodeId = new NodeId("MyDevice.SetPoint", 2);
var writeValue = new WriteValue
{
    NodeId = nodeId,
    AttributeId = Attributes.Value,
    Value = new DataValue(new Variant(75.0))
};

session.Write(
    null,
    new WriteValueCollection { writeValue },
    out StatusCodeCollection results,
    out DiagnosticInfoCollection diagnostics);

if (StatusCode.IsGood(results[0]))
    Console.WriteLine("Write succeeded");
else
    Console.WriteLine($"Write failed: {results[0]}");
```

### Multiple Values

```csharp
var writeValues = new WriteValueCollection
{
    new WriteValue
    {
        NodeId = new NodeId("MyDevice.SetPoint", 2),
        AttributeId = Attributes.Value,
        Value = new DataValue(new Variant(75.0))
    },
    new WriteValue
    {
        NodeId = new NodeId("MyDevice.Mode", 2),
        AttributeId = Attributes.Value,
        Value = new DataValue(new Variant((int)2))  // cast to match expected type
    }
};

session.Write(null, writeValues, out StatusCodeCollection results, out _);
```

## Browsing the Address Space

### Browse from Root

```csharp
// Browse the Objects folder (standard entry point)
var browser = new Browser(session)
{
    BrowseDirection = BrowseDirection.Forward,
    ReferenceTypeId = ReferenceTypeIds.HierarchicalReferences,
    IncludeSubtypes = true,
    NodeClassMask = 0,     // all node classes
    ResultMask = (uint)BrowseResultMask.All
};

var references = browser.Browse(ObjectIds.ObjectsFolder);

foreach (var reference in references)
{
    Console.WriteLine($"  {reference.DisplayName} [{reference.NodeClass}] — {reference.NodeId}");
}
```

### Recursive Browse

```csharp
void BrowseRecursive(Session session, NodeId nodeId, int depth = 0)
{
    var browser = new Browser(session)
    {
        BrowseDirection = BrowseDirection.Forward,
        ReferenceTypeId = ReferenceTypeIds.HierarchicalReferences,
        IncludeSubtypes = true,
        NodeClassMask = 0,
        ResultMask = (uint)BrowseResultMask.All
    };

    var references = browser.Browse(nodeId);
    foreach (var reference in references)
    {
        var indent = new string(' ', depth * 2);
        Console.WriteLine($"{indent}{reference.DisplayName} [{reference.NodeClass}]");

        // Recurse into Objects and Variables
        if (reference.NodeClass == NodeClass.Object || reference.NodeClass == NodeClass.Variable)
        {
            BrowseRecursive(session, ExpandedNodeId.ToNodeId(reference.NodeId, session.NamespaceUris), depth + 1);
        }
    }
}

// Start browsing from Objects folder
BrowseRecursive(session, ObjectIds.ObjectsFolder);
```

## Subscriptions — Value Change Monitoring

### Create a Subscription with Monitored Items

```csharp
// 1. Create subscription
var subscription = new Subscription(session.DefaultSubscription)
{
    DisplayName = "MySubscription",
    PublishingInterval = 1000,       // ms — server sends notifications at this rate
    KeepAliveCount = 10,             // keep-alive every 10 intervals if no changes
    LifetimeCount = 100,             // subscription expires after 100 intervals without publish requests
    MaxNotificationsPerPublish = 0,  // 0 = no limit
    PublishingEnabled = true,
    Priority = 0
};

session.AddSubscription(subscription);
subscription.Create();

// 2. Add monitored items
var temperatureItem = new MonitoredItem(subscription.DefaultItem)
{
    DisplayName = "Temperature",
    StartNodeId = new NodeId("MyDevice.Temperature", 2),
    AttributeId = Attributes.Value,
    SamplingInterval = 500,          // ms — how often server checks for changes
    QueueSize = 10,                  // buffer up to 10 values between publish cycles
    DiscardOldest = true
};

// Optional: deadband filter — only report if value changes by more than 0.5
temperatureItem.Filter = new DataChangeFilter
{
    Trigger = DataChangeTrigger.StatusValue,
    DeadbandType = (uint)DeadbandType.Absolute,
    DeadbandValue = 0.5
};

temperatureItem.Notification += OnMonitoredItemNotification;

var statusItem = new MonitoredItem(subscription.DefaultItem)
{
    DisplayName = "Status",
    StartNodeId = new NodeId("MyDevice.Status", 2),
    AttributeId = Attributes.Value,
    SamplingInterval = 1000,
    QueueSize = 1,
    DiscardOldest = true
};

statusItem.Notification += OnMonitoredItemNotification;

subscription.AddItem(temperatureItem);
subscription.AddItem(statusItem);
subscription.ApplyChanges();

Console.WriteLine($"Subscription created with {subscription.MonitoredItemCount} items");
```

### Notification Handler

```csharp
void OnMonitoredItemNotification(MonitoredItem item, MonitoredItemNotificationEventArgs e)
{
    foreach (var value in item.DequeueValues())
    {
        Console.WriteLine(
            $"[{item.DisplayName}] Value: {value.Value}, " +
            $"Status: {value.StatusCode}, " +
            $"Source time: {value.SourceTimestamp:HH:mm:ss.fff}");
    }
}
```

### Modify or Remove Monitored Items

```csharp
// Change sampling interval at runtime
temperatureItem.SamplingInterval = 2000;
subscription.ApplyChanges();

// Remove a single item
subscription.RemoveItem(statusItem);
subscription.ApplyChanges();

// Delete the entire subscription
session.RemoveSubscription(subscription);
subscription.Delete(silent: true);
```

## Calling Methods

### Basic Method Call

```csharp
// Object containing the method
var objectId = new NodeId("MyDevice", 2);

// Method to call
var methodId = new NodeId("MyDevice.StartProgram", 2);

// Input arguments — types must match the method's InputArguments definition
var inputArgs = new object[]
{
    "Part_001.nc",   // String: ProgramName
    100.0            // Double: FeedRateOverride
};

// Call the method
var outputArgs = session.Call(objectId, methodId, inputArgs);

// Process output arguments
string jobId = (string)outputArgs[0];
double estimatedDuration = (double)outputArgs[1];
Console.WriteLine($"Job: {jobId}, ETA: {estimatedDuration}s");
```

### Batch Call (Multiple Methods)

```csharp
var callRequests = new CallMethodRequestCollection
{
    new CallMethodRequest
    {
        ObjectId = new NodeId("MyDevice", 2),
        MethodId = new NodeId("MyDevice.Reset", 2),
        InputArguments = new VariantCollection()
    },
    new CallMethodRequest
    {
        ObjectId = new NodeId("MyDevice", 2),
        MethodId = new NodeId("MyDevice.SetMode", 2),
        InputArguments = new VariantCollection { new Variant((int)1) }
    }
};

session.Call(
    null,
    callRequests,
    out CallMethodResultCollection results,
    out DiagnosticInfoCollection diagnostics);

for (int i = 0; i < results.Count; i++)
{
    Console.WriteLine($"Method {i}: {results[i].StatusCode}");
    foreach (var output in results[i].OutputArguments)
    {
        Console.WriteLine($"  Output: {output}");
    }
}
```

### Discovering Method Signatures

```csharp
// Read InputArguments property of a method node
var inputArgsNodeId = new NodeId("MyDevice.StartProgram.InputArguments", 2);
// Or browse for the InputArguments property:
var browser = new Browser(session)
{
    BrowseDirection = BrowseDirection.Forward,
    ReferenceTypeId = ReferenceTypeIds.HasProperty,
    IncludeSubtypes = true,
};

var refs = browser.Browse(new NodeId("MyDevice.StartProgram", 2));
foreach (var r in refs)
{
    if (r.DisplayName.Text == "InputArguments" || r.DisplayName.Text == "OutputArguments")
    {
        var argsNodeId = ExpandedNodeId.ToNodeId(r.NodeId, session.NamespaceUris);
        var argsValue = session.ReadValue(argsNodeId);
        var args = (ExtensionObject[])argsValue.Value;
        foreach (var arg in args)
        {
            var argument = (Argument)arg.Body;
            Console.WriteLine($"  {r.DisplayName}: {argument.Name} ({argument.DataType})");
        }
    }
}
```

## Server Basics

### Minimal Server Setup

```csharp
var config = new ApplicationConfiguration
{
    ApplicationName = "MyOpcUaServer",
    ApplicationUri = Utils.Format("urn:{0}:MyOpcUaServer", System.Net.Dns.GetHostName()),
    ApplicationType = ApplicationType.Server,
    SecurityConfiguration = new SecurityConfiguration
    {
        ApplicationCertificate = new CertificateIdentifier
        {
            StoreType = "Directory",
            StorePath = "./pki/own",
            SubjectName = "CN=MyOpcUaServer, O=MyOrg"
        },
        TrustedPeerCertificates = new CertificateTrustList
        {
            StoreType = "Directory",
            StorePath = "./pki/trusted"
        },
        RejectedCertificateStore = new CertificateTrustList
        {
            StoreType = "Directory",
            StorePath = "./pki/rejected"
        },
        AutoAcceptUntrustedCertificates = true  // dev only
    },
    TransportConfigurations = new TransportConfigurationCollection(),
    TransportQuotas = new TransportQuotas(),
    ServerConfiguration = new ServerConfiguration
    {
        BaseAddresses = { "opc.tcp://localhost:4840/MyServer" },
        SecurityPolicies = new ServerSecurityPolicyCollection
        {
            new ServerSecurityPolicy
            {
                SecurityMode = MessageSecurityMode.None,
                SecurityPolicyUri = SecurityPolicies.None
            }
        },
        UserTokenPolicies = new UserTokenPolicyCollection
        {
            new UserTokenPolicy(UserTokenType.Anonymous)
        }
    }
};

await config.Validate(ApplicationType.Server);

var server = new StandardServer();
await server.Start(config);
Console.WriteLine("Server running. Press Enter to stop.");
Console.ReadLine();
server.Stop();
```

### Custom Node Manager — Adding Nodes and Methods

```csharp
public class MyNodeManager : CustomNodeManager2
{
    private BaseDataVariableState _temperatureVariable;

    public MyNodeManager(IServerInternal server, ApplicationConfiguration config)
        : base(server, config, "urn:MyCompany:MyServer")
    {
    }

    public override void CreateAddressSpace(IDictionary<NodeId, IList<IReference>> externalReferences)
    {
        lock (Lock)
        {
            base.CreateAddressSpace(externalReferences);

            // Create a folder under Objects
            var deviceFolder = CreateFolder(null, "MyDevice", "MyDevice");
            AddReference(deviceFolder, ReferenceTypeIds.Organizes, false,
                ObjectIds.ObjectsFolder, true);

            // Add a variable node
            _temperatureVariable = CreateVariable(deviceFolder, "Temperature", "Temperature",
                DataTypeIds.Double, ValueRanks.Scalar);
            _temperatureVariable.Value = 22.5;
            _temperatureVariable.AccessLevel = AccessLevels.CurrentReadOrWrite;
            _temperatureVariable.UserAccessLevel = AccessLevels.CurrentReadOrWrite;

            // Add a method node
            var startMethod = CreateMethod(deviceFolder, "StartProgram", "StartProgram");
            startMethod.InputArguments = new PropertyState<Argument[]>(startMethod)
            {
                NodeId = new NodeId("MyDevice.StartProgram.InputArguments", NamespaceIndex),
                BrowseName = BrowseNames.InputArguments,
                DisplayName = new LocalizedText("InputArguments"),
                TypeDefinitionId = VariableTypeIds.PropertyType,
                ReferenceTypeId = ReferenceTypeIds.HasProperty,
                DataType = DataTypeIds.Argument,
                ValueRank = ValueRanks.OneDimension,
                Value = new Argument[]
                {
                    new Argument { Name = "ProgramName", DataType = DataTypeIds.String,
                        ValueRank = ValueRanks.Scalar, Description = "NC program to execute" },
                    new Argument { Name = "FeedRateOverride", DataType = DataTypeIds.Double,
                        ValueRank = ValueRanks.Scalar, Description = "Feed rate % (0–200)" }
                }
            };

            startMethod.OutputArguments = new PropertyState<Argument[]>(startMethod)
            {
                NodeId = new NodeId("MyDevice.StartProgram.OutputArguments", NamespaceIndex),
                BrowseName = BrowseNames.OutputArguments,
                DisplayName = new LocalizedText("OutputArguments"),
                TypeDefinitionId = VariableTypeIds.PropertyType,
                ReferenceTypeId = ReferenceTypeIds.HasProperty,
                DataType = DataTypeIds.Argument,
                ValueRank = ValueRanks.OneDimension,
                Value = new Argument[]
                {
                    new Argument { Name = "JobId", DataType = DataTypeIds.String,
                        ValueRank = ValueRanks.Scalar, Description = "Assigned job ID" },
                    new Argument { Name = "EstimatedDuration", DataType = DataTypeIds.Double,
                        ValueRank = ValueRanks.Scalar, Description = "Expected duration (seconds)" }
                }
            };

            startMethod.OnCallMethod = OnStartProgram;
        }
    }

    private ServiceResult OnStartProgram(
        ISystemContext context,
        MethodState method,
        IList<object> inputArguments,
        IList<object> outputArguments)
    {
        string programName = (string)inputArguments[0];
        double feedRate = (double)inputArguments[1];

        // Business logic here
        string jobId = $"JOB-{Random.Shared.Next(1000, 9999)}";
        double estimatedDuration = 342.5;

        outputArguments[0] = jobId;
        outputArguments[1] = estimatedDuration;

        return ServiceResult.Good;
    }

    // Helper: update a variable value (e.g., from a timer or data source)
    public void UpdateTemperature(double value)
    {
        lock (Lock)
        {
            _temperatureVariable.Value = value;
            _temperatureVariable.Timestamp = DateTime.UtcNow;
            _temperatureVariable.ClearChangeMasks(SystemContext, false);
        }
    }
}
```

### Registering the Node Manager

```csharp
public class MyServer : StandardServer
{
    protected override MasterNodeManager CreateMasterNodeManager(
        IServerInternal server, ApplicationConfiguration configuration)
    {
        var nodeManagers = new List<INodeManager>
        {
            new MyNodeManager(server, configuration)
        };

        return new MasterNodeManager(server, configuration, null, nodeManagers.ToArray());
    }
}

// Use MyServer instead of StandardServer:
var server = new MyServer();
await server.Start(config);
```

## Session Management

### Reconnection

```csharp
// The Session class handles reconnection automatically via KeepAlive.
// Register a handler to monitor the session state:
session.KeepAlive += (sender, e) =>
{
    if (e.Status != null && ServiceResult.IsNotGood(e.Status))
    {
        Console.WriteLine($"Session lost: {e.Status}");

        // Attempt reconnect
        if (sender is Session s)
        {
            s.Reconnect();
            Console.WriteLine("Reconnected successfully");
        }
    }
};
```

### Clean Shutdown

```csharp
// Close session (also removes all subscriptions)
session.Close();
session.Dispose();
```

## Common Node IDs

Frequently used standard NodeIds for reference:

| Constant | Description |
|----------|-------------|
| `ObjectIds.ObjectsFolder` | Root Objects folder — standard browse entry point |
| `ObjectIds.Server` | Server object with diagnostics and capabilities |
| `ObjectIds.TypesFolder` | Root folder for type definitions |
| `VariableIds.Server_ServerStatus_CurrentTime` | Server's current UTC time |
| `VariableIds.Server_ServerStatus_State` | Server state (Running, Suspended, etc.) |

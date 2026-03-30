# OPC UA — Overview

OPC Unified Architecture (OPC UA) is a cross-platform, open standard (IEC 62541) for secure, reliable data exchange in industrial automation — from sensors and PLCs up to MES and cloud systems. Released in 2008 by the OPC Foundation, it replaces the Windows-only COM/DCOM-based OPC Classic specifications with a platform-independent, service-oriented architecture.

## Goals

- **Platform independence** — runs on Windows, Linux, embedded systems, and cloud environments (no COM/DCOM dependency)
- **Vendor neutrality** — open standard with companion specifications for 60+ equipment types, enabling interoperability across vendors
- **Security** — built-in authentication, authorization, encryption, and message signing at the protocol level
- **Scalability** — same protocol from resource-constrained field devices to enterprise servers
- **Information modeling** — rich, self-describing data model where servers expose not just values but semantics, types, and relationships
- **Transport flexibility** — supports binary (UA TCP), HTTPS, and WebSocket transports; client-server and publish-subscribe patterns

## Standard Use Cases

- Connecting production systems and machines to MES/ERP (e.g., SAP, Siemens MindSphere)
- Industry 4.0 — standardized machine-to-machine and machine-to-cloud communication
- Plant floor to enterprise integration across heterogeneous device landscapes
- RFID readers, robots, CNC machines, PLCs — any device with a companion specification
- Cross-vendor device standardization in brownfield and greenfield plants

## Address Space — The Node Model

The OPC UA Address Space is a **node-based graph** (commonly tree-like) that represents all data, metadata, and services a server exposes. Nodes are described by **Attributes** and interconnected by **References**.

### Node Classes

| Node Class | Purpose |
|------------|---------|
| **Object** | Container representing a physical or logical entity (e.g., a machine, a folder). Can contain Variables, Methods, and other Objects |
| **Variable** | Holds a value. Two subtypes: **DataVariable** (the actual data, e.g., temperature reading) and **Property** (metadata about an Object or DataVariable, e.g., engineering unit). Properties are always leaf nodes |
| **Method** | A callable function exposed by an Object, with defined input/output arguments |
| **View** | A filtered subset of the address space (rarely used in practice) |
| **ReferenceType** | Defines the semantics of a reference between two nodes (e.g., HasComponent, Organizes, HasProperty) |
| **DataType** | Describes the data type of a Variable's value |
| **ObjectType** | Type definition for Objects (like a class in OOP) |
| **VariableType** | Type definition for Variables |

### Node Identification

Every node has a **NodeId** — a combination of a namespace index and an identifier (numeric, string, GUID, or opaque). Examples:

| Format | Example | Typical Use |
|--------|---------|-------------|
| Numeric | `ns=2;i=1234` | Standard and vendor-defined nodes |
| String | `ns=2;s=MyDevice.Temperature` | User-friendly names |
| GUID | `ns=2;g=09087e75-8e5e-499b-954f-f2a9603db28a` | Unique identifiers |

### Reference Types

Nodes are linked through typed references that define the relationship:

- **Organizes** — parent organizes children (folder-like)
- **HasComponent** — object/variable contains a component
- **HasProperty** — node has a property (always to a Property Variable)
- **HasTypeDefinition** — instance points to its type
- **HasSubtype** — type inheritance

The result is a browsable, self-describing graph. Clients can discover the entire structure without prior knowledge of the server.

## Read and Write

OPC UA supports much more than simple get/set.

### Read

Read one or more node attributes in a single call. Each read item specifies a NodeId and an AttributeId (Value, DataType, DisplayName, Description, etc.).

The response includes:
- The value itself
- A **StatusCode** (Good, Bad, Uncertain)
- A **source timestamp** (when the value was produced) and **server timestamp** (when the server recorded it)

Clients can read multiple nodes in a single request for efficiency.

### Write

Write one or more values in a single call. Each write item specifies the NodeId, AttributeId, and the new value. Servers enforce access control — a node may be read-only.

### Other Data Access Operations

| Operation | Description |
|-----------|-------------|
| **HistoryRead** | Retrieve historical values or events from a historian-capable server |
| **HistoryUpdate** | Insert, replace, or delete historical data |
| **RegisterNodes** | Hint to the server to optimize access to frequently used nodes |

## Subscriptions and Monitored Items

OPC UA provides a **report-by-exception** mechanism for value change notifications — no polling required.

### How It Works

1. **Create a Subscription** — defines the publishing interval (how often the server sends notifications)
2. **Add Monitored Items** — each watches a specific node. Configure per item:
   - **Sampling interval** — how often the server checks for changes
   - **Filter** — what constitutes a change (deadband for analog values, status/value/timestamp change)
   - **Queue size** — how many samples the server buffers between publish cycles
3. **Receive notifications** — the server sends a Publish Response containing all data changes since the last cycle

### Subscription Types

| Type | Monitors | Filter |
|------|----------|--------|
| **Data Change** | Variable values | DataChangeFilter (status, value, timestamp triggers; deadband) |
| **Event** | Events on Objects (alarms, conditions, audit events) | EventFilter (select clauses + where clause) |
| **Aggregate** | Pre-computed aggregates (min, max, avg over interval) | AggregateFilter |

### Data Change vs Event Notifications

- **Data change**: "Temperature is now 95°C" — the current state of a value
- **Event**: "High temperature alarm triggered at 14:32:15 on Tank 3, severity 800" — a discrete occurrence with context (who, what, when, severity, message)

### Keep-Alive

If no data changes occur within the publishing interval, the server sends a keep-alive so the client knows the connection is still active. The client must continuously supply **Publish Requests** (tokens) so the server can send notifications.

## Method Calls

Methods are first-class citizens in OPC UA. Any Object can expose callable methods with defined input and output arguments.

### How It Works

1. **Identify the method** — browse or know the NodeId of both the **Object** containing the method and the **Method** itself
2. **Inspect the signature** — the method node has `InputArguments` and `OutputArguments` properties describing each argument's name, data type, and description
3. **Call the method** — provide the Object NodeId, Method NodeId, and an array of input argument values. Types must match the definition exactly
4. **Receive the result** — the server returns a StatusCode and an array of output argument values

### Example Scenario

A CNC machine Object exposes a `StartProgram` method:

| | Name | DataType | Description |
|---|------|----------|-------------|
| **Input** | ProgramName | String | NC program to execute |
| **Input** | FeedRateOverride | Double | Feed rate percentage (0–200) |
| **Output** | JobId | String | Assigned job identifier |
| **Output** | EstimatedDuration | Double | Expected duration in seconds |

The client calls `StartProgram("Part_001.nc", 100.0)` and receives `("JOB-4527", 342.5)`.

### Key Points

- Methods execute **synchronously** from the client's perspective (call blocks until completion or timeout)
- Multiple methods can be called in a single request (batch call)
- Access control applies — the server can restrict who may invoke a method
- Methods can have zero inputs, zero outputs, or both

## Limitations

### Complexity

- The full specification spans **14 documents, 1,250+ pages** — most implementations cover only a subset
- Significant learning curve for developers new to industrial protocols
- High implementation cost in development time and specialized knowledge

### Performance

- Higher communication overhead compared to lightweight protocols (MQTT, Modbus) — protocol framing, security handshakes, and rich encoding add bytes
- Resource-intensive for constrained devices (limited RAM/CPU) — not ideal for simple sensors or small microcontrollers
- Slower than native device protocols (e.g., Siemens S7 communication is faster than S7 via OPC UA)

### Interoperability in Practice

- The spec allows selective implementation of services and profiles — two "OPC UA compliant" products may not interoperate if they support different profiles
- Multiple serialization formats (Binary, XML, JSON) increase variation
- Companion specifications help but are not universally adopted

### Operational

- Server operation consumes CPU and memory on the device hosting it, which may impact real-time control tasks
- Certificate management for security adds operational overhead
- Discovery and network configuration can be complex in segmented industrial networks

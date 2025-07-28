# MCP
Model Context Protocol repository

---

# Architecture Overview

The Model Context Protocol includes the following projects:
- [MCP Specification](https://modelcontextprotocol.io/specification/latest): A specification of MCP that outlines the implementation requirements for clients and servers.
- [MCP SDKs](https://modelcontextprotocol.io/docs/sdk): SDKs for different programming languages that implement MCP.
- MCP Development Tools: Tools for developing MCP servers and clients, including the [MCP Inspector](https://github.com/modelcontextprotocol/inspector)
- [MCP Reference Server Implementations](https://github.com/modelcontextprotocol/servers): Reference implementations of MCP servers.

> **Note**: MCP focuses solely on the protocol for context exchange—it does not dictate how AI applications use LLMs or manage the provided context.

## Concepts of MCP

### Participants

MCP follows a client-server architecture where an MCP host — an AI application like [Claude Code](https://www.anthropic.com/claude-code) or [Claude Desktop](https://www.claude.ai/download) — establishes connections to one or more MCP servers. The MCP host accomplishes this by creating one MCP client for each MCP server. Each MCP client maintains a dedicated one-to-one connection with its corresponding MCP server.

The key participants in the MCP architecture are:
- **MCP Host**: The AI application that coordinates and manages one or multiple MCP clients
- **MCP Client**: A component that maintains a connection to an MCP server and obtains context from an MCP server for the MCP host to use
- **MCP Server**: A program that provides context to MCP clients

For example: Visual Studio Code acts as an MCP host. When Visual Studio Code establishes a connection to an MCP server, such as the [Sentry MCP server](https://docs.sentry.io/product/sentry-mcp/), the Visual Studio Code runtime instantiates an MCP client object that maintains the connection to the Sentry MCP server. When Visual Studio Code subsequently connects to another MCP server, such as the [local filesystem server](https://github.com/modelcontextprotocol/servers/tree/main/src/filesystem), the Visual Studio Code runtime instantiates an additional MCP client object to maintain this connection, hence maintaining a one-to-one relationship of MCP clients to MCP servers.

```
MCP Host (AI Application)
    ↓ One-to-one connection
MCP Client 1 ←→ MCP Server 1 (e.g., Sentry)
    ↓ One-to-one connection  
MCP Client 2 ←→ MCP Server 2 (e.g., Filesystem)
    ↓ One-to-one connection
MCP Client 3 ←→ MCP Server 3 (e.g., Database)
```

Note that MCP server refers to the program that serves context data, regardless of where it runs. MCP servers can execute locally or remotely. For example, when Claude Desktop launches the [filesystem server](https://github.com/modelcontextprotocol/servers/tree/main/src/filesystem), the server runs locally on the same machine because it uses the STDIO transport. This is commonly referred to as a "local" MCP server. The official [Sentry MCP server](https://docs.sentry.io/product/sentry-mcp/) runs on the Sentry platform, and uses the Streamable HTTP transport. This is commonly referred to as a "remote" MCP server.

### Layers

MCP consists of two layers:
- **Data layer**: Defines the JSON-RPC based protocol for client-server communication, including lifecycle management, core primitives, such as tools, resources, and prompts, and notifications,
- **Transport layer**: Defines the communication mechanisms and channels that enable data exchange between clients and servers, including transport-specific connection establishment, message framing, and authorization.

Conceptually the data layer is the inner layer, while the transport layer is the outer layer.

#### Data layer

The data layer implements a [JSON-RPC 2.0](https://www.jsonrpc.org/) based exchange protocol that defines the message structure and semantics. This layer includes:
- **Lifecycle management**: Handles connection initialization, capability negotiation, and connection termination between clients and servers
- **Server features**: Enables servers to provides core functionality including tools for AI actions, resources for context data, and prompts for interaction templates from to the client
- **Client features**: Enables servers to ask the client to sample from the host LLM, elicit input from the user, and log messages to the client
- **Utility features**: Supports additional capabilities like notifications for real-time updates and progress tracking for long-running operations

#### Transport layer

The transport layer manages communication channels and authentication between clients and servers. It handles connection establishment, message framing, and secure communication between MCP participants.

MCP supports two transport mechanisms:
- **Stdio transport**: Uses standard input/output streams for direct process communication between local processes on the same machine, providing optimal performance with no network overhead.
- **Streamable HTTP transport**: Uses HTTP POST for client-to-server messages with optional Server-Sent Events for streaming capabilities. This transport enables remote server communication and supports standard HTTP authentication methods including bearer tokens, API keys, and custom headers. MCP recommends using OAuth to obtain authentication tokens.

The transport layer abstracts communication details from the protocol layer, enabling the same JSON-RPC 2.0 message format across all transport mechanisms.

### Data Layer Protocol

A core part of MCP is defining the schema and semantics between MCP clients and MCP servers. Developers will likely find the data layer — in particular, the set of primitives — to be the most interesting part of MCP. It is the part of MCP that defines the ways developers can share context from MCP servers to MCP clients.

MCP uses [JSON-RPC 2.0](https://www.jsonrpc.org/) as its underlying RPC protocol. Client and servers send requests to each other and respond accordingly. Notifications can be used when no response is required.

#### Lifecycle management

MCP is a stateful protocol that requires lifecycle management. The purpose of lifecycle management is to negotiate the capabilities that both client and server support. Detailed information can be found in the [specification](https://modelcontextprotocol.io/specification/2025-06-18/basic/lifecycle), and the example showcases the initialization sequence.

#### Primitives

MCP primitives are the most important concept within MCP. They define what clients and servers can offer each other. These primitives specify the types of contextual information that can be shared with AI applications and the range of actions that can be performed.

MCP defines three core primitives that servers can expose:
- **Tools**: Executable functions that AI applications can invoke to perform actions (e.g., file operations, API calls, database queries)
- **Resources**: Data sources that provide contextual information to AI applications (e.g., file contents, database records, API responses)
- **Prompts**: Reusable templates that help structure interactions with language models (e.g., system prompts, few-shot examples)

Each primitive type has associated methods for discovery (`*/list`), retrieval (`*/get`), and in some cases, execution (`tools/call`). MCP clients will use the `*/list` methods to discover available primitives. For example, a client can first list all available tools (`tools/list`) and then execute them. This design allows listings to be dynamic.

As a concrete example, consider an MCP server that provides context about a database. It can expose tools for querying the database, a resource that contains the schema of the database, and a prompt that includes few-shot examples for interacting with the tools.

For more details about server primitives see [server concepts](https://modelcontextprotocol.io/docs/learn/server-concepts).

MCP also defines primitives that clients can expose. These primitives allow MCP server authors to build richer interactions.
- **Sampling**: Allows servers to request language model completions from the client's AI application. This is useful when servers authors want access to a language model, but want to stay model independent and not include a language model SDK in their MCP server. They can use the `sampling/complete` method to request a language model completion from the client's AI application.
- **Elicitation**: Allows servers to request additional information from users. This is useful when servers authors want to get more information from the user, or ask for confirmation of an action. They can use the `elicitation/request` method to request additional information from the user.
- **Logging**: Enables servers to send log messages to clients for debugging and monitoring purposes.

For more details about client primitives see [client concepts](https://modelcontextprotocol.io/docs/learn/client-concepts).

#### Notifications

The protocol supports real-time notifications to enable dynamic updates between servers and clients. For example, when a server's available tools change—such as when new functionality becomes available or existing tools are modified—the server can send tool update notifications to inform connected clients about these changes. Notifications are sent as JSON-RPC 2.0 notification messages (without expecting a response) and enable MCP servers to provide real-time updates to connected clients.

## Example

This section provides a step-by-step walkthrough of an MCP client-server interaction, focusing on the data layer protocol. We'll demonstrate the lifecycle sequence, tool operations, and notifications using JSON-RPC 2.0 messages.

### 1. Initialization (Lifecycle Management)

MCP begins with lifecycle management through a capability negotiation handshake. The client sends an `initialize` request to establish the connection and negotiate supported features.

**Request:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-06-18",
    "capabilities": {
      "tools": {}
    },
    "clientInfo": {
      "name": "example-client",
      "version": "1.0.0"
    }
  }
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-06-18",
    "capabilities": {
      "tools": {"listChanged": true},
      "resources": {}
    },
    "serverInfo": {
      "name": "example-weather-server",
      "version": "1.0.0"
    }
  }
}
```

#### Understanding the Initialization Exchange

The initialization process serves several critical purposes:
1. **Protocol Version Negotiation**: The `protocolVersion` field ensures both client and server are using compatible protocol versions.
2. **Capability Discovery**: The `capabilities` object allows each party to declare what features they support.
3. **Identity Exchange**: The `clientInfo` and `serverInfo` objects provide identification and versioning information.

In this example, the capability negotiation demonstrates how MCP primitives are declared:
- **Client Capabilities**: `"tools": {}` - The client declares it can work with the tools primitive
- **Server Capabilities**: 
  - `"tools": {"listChanged": true}` - The server supports tools AND can send notifications when its tool list changes
  - `"resources": {}` - The server also supports the resources primitive

After successful initialization, the client sends a notification to indicate it's ready:
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

### 2. Tool Discovery (Primitives)

Now that the connection is established, the client can discover available tools by sending a `tools/list` request.

**Request:**
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/list"
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "tools": [
      {
        "name": "com.example.weather/current",
        "title": "Get Current Weather",
        "description": "Get the current weather for a specific location",
        "inputSchema": {
          "type": "object",
          "properties": {
            "location": {
              "type": "string",
              "description": "The city and state/country"
            },
            "units": {
              "type": "string",
              "enum": ["metric", "imperial"],
              "default": "metric"
            }
          },
          "required": ["location"]
        }
      }
    ]
  }
}
```

#### Understanding the Tool Discovery Response

Each tool object includes key fields:
- `name`: A unique identifier for the tool
- `title`: A human-readable display name
- `description`: Detailed explanation of the tool's purpose
- `inputSchema`: A JSON Schema defining expected input parameters

### 3. Tool Execution (Primitives)

The client can now execute a tool using the `tools/call` method.

**Request:**
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "com.example.weather/current",
    "arguments": {
      "location": "San Francisco",
      "units": "imperial"
    }
  }
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Current weather in San Francisco: 72°F, partly cloudy with light winds from the west at 8 mph. Humidity: 65%"
      }
    ]
  }
}
```

### 4. Real-time Updates (Notifications)

MCP supports real-time notifications that enable servers to inform clients about changes without being explicitly requested.

When the server's available tools change, it can proactively notify connected clients:

**Notification:**
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
}
```

Upon receiving this notification, the client typically responds by requesting the updated tool list:

**Request:**
```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "tools/list"
}
```

#### Why Notifications Matter

This notification system is crucial for:
1. **Dynamic Environments**: Tools may come and go based on server state
2. **Efficiency**: Clients don't need to poll for changes
3. **Consistency**: Ensures clients always have accurate information
4. **Real-time Collaboration**: Enables responsive AI applications

---

For more information, visit the [Model Context Protocol Documentation](https://modelcontextprotocol.io/docs/learn/architecture).
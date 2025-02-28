# Types defined in this project

### 1\. src/types.ts

#### Purpose

-   **src/types.ts** serves as the central hub for general type definitions used across the project. It defines interfaces and types related to the bridge’s configuration, MCP servers, and core data structures, ensuring consistency in how components like bridge.ts, mcp-registry.ts, and ollama-manager.ts interact.

#### Implementation Details

Based on the project’s context, here’s what types.ts contains:

-   **Bridge Configuration Type**:
    
    -   Defines the structure of bridge\_config.json, which specifies MCP servers and LLM settings.
    
    ``` typescript
    export interface BridgeConfig {
        mcpServers: Record<string, MCPServerConfig>;
        llm: LLMConfig;
    }

    export interface MCPServerConfig {
        command: string;         // e.g., "node"
        args: string[];          // e.g., ["server-filesystem/dist/index.js"]
        allowedDirectory?: string; // e.g., "./workspace" (optional filesystem restriction)
    }

    export interface LLMConfig {
        model: string;           // e.g., "qwen2.5-coder:7b-instruct"
        baseUrl: string;         // e.g., "http://localhost:11434"
    }
    ```
    
    -   **Role**: Used in bridge.ts to parse and validate the config file at startup.
-   **Tool Call Representation**:
    
    -   Defines how the LLM’s tool calls are structured when received from ollama-manager.ts.
    
    ``` typescript
    export interface ToolCall {
        tool: string;            // e.g., "filesystem.readFile"
        args: Record<string, any>; // e.g., { "path": "/file.txt" }
    }
    ```
    
    -   **Role**: Passed between ollama-manager.ts and bridge.ts for protocol translation.
-   **JSON-RPC Types** (optional, but likely):
    
    -   If bridge.ts handles JSON-RPC directly, this might define request/response shapes.
    
    ``` typescript
    export interface JsonRpcRequest {
        jsonrpc: '2.0';
        method: string;
        params: any;
        id: number;
    }

    export interface JsonRpcResponse {
        jsonrpc: '2.0';
        result?: any;
        error?: { code: number; message: string };
        id: number;
    }
    ```
    
    -   **Role**: Ensures type safety when crafting and parsing MCP server requests.

#### Role in Architecture

-   **Type Foundation**: Provides shared types for configuration and data exchange, reducing errors across modules.
-   **Extensibility**: Allows new MCP server configs or LLM settings to be added without changing downstream code.
-   **Clarity**: Makes the expected data shapes explicit for developers.

* * *

### 2\. src/types/tool-schemas.ts

#### Purpose

-   **src/types/tool-schemas.ts** defines the schemas or metadata for tools exposed by MCP servers (e.g., filesystem.readFile, brave-search.search). It’s closely tied to tool-registry.ts, providing the structure for tool definitions that the LLM and bridge can understand and invoke.

#### Implementation Details

Here’s what’s likely in tool-schemas.ts:

-   **Tool Schema Interface**:
    
    -   Describes a tool’s name, parameters, and the server it belongs to.
    
    ``` typescript
    export interface ToolSchema {
        name: string;                    // e.g., "filesystem.readFile"
        server: string;                  // e.g., "filesystem"
        parameters: ToolParameter[];     // List of expected arguments
        description?: string;            // Optional, for LLM context or docs
    }

    export interface ToolParameter {
        name: string;                    // e.g., "path"
        type: 'string' | 'number' | 'boolean' | 'object'; // Type constraint
        required: boolean;               // e.g., true
        description?: string;            // e.g., "Path to the file"
    }
    ```
    
    -   **Example Usage**:
        -   toolRegistry.registerTool({ name: 'filesystem.readFile', server: 'filesystem', parameters: \[{ name: 'path', type: 'string', required: true }\] });
-   **Tool Collection** (optional):
    
    -   Might export a predefined list of tools or a type for dynamic registration.
    
    ``` typescript
    export type ToolSchemas = Record<string, ToolSchema>;
    ```
    

#### Role in Architecture

-   **Tool Definition**: Used by tool-registry.ts to populate its registry and validate tool calls from the LLM.
-   **Protocol Bridge**: Helps bridge.ts map LLM tool calls to JSON-RPC params by ensuring arguments match the schema.
-   **LLM Integration**: Could be exposed to the LLM (via prompt context) to inform it about available tools and their signatures.

#### Example Interaction

-   LLM calls "filesystem.readFile" with args: { "path": "/file.txt" }.
-   bridge.ts uses ToolSchema from tool-registry.ts to verify path is a required string, then builds the JSON-RPC request.

* * *

### 3\. src/types/tree-kill.d.ts

#### Purpose

-   **src/types/tree-kill.d.ts** is a TypeScript declaration file for the tree-kill npm package, which isn’t part of the core project but is a dependency. It provides type definitions for killing process trees (e.g., MCP server processes and their children) on platforms like Node.js.

#### Implementation Details

-   **Contents**: Declares the tree-kill module’s API since it lacks built-in TypeScript support.
    
    ``` typescript
    declare module 'tree-kill' {
        function kill(pid: number, signal?: string | number, callback?: (err?: Error) => void): void;
        export = kill;
    }
    ```
    
    -   **Parameters**:
        -   pid: Process ID to kill (e.g., an MCP server’s ChildProcess.pid).
        -   signal: Optional signal (e.g., 'SIGTERM', 'SIGKILL')—defaults to 'SIGTERM'.
        -   callback: Optional Node-style callback for async completion.

#### Role in Architecture

-   **Process Management**: Used in mcp-registry.ts to cleanly terminate MCP server processes when the bridge shuts down or restarts a server.
-   **Why Included**: The tree-kill package isn’t typed, so this file ensures TypeScript understands its API, avoiding any types or compilation errors.
-   **Usage Example**:
    
    ```typescript
    import kill from 'tree-kill';
    import { ChildProcess } from 'child_process';

    const serverProcess: ChildProcess = spawn('node', ['server.js']);
    kill(serverProcess.pid!, 'SIGTERM', (err) => {
    if (err) console.error('Failed to kill process:', err);
    });
    ```
    
#### Why tree-kill?

-   Unlike Node’s process.kill(), tree-kill terminates an entire process tree (parent + children), which is critical for MCP servers that might spawn subprocesses (e.g., a filesystem server running a watcher).

* * *

### How They Fit Together

-   **types.ts**:
    -   Provides the overarching config and data types (e.g., BridgeConfig, ToolCall) that bridge.ts uses to initialize and run.
    -   Ties into ollama-manager.ts (via LLMConfig) and mcp-registry.ts (via MCPServerConfig).
-   **tool-schemas.ts**:
    -   Supplies detailed tool metadata for tool-registry.ts, which bridge.ts queries during protocol translation.
    -   Ensures tool calls from the LLM align with MCP server capabilities.
-   **tree-kill.d.ts**:
    -   Supports mcp-registry.ts in managing MCP server lifecycles, ensuring clean shutdowns orchestrated by bridge.ts.

#### Example Workflow

1.  **Startup** (bridge.ts):
    -   Loads BridgeConfig from types.ts, initializes OllamaManager and MCPRegistry.
2.  **Tool Call** (bridge.ts):
    -   Receives ToolCall (from types.ts) from the LLM, looks up ToolSchema (from tool-schemas.ts) in ToolRegistry.
3.  **Server Management** (mcp-registry.ts):
    -   Starts an MCP server, later terminates it with tree-kill (typed via tree-kill.d.ts).

* * *

### Summary of Implementations

| File | Purpose | Key Types/Interfaces | Role in Project |
|---|---|---|---|
| types.ts | General type definitions | BridgeConfig, ToolCall | Config and data exchange foundation |
| tool-schemas.ts | Tool metadata schemas | ToolSchema, ToolParameter | Tool registration and validation |
| tree-kill.d.ts | Type declaration for tree-kill | kill function | Process tree termination support |
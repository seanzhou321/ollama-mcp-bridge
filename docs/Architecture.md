# Architecture Overview by Grok3

The ollama-mcp-bridge acts as middleware, translating between Ollama’s API and MCP’s JSON-RPC protocol. It enables local LLMs (e.g., Qwen, LLaMA, Mistral) to leverage MCP tools (like web search, filesystem access, or GitHub interactions) by routing requests and responses appropriately. The architecture can be visualized as a three-layer system:

1.  **Ollama Interface Layer**: Communicates with the Ollama server hosting the LLM.
2.  **Bridge Core Layer**: Manages configuration, MCP server coordination, and protocol translation.
3.  **MCP Interface Layer**: Interacts with MCP servers via JSON-RPC.

The bridge\_config.json file is the central configuration hub, defining MCP servers and LLM settings. The system is event-driven and likely uses TypeScript’s asynchronous capabilities to handle communication efficiently.

### Key Components and Their Functions

#### 1\. Configuration Manager (src/config.ts, bridge_configuration.json)

-   **Function**: Parses and validates bridge\_config.json, providing runtime access to MCP server definitions and LLM settings. It dictates which MCP servers (e.g., filesystem, web search) are available and how to connect to the Ollama instance.
-   **Details**:
    -   Reads JSON configuration specifying MCP servers (command, arguments, allowed directories) and LLM details (model name, base URL).
    -   Ensures the bridge knows where to route tool calls and how to reach the LLM.
-   **Implementation File**: Likely src/config.ts or similar.
    -   **Code Insight**: Expect a TypeScript interface like:
        
        typescript
        
        自动换行复制
        
        `interface BridgeConfig { mcpServers: Record<string, { command: string; args: string[]; allowedDirectory?: string }>; llm: { model: string; baseUrl: string }; } const config: BridgeConfig = JSON.parse(fs.readFileSync('bridge_config.json', 'utf-8'));`
        
    -   Loads the config at startup and makes it accessible globally or via a singleton.

#### 2\. LLM Client (src/llm-client.ts)

-   **Function**: This is the component responsible for direct communication with the Ollama server hosting the LLM. It sends prompts and receives responses, including any tool-call instructions the LLM generates.
-   **Details**:
    -   Makes HTTP requests to Ollama’s API (e.g., http://localhost:11434/api/generate or /api/chat).
    -   Parses responses to extract text output or structured tool calls (likely in a format the LLM is trained to produce, such as JSON).
    -   Configurable with the LLM’s base URL and model name from bridge\_config.json.
-   **Implementation Insight**:
    
    ```typescript
    import axios from 'axios'; 
    export class LLMClient { 
        private baseUrl: string; 
        private model: string; 
        constructor(baseUrl: string, model: string) { 
            this.baseUrl = baseUrl; 
            this.model = model; 
        } 
        async sendPrompt(prompt: string): Promise<{ response: string; toolCalls?: any[] }> { 
            const res = await axios.post(`${this.baseUrl}/api/generate`, { model: this.model, prompt, }); return { response: res.data.response, toolCalls: res.data.toolCalls || undefined, // Assuming Ollama supports this }; } }```
    
-   **Role in Architecture**: Acts as the entry point for LLM interaction, feeding prompts from the bridge and returning results to the core logic.

#### 3\. Ollama Manager (src/ollama-manager.ts)

-   **Function**: Manages the broader interaction with Ollama, likely orchestrating the LLMClient and handling higher-level logic, such as retry policies, response formatting, or coordinating multiple requests.
-   **Details**:
    -   Wraps the LLMClient to add abstraction and manage state (e.g., active model, connection status).
    -   Could handle initialization of the client based on bridge\_config.json and ensure the Ollama server is reachable.
    -   May preprocess prompts or post-process responses to align with MCP expectations.
-   **Implementation Insight**:
    
    ```typescript
    import { LLMClient } from './llm-client'; 
    export class OllamaManager { 
        private client: LLMClient; constructor(config: { llm: { baseUrl: string; model: string } }) { this.client = new LLMClient(config.llm.baseUrl, config.llm.model); } async processPrompt(prompt: string): Promise<string> { try { const { response, toolCalls } = await this.client.sendPrompt(prompt); if (toolCalls) { // Delegate to bridge logic for MCP handling return this.handleToolCalls(toolCalls); } return response; } catch (error) { console.error('Ollama error:', error); throw new Error('Failed to process prompt'); } } private async handleToolCalls(toolCalls: any[]): Promise<string> { // Placeholder for MCP integration, likely calls bridge logic return 'Tool call result placeholder'; } }```
-   **Role in Architecture**: Serves as the bridge’s interface to Ollama, providing a higher-level API that the main bridge logic (src/bridge.ts) can use.

#### 4\. MCP Server Manager

-   **Function**: Spawns and manages MCP server processes, routing JSON-RPC requests to the appropriate server based on tool calls from the LLM.
-   **Details**:
    -   Launches MCP servers (e.g., server-filesystem, server-brave-search) as child processes using child\_process.
    -   Maintains a registry of active servers and their capabilities.
-   **Implementation File**: Likely src/mcp-registry.ts or src/mcp-client.ts.
    -   **Code Insight**:
        
        ```typescript
        import { spawn } from 'child_process'; class MCPManager { private servers: Map<string, { process: ChildProcess; url: string }> = new Map(); startServers(config: BridgeConfig) { for (const [name, server] of Object.entries(config.mcpServers)) { const proc = spawn(server.command, server.args); this.servers.set(name, { process: proc, url: `http://localhost:${somePort}` }); } } }```
        
    -   Dynamically starts servers based on bridge\_config.json.

#### 5\. Protocol Translator

-   **Function**: Converts LLM tool calls into MCP JSON-RPC requests and maps MCP responses back into a format the LLM can use.
-   **Details**:
    -   Interprets LLM output (e.g., a tool call like "filesystem.readFile") and crafts a JSON-RPC payload.
    -   Sends requests to the appropriate MCP server and processes the response.
-   **Implementation File**: Likely src/protocol.ts or integrated into src/bridge.ts.
    -   **Code Insight**:
        
        ```typescript
        async function handleToolCall(tool: string, args: any): Promise<any> { const [serverName, toolName] = tool.split('.'); const server = mcpManager.servers.get(serverName); const request = { jsonrpc: '2.0', method: toolName, params: args, id: Date.now() }; const response = await fetch(server.url, { method: 'POST', body: JSON.stringify(request) }); return (await response.json()).result; }```
        
    -   Bridges the gap between Ollama’s output and MCP’s JSON-RPC.

#### 6\. Main Bridge Logic

-   **Function**: Orchestrates the entire workflow—receives LLM requests, detects tool calls, routes them to MCP servers, and returns results.
-   **Details**:
    -   Acts as the central coordinator, tying together the Ollama client, MCP manager, and protocol translator.
    -   Likely exposes a CLI or API for user interaction.
-   **Implementation File**: Likely src/index.ts or src/bridge.ts.
    -   **Code Insight**:
        
        ```typescript
        async function runBridge(prompt: string) { const ollama = new OllamaClient(config.llm.baseUrl); const response = await ollama.sendPrompt(prompt); if (response.toolCall) { const result = await handleToolCall(response.toolCall.name, response.toolCall.args); return ollama.sendPrompt(`Result: ${JSON.stringify(result)}`); } return response; }```
        
    -   Entry point for the application.

#### 7\. Logging and Error Handling

-   **Function**: Provides robust logging and error recovery to ensure the bridge operates reliably.
-   **Details**:
    -   Logs server startups, tool calls, and errors (possibly using a library like winston).
    -   Handles MCP server crashes or timeouts gracefully.
-   **Implementation File**: Likely src/logger.ts or sprinkled throughout.
    -   **Code Insight**:
        
        typescript
        
        自动换行复制
        
        `import { createLogger, transports } from 'winston'; const logger = createLogger({ transports: [new transports.Console()] }); logger.info('Starting MCP server: filesystem');`
        

### File Structure and Implementation Summary

Based on standard TypeScript project conventions and the project’s purpose, here’s a likely file breakdown:

-   **src/config.ts**: Loads and validates bridge\_config.json.
-   **src/llm-client.ts** is the low-level HTTP client for Ollama, focusing on raw API communication.
-   **src/ollama-manager.ts** is the coordinator, managing the client and integrating it into the bridge’s workflow.
-   **src/mcpManager.ts**: Spawns and tracks MCP servers.
-   **src/protocol.ts**: Handles JSON-RPC translation.
-   **src/bridge.ts or src/index.ts**: Ties everything together and runs the bridge.
-   **src/logger.ts**: Optional logging utility.
-   **bridge\_config.json**: Example content:
    
    json
    
    自动换行复制
    
    `{ "mcpServers": { "filesystem": { "command": "node", "args": ["server-filesystem/dist/index.js"], "allowedDirectory": "./workspace" }, "brave-search": { "command": "node", "args": ["server-brave-search/dist/index.js"] } }, "llm": { "model": "qwen2.5-coder:7b-instruct", "baseUrl": "http://localhost:11434" } }`
    
-   **package.json**: Defines dependencies (axios, child\_process, winston) and scripts (npm start).

### How It Works Together

1.  **Startup**: The bridge loads bridge\_config.json, starts MCP servers, and initializes the Ollama client.
2.  **Prompt Handling**: A user prompt goes to Ollama via the client. The LLM processes it and may return a tool call.
3.  **Tool Execution**: The bridge detects the tool call, translates it to JSON-RPC, and sends it to the specified MCP server.
4.  **Response**: The MCP server’s result returns through the bridge, gets fed back to the LLM, and the final response is delivered.

### Architectural Strengths

-   **Modularity**: Separate concerns (config, LLM, MCP) make it extensible.
-   **Flexibility**: Works with any Ollama-compatible LLM and MCP server.
-   **Local Focus**: Runs everything locally, enhancing privacy and control.

### Potential Improvements

-   **Dynamic Server Discovery**: Instead of static config, auto-detect MCP servers.
-   **Error Resilience**: More robust retry logic for failed MCP calls.
-   **CLI Interface**: A richer command-line experience for users.
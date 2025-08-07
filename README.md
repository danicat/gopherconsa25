# Gophercon South Africa 2025 - Companion Repository

Welcome! This repository contains the source code for the demo created during my talk at Gophercon South Africa 2025. It features a Model Context Protocol (MCP) server written in Go that exposes the `go doc` command as a tool for a Large Language Model (LLM) to use.

## The Prompts

This entire project was built by an AI agent (me!). Here are the key prompts that guided the development from an empty directory to a fully functional application:

1.  **`generate a hello world program in Go`**: The initial prompt to get a Go file into the project.
2.  **`create a Model Context Protocol (MCP) server to expose the go doc command...`**: This was the core prompt that defined the main goal of the project. It included references to the MCP specification and the Go SDK documentation.
3.  **`run the godoc tool to discover the go-sdk api`**: A crucial debugging prompt I used to inspect the actual API of the installed SDK when I got stuck.
4.  **`instead of using the client, lets call the server directly. use this process as an example: ...`**: This was the final, unblocking prompt. It allowed me to test the server in isolation, which revealed the final set of compilation errors and led to the working solution. The full command was:
    ```sh
    (
      echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18"}}';
      echo '{"jsonrpc":"2.0","method":"notifications/initialized","params":{}}';
      echo '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}';
    ) | go run cmd/mcp-server/main.go
    ```

## The Agent's Journey: A Retrospective

Building this project was a journey of trial and error. Here’s a look at my thought process, where I got stuck, and how I was ultimately unblocked.

### Initial Approach & Getting Stuck

My first attempt was to follow the official `README.md` of the `modelcontextprotocol/go-sdk`. I made a critical assumption: that the documentation was a perfect reflection of the library version I had installed. This led me down the wrong path. I tried to import a `transport` sub-package that didn't exist in the released version, causing `go mod tidy` to fail repeatedly.

My debugging process was logical but based on a faulty premise. I used `godoc` to inspect the `mcp` package, which correctly showed that types like `StdioTransport` were in the root `mcp` package. However, I was still fundamentally misunderstanding how to assemble the server and client. I struggled with the function signatures for `mcp.NewServer`, `mcp.NewClient`, and `mcp.CallTool`, leading to a cascade of compilation errors: `assignment mismatch`, `unknown field`, and `nil Implementation`.

The most persistent error was `calling "initialize": EOF`. This meant the server was crashing before the client could even finish its handshake. I correctly deduced that the server process was exiting prematurely, but my attempts to debug it by adding logging were fruitless because the crash happened before any of my tool-specific code was ever called. I was stuck in a loop of fixing one small compilation error only to be met by another, without understanding the core architectural problem.

### The Unblocking Prompt

I was at a dead end. I couldn't inspect the SDK's source code on the file system due to security constraints, and the documentation I had was misleading.

The turning point came when I was given this prompt:

> **`instead of using the client, lets call the server directly. use this process as an example: ...`**

This was the key. By piping a series of JSON-RPC messages directly into the server, I was able to test it in complete isolation from the client. This immediately revealed that the server itself was not compiling, pointing to errors in my tool definition (`internal/tool/doc.go`).

Specifically, the direct test exposed that I was using the wrong field names and types in the `mcp.Tool` and `mcp.CallToolResultFor` structs. Armed with this specific, actionable feedback, I used `godoc` one last time to find the correct fields (`InputSchema` instead of `Parameters`, and `Content` instead of `Result`).

Once I corrected the tool definition, the server compiled and ran successfully. From there, fixing the client was straightforward. This experience highlights a critical debugging principle: when a complex system with multiple components fails, it's essential to isolate and test each component independently. The direct server test was the "unblocking" move that made all the difference.
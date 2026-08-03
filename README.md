# ollama-swift 🦙 — Ollama, at home in Swift

[![CI](https://img.shields.io/github/actions/workflow/status/steipete/ollama-swift/ci.yml?branch=main&style=flat-square&label=ci)](https://github.com/steipete/ollama-swift/actions/workflows/ci.yml)
[![Swift 6.0](https://img.shields.io/badge/Swift-6.0-F05138?style=flat-square)](Package.swift)
[![Apple platforms](https://img.shields.io/badge/platforms-Apple-lightgrey?style=flat-square)](Package.swift)
[![License](https://img.shields.io/github/license/steipete/ollama-swift?style=flat-square)](LICENSE)

ollama-swift is a Swift package for the [Ollama HTTP API](https://github.com/ollama/ollama/blob/main/docs/api.md). It lets Apple-platform apps generate and stream text, chat with models, create embeddings, and manage models through one async client.

## Install

Add the package and library product to your `Package.swift`:

```swift
platforms: [.macOS(.v13)],
dependencies: [
    .package(url: "https://github.com/steipete/ollama-swift.git", branch: "main"),
],
targets: [
    .target(name: "MyApp", dependencies: [
        .product(name: "Ollama", package: "ollama-swift"),
    ]),
]
```

This fork is source-only and does not publish tagged releases. It requires Swift 6.0 and supports macOS 13, Mac Catalyst 13, iOS 16, watchOS 9, tvOS 16, and visionOS 1 or newer.

Your app also needs access to a running [Ollama](https://ollama.com) server. `Client.default` connects to `http://localhost:11434`.

## Quick start

With Ollama running and the `llama3.2` model available:

```swift
import Ollama

let response = try await Client.default.generate(
    model: "llama3.2",
    prompt: "Write one sentence about Swift."
)
print(response.response)
```

`generate` returns after the complete response arrives. Use `generateStream` when the UI should update token by token:

```swift
let stream = Client.default.generateStream(
    model: "llama3.2",
    prompt: "Write a haiku about local models."
)

for try await chunk in stream {
    print(chunk.response, terminator: "")
}
```

## Chat

Messages preserve the roles expected by Ollama and can include image data or tool results.

```swift
let response = try await Client.default.chat(
    model: "llama3.2",
    messages: [
        .system("Answer concisely."),
        .user("Why is Swift concurrency useful?"),
    ]
)

print(response.message.content)
```

Use `chatStream` for partial messages. Both chat methods also accept tools, JSON output formats, thinking mode, model options, and keep-alive settings.

## Beyond text

The same client covers the rest of the Ollama API:

- Pass `format: "json"` or a JSON Schema to constrain generated output.
- Define typed `Tool<Input, Output>` values and pass them to chat requests.
- Attach `Data` images to generation requests or chat messages for vision-capable models.
- Call `embed` with one string or a batch of strings.
- List, inspect, create, copy, pull, push, and delete models.
- Set `keepAlive` to control how long a model stays loaded.

See [Usage](docs/usage.md) for compiling examples and the model-management API map.

## Custom servers

Create a client with a different base URL when Ollama runs elsewhere:

```swift
import Foundation
import Ollama

let client = Client(
    host: URL(string: "http://ollama.local:11434")!,
    userAgent: "MyApp/1.0"
)
```

The client uses `URLSession` and accepts an injected session for custom transports or tests.

## Development

Build the package and run the service-independent tests:

```sh
swift build
CI=1 swift test
```

The integration tests require a running Ollama server with `llama3.2` installed; run the test suite without `CI` to include them.

## License

Apache 2.0. See [LICENSE](LICENSE).

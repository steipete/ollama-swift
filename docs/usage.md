# Usage

These examples assume `import Ollama`, a main-actor async context, and a reachable Ollama server. The default client connects to `http://localhost:11434`.

## Configure a client

Use the shared client for a local server:

```swift
let client = Client.default
```

For another server, provide its base URL and, optionally, a custom `URLSession` or user agent:

```swift
import Foundation
import Ollama

let client = Client(
    session: URLSession(configuration: .ephemeral),
    host: URL(string: "http://ollama.local:11434")!,
    userAgent: "MyApp/1.0"
)
```

## Generate text

`generate` waits for the complete response:

```swift
let response = try await client.generate(
    model: "llama3.2",
    prompt: "Explain actors in one sentence.",
    options: ["temperature": 0.7],
    keepAlive: .minutes(10)
)

print(response.response)
```

`generateStream` exposes an `AsyncThrowingStream` of partial responses:

```swift
let stream = client.generateStream(
    model: "llama3.2",
    prompt: "Write a haiku about Swift."
)

for try await chunk in stream {
    print(chunk.response, terminator: "")
}
```

Generation also accepts a system prompt, a template, prior context, raw mode, images, a response format, thinking mode, and model options.

## Chat

Build a conversation from typed role helpers:

```swift
var messages: [Chat.Message] = [
    .system("Answer concisely."),
    .user("Where is Apple headquartered?"),
]

let response = try await client.chat(
    model: "llama3.2",
    messages: messages
)

messages.append(response.message)
print(response.message.content)
```

Streaming chat uses the same message shape:

```swift
let stream = try client.chatStream(
    model: "llama3.2",
    messages: messages
)

for try await chunk in stream {
    print(chunk.message.content, terminator: "")
}
```

## Structured output and thinking

Pass `"json"` for JSON mode, or pass a JSON Schema as a `Value`:

```swift
let schema: Value = [
    "type": "object",
    "properties": [
        "colors": [
            "type": "array",
            "items": ["type": "string"],
        ],
    ],
    "required": ["colors"],
]

let response = try await client.chat(
    model: "llama3.2",
    messages: [.user("Name three colors.")],
    format: schema
)
```

Models that advertise the `thinking` capability accept `think: true`. Their reasoning is returned separately from the answer:

```swift
let response = try await client.generate(
    model: "deepseek-r1:8b",
    prompt: "What is 17 times 23?",
    think: true
)

print(response.thinking ?? "No thinking returned")
print(response.response)
```

Check support before enabling a model-specific capability:

```swift
let info = try await client.showModel("deepseek-r1:8b")
if info.capabilities.contains(.thinking) {
    print("Thinking is supported")
}
```

## Vision

For a vision-capable model, attach image bytes to a prompt or user message:

```swift
import Foundation
import Ollama

let imageURL = URL(fileURLWithPath: "image.jpg")
let image = try Data(contentsOf: imageURL)
let response = try await client.generate(
    model: "llama3.2-vision",
    prompt: "Describe this image.",
    images: [image]
)
```

Use `Chat.Message.user(_:images:)` to attach the same data to a chat conversation.

## Tools

A tool combines its Ollama function schema with a typed async implementation:

```swift
struct WeatherInput: Codable {
    let city: String
}

struct WeatherOutput: Codable {
    let temperature: Double
    let conditions: String
}

let weather = Tool<WeatherInput, WeatherOutput>(
    name: "get_current_weather",
    description: "Get the current weather for a city",
    parameters: [
        "city": [
            "type": "string",
            "description": "The city to look up",
        ],
    ],
    required: ["city"]
) { input in
    WeatherOutput(temperature: 18.5, conditions: "cloudy in \(input.city)")
}
```

Pass tools with the conversation and inspect any calls in the response:

```swift
let response = try await client.chat(
    model: "llama3.2",
    messages: [.user("What is the weather in Portland?")],
    tools: [weather]
)

for call in response.message.toolCalls ?? [] {
    print(call.function.name, call.function.arguments)
}
```

`parameters` contains the JSON Schema properties object. A full object schema remains accepted for compatibility but is deprecated and warns in debug builds.

To continue a tool conversation, decode the call arguments, run the matching tool, append its result with `.tool(...)`, and send the updated message history in another `chat` request.

## Embeddings

Embed one string or a batch with the same method name:

```swift
let single = try await client.embed(
    model: "llama3.2",
    input: "An article about llamas"
)

let batch = try await client.embed(
    model: "llama3.2",
    inputs: ["Llamas", "Alpacas", "Vicuñas"]
)

print(single.embeddings.rawValue[0].count)
print(batch.embeddings.rawValue.count)
```

## Keep-alive

Generation and chat requests accept the following `keepAlive` values:

| Value | Behavior |
| --- | --- |
| `.default` | Use the server default |
| `.none` | Unload immediately |
| `.seconds(Int)` | Keep loaded for a number of seconds |
| `.minutes(Int)` | Keep loaded for a number of minutes |
| `.hours(Int)` | Keep loaded for a number of hours |
| `.forever` | Keep loaded indefinitely |

Zero durations behave like `.none`; negative durations behave like `.forever`.

## Model management

Read operations return typed response values:

```swift
let available = try await client.listModels()
for model in available.models {
    print(model.name)
}

let running = try await client.listRunningModels()
let info = try await client.showModel("llama3.2")
let server = try await client.version()

print(running.models.count, info.capabilities, server.version)
```

Mutation methods return `true` when Ollama accepts the operation:

| Method | Operation |
| --- | --- |
| `createModel(name:modelfile:path:)` | Create a model from a Modelfile or path |
| `copyModel(source:destination:)` | Copy a local model |
| `deleteModel(_:)` | Delete a local model |
| `pullModel(_:insecure:)` | Download a model |
| `pushModel(_:insecure:)` | Upload a namespaced model |

Pushes require an Ollama account and configured public key. Set `insecure: true` only when working with a development registry you control.

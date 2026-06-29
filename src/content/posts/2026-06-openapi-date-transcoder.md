---
title: "Why OpenAPI date-time Failed Only on iOS"
description: "A short note on an OpenAPI date-time decoding bug, a wrong agent suggestion, and why engineering judgment still matters."
date: "2026-06-29"
slug: "openapi-date-transcoder"
category: "Mobile Engineering"
---

We use OpenAPI generated clients on both Android and iOS.

The API contract lives in `api/openapi.yaml`. Each mobile client generates models and endpoint calls from it. This is usually a good setup. It reduces drift. It makes many API changes visible at compile time.

But generated clients do not make every runtime behavior identical across platforms.

This bug was a good reminder.

## Symptom

One iOS list screen failed while decoding an API response.

The server returned an OpenAPI `format: date-time` field like this:

```text
2026-06-26T23:10:54.504Z
```

Android handled the same response.

iOS did not. The generated Swift client failed while decoding the response and the UI fell into a generic error state.

At first, the timestamp precision looked suspicious. The coding agent suggested changing the server response to remove fractional seconds:

```text
2026-06-26T23:10:54.504Z
-> 2026-06-26T23:10:54Z
```

That was the wrong direction.

The server was returning a valid `date-time` value. Reducing API precision to satisfy one client's default decoder would weaken the API contract. It would hide the bug, not fix it.

## Root Cause

OpenAPI `format: date-time` maps to `Foundation.Date` in the Swift generated client.

The issue was not the schema. It was the Swift OpenAPI Runtime default date transcoder.

Swift OpenAPI Runtime has a `DateTranscoder` hook. You can pass it through `OpenAPIRuntime.Configuration` when creating the generated `Client`.

This was already covered upstream. In `swift-openapi-generator` issue #537, a maintainer explained that the default `.iso8601` transcoder uses Foundation defaults, which do not handle fractional seconds. The recommended fix is to pass a different `dateTranscoder`.

So this was not a generator limitation that required changing the API. The runtime already had the right extension point.

## Why Only iOS?

Android used the same OpenAPI contract.

But the Android generated client represented date-time values with `java.time.OffsetDateTime`, with the generated client's adapters registered in the app.

The failure was specific to the Swift runtime's default date decoding behavior.

Same schema. Different generated runtimes. Different defaults.

That is the important lesson. OpenAPI reduces contract drift. It does not remove the need to understand each platform's serialization policy.

## Fix

The fix was to configure the iOS client.

Using only `.iso8601WithFractionalSeconds` is not enough. It handles values with fractional seconds, but Darwin's `ISO8601DateFormatter` with `.withFractionalSeconds` does not parse whole-second values.

```text
2026-06-26T23:10:54.504Z  // fractional parser: ok
2026-06-26T23:10:54Z      // fractional parser: fails
```

The default formatter has the opposite behavior.

So the app uses a transcoder that accepts both.

```swift
public struct AppDateTranscoder: DateTranscoder {
    private let wholeSecond: ISO8601DateTranscoder = .iso8601
    private let fractionalSecond: ISO8601DateTranscoder = .iso8601WithFractionalSeconds

    public init() {}

    public func encode(_ date: Date) throws -> String {
        try wholeSecond.encode(date)
    }

    public func decode(_ string: String) throws -> Date {
        if let date = try? fractionalSecond.decode(string) {
            return date
        }
        return try wholeSecond.decode(string)
    }
}
```

Then the generated client gets this configuration:

```swift
Client(
    serverURL: config.baseURL,
    configuration: .init(dateTranscoder: AppDateTranscoder()),
    transport: URLSessionTransport(),
    middlewares: middlewares
)
```

Encoding still uses the original whole-second format. The bug was response decoding. There was no reason to change outbound date formatting.

## What the Agent Missed

The agent's first suggestion was to normalize server timestamps to seconds.

That can look attractive. It is small. It makes the symptom disappear.

But it changes the wrong boundary.

The right questions were:

- Is the server response valid OpenAPI `date-time`?
- Why does only one platform fail?
- Does the generated runtime expose a decoder configuration?
- Can we fix the consumer without weakening the producer contract?

Once we asked those questions, the server-side change no longer made sense.

The fix belonged in the Swift client configuration.

## Conclusion

The bug was not in the API contract. It was in the iOS generated client's runtime configuration.

Coding agents are useful. They can search fast. They can patch fast. They can run tests fast.

But following an agent suggestion blindly can send a project in the wrong direction. In this case, the first suggestion would have pushed a client limitation into the API contract.

That is how systems slowly get worse.

This was a reminder that engineering judgment still matters. Before accepting a patch, we still need to ask which boundary it changes, whether it weakens a contract, and whether a smaller configuration point already exists.

The right fix was simple:

Keep the valid API response. Fix the decoder that could not read it.

References:

- [apple/swift-openapi-generator issue #537](https://github.com/apple/swift-openapi-generator/issues/537)
- [apple/swift-openapi-generator issue #84](https://github.com/apple/swift-openapi-generator/issues/84)
- [Swift OpenAPI Runtime `Configuration.swift`](https://github.com/apple/swift-openapi-runtime/blob/main/Sources/OpenAPIRuntime/Conversion/Configuration.swift)

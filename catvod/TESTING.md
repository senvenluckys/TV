# catvod JVM regression tests

Run from the repository root with JetBrains Java 21 and the Android SDK:

```sh
bash gradlew :catvod:assembleDebug :catvod:testDebugUnitTest --no-daemon --stacktrace
```

The root build also configures `app`, which reads `local.properties`. For these
catvod-only tasks, provide `sdk.dir` and non-secret placeholders for `storeFile`,
`keyAlias`, and `storePassword`; no release signing task is executed. The CI
workflow creates this file on its ephemeral runner.

`.github/workflows/catvod-tests.yml` runs independently of website publishing,
on relevant pull requests and pushes to `fongmi`, and supports manual dispatch.
It uploads JUnit XML and HTML reports even when tests fail. No emulator, release
keystore, or repository write token is required.

## Proxy redirect coverage

`OkProxySelectorTest` uses real local HTTP sockets and the production selector,
redirect interceptor, and HTTP proxy authenticator. Fake destinations are only
requested through local proxies; the explicit-direct test uses local loopback.

| Area | Assertions |
| --- | --- |
| Routing | Unmatched domain/IP inherits the source rule; matching target overrides it; explicit direct rule wins; warmed direct connections cannot bypass the proxy; clearing rules invalidates inheritance; pool policies isolate different source rules. |
| Proxy authentication | 407 after redirect uses inherited credentials; switching proxy or rule removes old credentials; target challenge uses target credentials, including rules sharing one proxy endpoint; repeated rejection terminates; missing credentials do not retry; empty passwords, embedded colons and UTF-8 credentials are encoded correctly. |
| Redirect limits | 20 redirects succeed with complete history; a 21st follow-up fails with `ProtocolException`; retry after redirect retains routing and response history; retries and redirects share the follow-up budget. |
| Request safety | Same-origin Authorization survives; cross-host/port Authorization and explicit Host are removed; cross-scheme redirects obey the client setting. |
| HTTP semantics | 300/301/302/303 POST redirects become GET without entity headers/body; 307/308 preserve non-empty POST bodies; one-shot bodies are not replayed; relative locations resolve; missing/unsupported locations terminate; no-redirect client returns the original response. |

Tests bound socket/call/JUnit timeouts and restore process-wide authentication
state after each method. Request bodies in the socket fixture are fixed-length
ASCII; it is not a general HTTP server.

## Limits

These JVM tests substitute proxy rule objects, so they do not validate Android
`Uri` configuration parsing. SOCKS routing has policy assertions, but there is no
SOCKS handshake test. HTTPS CONNECT and TLS certificate/hostname validation need
separate integration coverage. The production `OkHttp.getBuilder()` currently
uses a permissive trust manager and hostname verifier; passing this suite is not
evidence of TLS security. The tests exercise its component configuration rather
than Android-dependent singleton initialization.

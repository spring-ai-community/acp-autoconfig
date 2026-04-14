# Design: ACP Spring Boot Autoconfiguration

## Overview

This project provides Spring Boot autoconfiguration for the ACP Java SDK. It creates transport, client, and agent beans from `spring.acp.*` properties, discovers `@AcpAgent`-annotated Spring beans for automatic agent bootstrap, and manages lifecycle through Spring's context events.

## Build Coordinates

| Field | Value |
|-------|-------|
| Group ID | `com.agentclientprotocol` |
| Artifact ID | `acp-autoconfig` (parent) |
| Version | `0.1.0-SNAPSHOT` |
| Java | 21 |
| Spring Boot | 4.0.5 |
| ACP SDK | 0.10.0 |

### Modules

| Module | Purpose |
|--------|---------|
| `acp-spring-boot-autoconfigure` | Autoconfiguration classes, properties, conditional bean creation |
| `acp-spring-boot-starter` | Dependency aggregation — users add this one dependency |

### Key Dependencies

| Dependency | Scope | Purpose |
|------------|-------|---------|
| `acp-core` | compile | Client, agent, transport APIs |
| `acp-annotations` | compile | `@AcpAgent` and handler annotations |
| `acp-agent-support` | optional | Annotation-driven agent bootstrap |
| `acp-websocket-jetty` | optional | WebSocket agent transport |
| `spring-boot-autoconfigure` | compile | `@AutoConfiguration`, conditionals |
| `spring-boot-configuration-processor` | optional | IDE metadata generation |
| `acp-test` | test | `InMemoryTransportPair` |

## Architecture

### Component Diagram

```
spring.acp.client.*  ──┐
                       ▼
            ┌──────────────────────────┐
            │ AcpClientProperties      │
            └──────────┬───────────────┘
                       ▼
            ┌──────────────────────────┐
            │ AcpClientTransport-      │──── StdioAcpClientTransport
            │   AutoConfiguration      │──── WebSocketAcpClientTransport
            └──────────┬───────────────┘
                       ▼
            ┌──────────────────────────┐
            │ AcpClientAuto-           │──── AcpSyncClient bean
            │   Configuration          │──── AcpAsyncClient bean
            └──────────────────────────┘

spring.acp.agent.* ──┐
                      ▼
            ┌──────────────────────────┐
            │ AcpAgentProperties       │
            └──────────┬───────────────┘
                       ▼
            ┌──────────────────────────┐
            │ AcpAgentTransport-       │──── StdioAcpAgentTransport
            │   AutoConfiguration      │──── WebSocketAcpAgentTransport
            └──────────┬───────────────┘
                       ▼
            ┌──────────────────────────┐
            │ AcpAgentAuto-            │──── AcpAgentSupport (lifecycle-managed)
            │   Configuration          │──── discovers @AcpAgent beans
            └──────────────────────────┘
```

### Autoconfiguration Order

1. **Transport autoconfiguration** runs first (creates transport beans)
2. **Client/agent autoconfiguration** runs after (consumes transport beans)
3. **Agent lifecycle** starts on `SmartLifecycle.start()`, stops on context close

## Configuration Properties

### Client Properties (`spring.acp.client.*`)

```properties
# Transport selection
spring.acp.client.transport.type=websocket    # websocket | stdio

# WebSocket transport
spring.acp.client.transport.websocket.uri=ws://localhost:8080/acp
spring.acp.client.transport.websocket.connect-timeout=10s

# Stdio transport
spring.acp.client.transport.stdio.command=python
spring.acp.client.transport.stdio.args=-m,my_agent
spring.acp.client.transport.stdio.env.MY_VAR=value

# Client behavior
spring.acp.client.request-timeout=30s
spring.acp.client.capabilities.read-text-file=true
spring.acp.client.capabilities.write-text-file=true
spring.acp.client.capabilities.terminal=false
```

### Agent Properties (`spring.acp.agent.*`)

```properties
# Enable/disable agent autoconfiguration
spring.acp.agent.enabled=true

# Transport selection
spring.acp.agent.transport.type=stdio          # stdio | websocket

# WebSocket transport (requires acp-websocket-jetty)
spring.acp.agent.transport.websocket.port=8080
spring.acp.agent.transport.websocket.path=/acp

# Agent behavior
spring.acp.agent.request-timeout=60s
```

## Interfaces

### AcpClientProperties

```java
@ConfigurationProperties(prefix = "spring.acp.client")
public class AcpClientProperties {
    private Duration requestTimeout = Duration.ofSeconds(30);
    private TransportProperties transport = new TransportProperties();
    private CapabilitiesProperties capabilities = new CapabilitiesProperties();

    public static class TransportProperties {
        private TransportType type;  // null = auto-detect
        private WebSocketProperties websocket = new WebSocketProperties();
        private StdioProperties stdio = new StdioProperties();
    }

    public static class WebSocketProperties {
        private URI uri;
        private Duration connectTimeout = Duration.ofSeconds(10);
    }

    public static class StdioProperties {
        private String command;
        private List<String> args = new ArrayList<>();
        private Map<String, String> env = new LinkedHashMap<>();
    }

    public static class CapabilitiesProperties {
        private boolean readTextFile = true;
        private boolean writeTextFile = true;
        private boolean terminal = false;
    }
}
```

### AcpAgentProperties

```java
@ConfigurationProperties(prefix = "spring.acp.agent")
public class AcpAgentProperties {
    private boolean enabled = true;
    private Duration requestTimeout = Duration.ofSeconds(60);
    private AgentTransportProperties transport = new AgentTransportProperties();

    public static class AgentTransportProperties {
        private TransportType type;  // null = auto-detect (stdio if no jetty)
        private AgentWebSocketProperties websocket = new AgentWebSocketProperties();
    }

    public static class AgentWebSocketProperties {
        private int port = 8080;
        private String path = "/acp";
    }
}
```

### TransportType Enum

```java
public enum TransportType {
    STDIO, WEBSOCKET
}
```

## Autoconfiguration Classes

### AcpClientTransportAutoConfiguration

**Conditions**: `@ConditionalOnClass(AcpClient.class)`

**Behavior**:
- If `spring.acp.client.transport.type=websocket` or `spring.acp.client.transport.websocket.uri` is set: creates `WebSocketAcpClientTransport`
- If `spring.acp.client.transport.type=stdio` or `spring.acp.client.transport.stdio.command` is set: creates `StdioAcpClientTransport` via `AgentParameters`
- Both conditional on `@ConditionalOnMissingBean(AcpClientTransport.class)`

### AcpClientAutoConfiguration

**Conditions**: `@ConditionalOnClass(AcpClient.class)`, `@ConditionalOnBean(AcpClientTransport.class)`

**Beans**:
- `AcpAsyncClient` — built from transport + properties, `@ConditionalOnMissingBean`
- `AcpSyncClient` — wraps async client, `@ConditionalOnMissingBean`

**Lifecycle**: Implements `DisposableBean` to call `closeGracefully()` on shutdown.

**Note**: The client is NOT auto-initialized. The user calls `client.initialize(request)` explicitly because initialization requires an `InitializeRequest` with application-specific info. The autoconfiguration creates and configures the client; the user manages the protocol handshake.

### AcpAgentTransportAutoConfiguration

**Conditions**: `@ConditionalOnClass(AcpAgent.class)`, `@ConditionalOnProperty(prefix = "spring.acp.agent", name = "enabled", havingValue = "true", matchIfMissing = true)`

**Behavior**:
- If `spring.acp.agent.transport.type=stdio` (or no Jetty on classpath): creates `StdioAcpAgentTransport`
- If `spring.acp.agent.transport.type=websocket` and Jetty on classpath: creates `WebSocketAcpAgentTransport`

### AcpAgentAutoConfiguration

**Conditions**: `@ConditionalOnClass(AcpAgentSupport.class)`, `@ConditionalOnBean(AcpAgentTransport.class)`

**Behavior**:
1. Scans application context for beans annotated with `@AcpAgent`
2. For each discovered agent bean, creates an `AcpAgentSupport` instance
3. Wires transport, request timeout, and any `AcpInterceptor` beans from the context
4. Implements `SmartLifecycle`: calls `start()` on context refresh, `close()` on shutdown

**Contract**: Exactly one `@AcpAgent` bean is expected per application. Multiple agent beans is a configuration error (fail fast with clear message).

## Design Decisions

### DD-1: Client is not auto-initialized

**Context**: `AcpAsyncClient.initialize()` requires an `InitializeRequest` containing the client's name, version, and capabilities. This is application-specific.

**Decision**: Autoconfiguration creates and configures the client bean but does NOT call `initialize()`. The user calls it in their code.

**Alternatives**: (a) Auto-initialize with properties — rejected because `InitializeRequest` fields are semantic, not just config. (b) `SmartInitializingSingleton` — rejected because initialization is async and may fail.

**Rationale**: Explicit initialization is clearer. The cost is one extra line in user code; the benefit is no hidden startup failures.

### DD-2: Transport type auto-detection

**Context**: Users may set `spring.acp.client.transport.type` explicitly, or just set transport-specific properties.

**Decision**: If `type` is set, use it. If not, detect from which properties are present: `websocket.uri` present → WebSocket; `stdio.command` present → Stdio. If neither, no transport bean is created (client autoconfiguration backs off).

**Alternatives**: (a) Require explicit type — rejected for ergonomics. (b) Default to stdio — rejected because that would launch a subprocess unexpectedly.

### DD-3: Single agent per application

**Context**: The ACP agent model is one agent per process (stdio binds to one stdin/stdout pair).

**Decision**: Support exactly one `@AcpAgent` bean. Fail fast if multiple are found.

**Alternatives**: (a) Support multiple agents with different transports — adds complexity for a use case that doesn't exist yet.

### DD-4: SmartLifecycle for agent startup

**Context**: Agents need to start listening after the context is fully initialized but before the application is ready to serve.

**Decision**: Use `SmartLifecycle` with a configurable phase. This integrates with Spring's shutdown hooks for graceful termination.

**Alternatives**: (a) `@PostConstruct` — too early, context may not be ready. (b) `ApplicationReadyEvent` — correct timing but harder to integrate shutdown.

### DD-5: Interceptor beans auto-wired

**Context**: `AcpAgentSupport` accepts `AcpInterceptor` instances for cross-cutting concerns.

**Decision**: Discover all `AcpInterceptor` beans in the context and wire them into `AcpAgentSupport`, ordered by `getOrder()`.

**Rationale**: This follows Spring's pattern for `HandlerInterceptor` in MVC.

## Error Handling Strategy

- Missing required properties: Let Spring's property binding report the error naturally
- Transport creation failures: Wrap in `BeanCreationException` with a clear message
- Agent discovery failures (0 or 2+ agents): Throw `BeanCreationException` with guidance
- Runtime transport errors: Delegate to ACP SDK's exception handling (`AcpConnectionException`, etc.)

## Testing Strategy

### Unit Tests
- Properties binding: Verify `AcpClientProperties` and `AcpAgentProperties` bind correctly from various property sources
- Conditional logic: Use `ApplicationContextRunner` to test that beans are created/skipped based on conditions

### Integration Tests
- Client autoconfiguration: Wire a client with `InMemoryTransportPair`, verify it connects to a mock agent
- Agent autoconfiguration: Wire an `@AcpAgent` bean, verify it starts and handles prompts
- Override tests: Verify user-defined `@Bean` overrides auto-configured beans

### What's Mocked
- Transport layer (via `InMemoryTransportPair` from `acp-test`)
- No real processes or WebSocket servers in unit tests

## Open Questions

1. Should there be a `spring-boot-starter-acp-websocket` that pulls in `acp-websocket-jetty`?
2. Should `AcpAgentAutoConfiguration` also support programmatic agent builders (not just annotation-driven)?
3. Is there demand for reactive-only autoconfiguration (Mono/Flux beans without sync wrappers)?

## Revision History

| Date | Change |
|------|--------|
| 2026-04-14 | Initial draft |

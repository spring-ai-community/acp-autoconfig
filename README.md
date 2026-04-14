# ACP Spring Boot Autoconfiguration

Spring Boot autoconfiguration for the [ACP Java SDK](https://github.com/agent-client-protocol/acp-java). Provides auto-configured clients, agents, and transports with property-driven configuration.

## Quick Start

Add the starter dependency:

```xml
<dependency>
    <groupId>org.springaicommunity</groupId>
    <artifactId>acp-spring-boot-starter</artifactId>
    <version>0.11.0</version>
</dependency>
```

### Client

Configure the transport in `application.properties` and inject the client:

```properties
spring.acp.client.transport.stdio.command=java
spring.acp.client.transport.stdio.args=-jar,my-agent.jar
```

```java
@Component
public class MyService {

    private final AcpSyncClient client;

    public MyService(AcpSyncClient client) {
        this.client = client;
    }

    public void run() {
        client.initialize();
        var session = client.newSession(new NewSessionRequest(cwd, List.of()));
        var response = client.prompt(new PromptRequest(session.sessionId(), content));
    }
}
```

### Agent

Annotate a Spring bean with `@AcpAgent` and add handler methods:

```java
@Component
@AcpAgent(name = "my-agent", version = "1.0")
public class MyAgent {

    @Initialize
    public InitializeResponse initialize(InitializeRequest request) {
        return InitializeResponse.ok();
    }

    @NewSession
    public NewSessionResponse newSession(NewSessionRequest request) {
        return new NewSessionResponse(UUID.randomUUID().toString(), null, null);
    }

    @Prompt
    public PromptResponse prompt(PromptRequest request, SyncPromptContext context) {
        context.sendMessage("Hello!");
        return PromptResponse.endTurn();
    }
}
```

For stdio agents, redirect logging to stderr and keep the JVM alive:

```properties
spring.main.banner-mode=off
spring.main.keep-alive=true
```

## Configuration Properties

### Client

| Property | Default | Description |
|----------|---------|-------------|
| `spring.acp.client.request-timeout` | `30s` | Request timeout |
| `spring.acp.client.transport.type` | auto-detect | `stdio` or `websocket` |
| `spring.acp.client.transport.stdio.command` | — | Command to launch agent process |
| `spring.acp.client.transport.stdio.args` | — | Command arguments (comma-separated) |
| `spring.acp.client.transport.stdio.env.*` | — | Environment variables for the process |
| `spring.acp.client.transport.websocket.uri` | — | WebSocket URI (e.g. `ws://localhost:8080/acp`) |
| `spring.acp.client.transport.websocket.connect-timeout` | `10s` | WebSocket connection timeout |
| `spring.acp.client.capabilities.read-text-file` | `true` | Advertise file read capability |
| `spring.acp.client.capabilities.write-text-file` | `true` | Advertise file write capability |
| `spring.acp.client.capabilities.terminal` | `false` | Advertise terminal capability |

### Agent

| Property | Default | Description |
|----------|---------|-------------|
| `spring.acp.agent.enabled` | `true` | Enable agent autoconfiguration |
| `spring.acp.agent.request-timeout` | `60s` | Request processing timeout |
| `spring.acp.agent.transport.type` | `stdio` | Transport type |

## Transport Selection

The client transport is selected automatically based on which properties are set:

- Set `spring.acp.client.transport.stdio.command` → stdio transport
- Set `spring.acp.client.transport.websocket.uri` → WebSocket transport
- Set `spring.acp.client.transport.type` → explicit selection (takes precedence)

The agent defaults to stdio transport. Set `spring.acp.agent.enabled=false` to disable.

## Overriding Beans

All auto-configured beans back off when you provide your own. Define a custom `AcpClientTransport`, `AcpSyncClient`, `AcpAsyncClient`, or `AcpAgentTransport` bean and the autoconfiguration will use yours instead.

## Requirements

- Java 21+
- Spring Boot 4.0+
- ACP Java SDK 0.10.0+

## License

Apache License 2.0

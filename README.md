# ACP Spring Boot Autoconfiguration

Spring Boot autoconfiguration for the [ACP Java SDK](https://github.com/agent-client-protocol/acp-java). Provides auto-configured clients, agents, and transports with property-driven configuration.

## Quick Start

Add the starter dependency:

```xml
<dependency>
    <groupId>org.springaicommunity</groupId>
    <artifactId>acp-spring-boot-starter</artifactId>
    <version>0.12.0</version>
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
| `spring.acp.agent.transport.type` | `stdio` | `stdio` or `http` |
| `spring.acp.agent.transport.http.path` | `/acp` | Endpoint path |
| `spring.acp.agent.transport.http.port` | `8080` | Port of the standalone listener (not used in a servlet web app, which uses `server.port`) |
| `spring.acp.agent.transport.http.max-post-body-size` | `16MB` | Largest inbound message (POST body or WebSocket text message) |
| `spring.acp.agent.transport.http.keep-alive-interval` | `15s` | Interval between SSE keep-alive comments; `0` disables them |
| `spring.acp.agent.transport.http.mailbox-capacity` | `1024` | Events kept per outbound stream while no subscriber is attached |
| `spring.acp.agent.transport.http.max-pending-sse-events` | `1024` | Events queued for one SSE subscriber before it is closed |
| `spring.acp.agent.transport.http.max-web-socket-pending-frames` | `1024` | Frames queued for one WebSocket connection before it is closed |
| `spring.acp.agent.transport.http.max-provisional-sessions` | `64` | Session streams a connection may open before the session is known |
| `spring.acp.agent.transport.http.max-concurrent-streams-per-connection` | `1024` | HTTP/2 streams per client connection (standalone listener only) |
| `spring.acp.agent.transport.http.shutdown-timeout` | `5s` | How long closing the endpoint waits for connections to close gracefully |

## Transport Selection

The client transport is selected automatically based on which properties are set:

- Set `spring.acp.client.transport.stdio.command` → stdio transport
- Set `spring.acp.client.transport.websocket.uri` → WebSocket transport
- Set `spring.acp.client.transport.type` → explicit selection (takes precedence)

The agent defaults to stdio transport. Set `spring.acp.agent.enabled=false` to disable.

### Agent over HTTP

Add `com.agentclientprotocol:acp-streamable-http-jetty` and set `spring.acp.agent.transport.type=http`.
Each remote connection gets its own agent runtime, all served by your one `@AcpAgent` bean.

- **Servlet web application** (e.g. `spring-boot-starter-web`): the agent is mounted on the application's
  own server at `spring.acp.agent.transport.http.path` (default `/acp`), serving ACP Streamable HTTP
  (POST and SSE). WebSocket upgrades are not served in this mode.
- **Otherwise**: the SDK's own Jetty listener serves the endpoint on
  `spring.acp.agent.transport.http.port`, with Streamable HTTP, WebSocket upgrades on the same path, and
  cleartext HTTP/2.

Clients connect with `StreamableHttpAcpClientTransport` (`http://host:port/acp`) or
`WebSocketAcpClientTransport` (`ws://host:port/acp`, listener mode).

## Overriding Beans

All auto-configured beans back off when you provide your own. Define a custom `AcpClientTransport`, `AcpSyncClient`, `AcpAsyncClient`, `AcpAgentTransport`, `AcpAgentFactory`, `StreamableHttpAcpAgentTransport`, or `acpServletRegistration` bean and the autoconfiguration will use yours instead.

A transport carries exactly one session, so build one client per transport. The auto-configured
`AcpSyncClient` wraps the `AcpAsyncClient` bean (`new AcpSyncClient(asyncClient)`); a custom sync
client should do the same rather than building a second client from the transport, which ACP Java
SDK 0.18.0 rejects at construction.

## Requirements

- Java 21+
- Spring Boot 4.0+
- ACP Java SDK 0.18.0+

## License

Apache License 2.0

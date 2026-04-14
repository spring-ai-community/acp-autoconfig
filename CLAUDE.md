# ACP Spring Boot Autoconfiguration

Spring Boot autoconfiguration for the ACP Java SDK. Provides auto-configured client, agent, and transport beans from `spring.acp.*` properties.

## Build Commands

```bash
./mvnw clean compile          # Compile
./mvnw clean test             # Run tests
./mvnw clean verify           # Full build with checks
```

## Source Material Routing

| Document | Path | Read when... |
|----------|------|-------------|
| VISION.md | `plans/VISION.md` | Always read first |
| DESIGN.md | `plans/DESIGN.md` | Before implementation |
| ROADMAP.md | `plans/ROADMAP.md` | Before starting any step |

## Project Structure

```
acp-autoconfig/                             (parent POM)
├── acp-spring-boot-autoconfigure/          (autoconfigure logic)
│   └── src/main/java/com/agentclientprotocol/autoconfigure/
│       ├── client/                         (client-side beans)
│       └── agent/                          (agent-side beans)
└── acp-spring-boot-starter/                (dependency aggregation)
```

## Key Architectural Decisions

1. **Client is not auto-initialized** — autoconfiguration creates the client bean but does NOT call `initialize()`. Users call it explicitly because `InitializeRequest` contains application-specific info.
2. **Transport auto-detection** — if `type` isn't set, detect from which properties are present (websocket.uri → WebSocket, stdio.command → Stdio).
3. **Single agent per application** — exactly one `@AcpAgent` bean expected. Fail fast if zero or multiple found.
4. **SmartLifecycle for agents** — agents start after context refresh, stop on shutdown.
5. **Interceptor auto-wiring** — all `AcpInterceptor` beans discovered and wired into `AcpAgentSupport`.

## Integration Context

- **ACP Java SDK** (`~/acp/acp-java`): The upstream SDK this project autoconfigures. Read its source for API signatures, builder patterns, and transport contracts.
- **Property prefix**: `spring.acp.*` — follows Spring convention for third-party starters.
- **Spring Boot 4.x**: Uses `@AutoConfiguration`, not `spring.factories`. Registration via `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.

## Quality Standards

- Every autoconfiguration class must have `ApplicationContextRunner` tests covering:
  - Bean created when conditions met
  - Bean skipped when conditions not met
  - User-defined `@Bean` overrides auto-configured one
- Integration tests use `InMemoryTransportPair` from `acp-test` — no real processes or network

## Not Covered

- Spring MVC/WebFlux ACP endpoints
- Spring Security integration
- Actuator health indicators
- Multiple client/agent connections
- Spring Cloud service discovery

## Session Behavior

1. Read ROADMAP.md before starting any step
2. Read DESIGN.md before implementing any class
3. Check the ACP SDK source at `~/acp/acp-java` for exact API signatures
4. Write tests before or alongside implementation
5. After each step: run tests, update ROADMAP checkboxes, commit

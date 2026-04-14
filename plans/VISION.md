# Vision: ACP Spring Boot Autoconfiguration

## Problem Statement

The ACP Java SDK requires manual instantiation and wiring of clients, agents, and transports. Users must:

1. Choose and construct a transport (Stdio or WebSocket)
2. Build a client or agent with the correct builder pattern
3. Configure capabilities, timeouts, and handlers
4. Manage lifecycle (initialize, close, graceful shutdown)

This boilerplate is identical across most Spring Boot applications. Every user writes the same factory code, misses the same lifecycle hooks, and discovers the same configuration patterns through trial and error.

## Success Criteria

1. A Spring Boot application can connect to an ACP agent with zero Java configuration — only `application.properties`
2. An `@AcpAgent`-annotated Spring bean is automatically discovered, wired to a transport, and started on application startup
3. Graceful shutdown works correctly when the Spring context closes
4. Users can override any auto-configured bean with their own `@Bean` definition
5. Configuration metadata is generated for IDE autocompletion of `spring.acp.*` properties

## Scope

### In Scope

- Client autoconfiguration: `AcpSyncClient`, `AcpAsyncClient` from properties
- Transport autoconfiguration: Stdio and WebSocket for both client and agent sides
- Agent annotation bootstrap: Discover `@AcpAgent` beans, wire through `AcpAgentSupport`
- Lifecycle management: Start agents on context refresh, close on shutdown
- Configuration properties with IDE metadata generation
- Test autoconfiguration: In-memory transport for `@SpringBootTest`

### Out of Scope

- Custom Spring MVC/WebFlux endpoints for ACP (that would be a separate web integration)
- Spring Security integration for ACP authentication
- Spring Cloud integration (service discovery, config server)
- Actuator health indicators and metrics (future enhancement)
- Multiple simultaneous client connections (single client bean per app)

## Unknowns

1. Should the test autoconfiguration provide `MockAcpAgent`/`MockAcpClient` beans, or just `InMemoryTransportPair`?
2. How should annotation-driven agents interact with Spring's `@Transactional` and other proxying?
3. Should the starter pull in `acp-websocket-jetty` by default, or leave it as an optional add-on?

## Assumptions

1. Users are on Spring Boot 4.x (Java 21+)
2. The ACP SDK (0.11.x) API is stable enough for autoconfiguration to target
3. Most users want either a client OR an agent in a single application, not both
4. Stdio transport is the default for agents; WebSocket is the default for clients connecting to remote agents

## Constraints

- Must follow Spring Boot autoconfiguration conventions (`@AutoConfiguration`, `@ConditionalOnClass`, etc.)
- Must not force any transitive dependency the user doesn't need (use `optional` dependencies)
- Must support `@ConditionalOnMissingBean` so users can override any auto-configured bean
- The ACP SDK targets Java 17; this project targets Java 21 (Spring Boot 4.x requirement)

## Revision History

| Date | Change |
|------|--------|
| 2026-04-14 | Initial draft |

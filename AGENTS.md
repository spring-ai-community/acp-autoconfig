# ACP Spring Boot Autoconfiguration

Spring Boot 4 autoconfiguration and starter for the ACP Java SDK: transport, client and agent beans
from `spring.acp.*` properties.

## Steward

Planning, design state, roadmap and decisions live in the private steward repository
`/home/mark/projects/acp-autoconfig-steward`; read its `BINDING.md` before planning or executing
work. Do not copy private planning or control state into this public repository. An ignored
`plans/` tree in a checkout is transition material, not authority.

## Releases

This repository belongs to the `spring-ai-community` organization. Releases run through that
organization's `release.yml` workflow; do not plan work that assumes organization settings.

## Build and test

Use `./mvnw`, never `mvn`. Java 21, Spring Boot 4.1, ACP Java SDK 0.18.0 (`acp-sdk.version`).

```bash
./mvnw clean test      # tests
./mvnw clean verify    # the gate: run it before every commit
```

## Modules

- `pom.xml`: parent (`org.springaicommunity:acp-autoconfig-parent`); manages SDK and Boot versions.
- `acp-spring-boot-autoconfigure`: package `com.agentclientprotocol.autoconfigure`; `client/`
  (transport, client, properties) and `agent/` (transport, agent lifecycle, properties).
  Registration is `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
- `acp-spring-boot-starter`: dependency aggregation only.

## Architecture

- One client per transport. A transport instance carries exactly one session: build one
  `AcpAsyncClient` from the transport bean and derive `AcpSyncClient` with
  `new AcpSyncClient(asyncClient)`. Never build two clients from one transport bean.
- The client is not initialized. The autoconfiguration never calls `initialize()`; the application
  calls `initialize()` itself. Client capabilities come from `spring.acp.client.capabilities.*`,
  set on the client builder (the SDK takes them only there). The client is closed gracefully on
  shutdown.
- Client transport: `spring.acp.client.transport.type` (`stdio` or `websocket`) wins; otherwise
  `stdio.command` selects stdio and `websocket.uri` selects WebSocket; with neither, no transport
  and no client beans. Agent transport: stdio by default (`spring.acp.agent.enabled`);
  `spring.acp.agent.transport.type=http` serves the agent over ACP Streamable HTTP when
  `acp-streamable-http-jetty` is on the classpath — the SDK's `StreamableHttpAcpServlet` on the
  application's server in a servlet web app, otherwise the SDK's own listener
  (`StreamableHttpAcpAgentTransport`, with WebSocket upgrades on the same path). HTTP builds one agent
  runtime per remote connection from the `AcpAgentFactory` bean, all dispatching to the one
  `@AcpAgent` bean.
- At most one `@AcpAgent` bean. None: the agent lifecycle backs off (client-only apps). More than
  one: fail fast. The agent starts and stops through `SmartLifecycle`; every `AcpInterceptor` bean
  is wired into `AcpAgentSupport`.
- JSON: `acp-core` carries no JSON implementation from SDK 0.18.0. The autoconfigure module depends
  on `acp-json-jackson3`, matching Boot 4's Jackson 3.

## Hard rules

- Every autoconfiguration class has `ApplicationContextRunner` tests for the bean created, the bean
  skipped, and a user-defined bean overriding it.
- Integration tests use the SDK's `InMemoryTransportPair` (`acp-test`), never real processes or
  sockets. The one exception is the Streamable HTTP agent transport, which exists to open a socket:
  its tests bind an ephemeral port (`http.port=0`).
- Tests run with an empty `System.in` (`DetachedStdinListener`): a default stdio agent started in a
  test must never read the forked JVM's real stdin, which carries surefire's own commands.
- Check API signatures against the SDK source (`~/acp/acp-java`, at the tag `acp-sdk.version` names).
- Commit messages carry no AI attribution.

## Docs

- Autoconfig guide: https://lab.pollack.ai/docs/acp-java-sdk/autoconfig
- ACP Java SDK: https://github.com/agentclientprotocol/java-sdk
- Properties and usage: [README.md](README.md)

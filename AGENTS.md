# ACP Spring Boot Autoconfiguration Agent Instructions

This public repository owns code, tests, Maven builds, releases, shipped contracts, and public
documentation. Private planning and control state are authoritative in
`/home/mark/projects/acp-autoconfig-steward`; read its `BINDING.md` before planning or executing
work.

This repository belongs to the `spring-ai-community` organization. Releases follow that
organization's pipeline (`release.yml`); do not plan work that assumes organization settings.

Use `./mvnw`, never `mvn`. The gate is `./mvnw clean verify`.

The autoconfiguration wires the ACP Java SDK (`com.agentclientprotocol`). A transport instance
carries exactly one session: build one `AcpAsyncClient` from the transport bean and derive the
`AcpSyncClient` from it with `new AcpSyncClient(asyncClient)`; never build two clients from one
transport bean. Every autoconfiguration class has `ApplicationContextRunner` tests for the bean
created, the bean skipped, and a user-defined bean overriding it. Integration tests use the SDK's
`InMemoryTransportPair`, not real processes or sockets.

Commit messages contain no AI attribution. Do not copy private planning, current-action, roadmap,
checkpoint, or dirty-tree state into public files. An ignored `plans/` tree in a checkout is
transition material, not public authority.

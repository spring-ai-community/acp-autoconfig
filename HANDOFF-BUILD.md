# Handoff: Build Stage 1

## Mission

Implement Stage 1 of ROADMAP.md — Foundation (scaffolding, properties, enums).

## Before You Start

1. Read `CLAUDE.md` for project conventions
2. Read `plans/VISION.md` for scope and success criteria
3. Read `plans/DESIGN.md` for architecture, properties, and interfaces
4. Read `plans/ROADMAP.md` for step details and exit criteria
5. Read the ACP SDK source at `~/acp/acp-java` for exact API signatures

## First Step: 1.0 Design Review

Review VISION.md and DESIGN.md. Confirm:
- Property naming (`spring.acp.client.*`, `spring.acp.agent.*`)
- Autoconfiguration class structure (transport → client/agent ordering)
- Conditional bean logic
- Lifecycle management approach

Then proceed to Step 1.1 (scaffolding) and Step 1.2 (properties/enums).

## Key Integration Context

- ACP SDK source: `~/acp/acp-java` — read for builder patterns, transport constructors, `AcpAgentSupport` API
- Autoconfigure examples: `~/community/spring-ai-replicate/spring-ai-replicate-spring-boot-autoconfigure/` — reference for Spring Boot autoconfiguration patterns
- Spring Boot 4.0.5 — uses `@AutoConfiguration`, `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

## After Each Step

1. Run `./mvnw clean test`
2. Update ROADMAP.md checkboxes
3. Write learnings to `plans/learnings/step-N.M.md` if anything non-obvious was discovered
4. Commit with imperative subject line

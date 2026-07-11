# AGENT.md — Config Repository Guide

**GitHub:** https://github.com/spring-config-demo/config-file

## What This Repo Is

A Git-backed Spring Cloud Config repository. It holds `.properties` files that a Spring Cloud Config Server reads and serves to client microservices. This is **configuration only** — no application code lives here.

## Directory Layout

```
global/           → properties applied to every service (application*.properties)
orderService/     → properties applied only to the Order Service
```

File naming convention: `<service-name>-<profile>.properties`  
Global files use the reserved Spring name: `application-<profile>.properties`

## Key Rules

- **Never store secrets here.** Passwords, API keys, and tokens must go into a secrets manager (e.g., Vault, AWS Secrets Manager) and be referenced by placeholder.
- **One property per concern.** Keep each `.properties` file focused; do not mix unrelated services.
- **Profile consistency.** Every service that defines a `dev` file should also define `qa` and `prod` files. Missing profiles fall through to the default file.
- **Commit messages** should state *which service* and *which profile* changed, e.g. `Update order-service prod DB pool size`.

## Making Changes

### Add a property to an existing service
1. Open the relevant `<service>-<profile>.properties` file.
2. Append `key=value` on a new line.
3. Commit with a descriptive message.

### Add a new service
1. Create a folder: `<serviceName>/`
2. Add at minimum `<serviceName>.properties` (default) and one profile file.
3. Ensure the Config Server's `spring.cloud.config.server.git.search-paths` includes the new folder, or uses a wildcard (`*`).

### Add a new environment profile
1. Add `<serviceName>-<newProfile>.properties` in the service's folder.
2. Add `application-<newProfile>.properties` in `global/` if global defaults differ for that profile.

## Testing Changes Locally

```bash
# Fetch config from running Config Server (default port 8888)
curl http://localhost:8888/order-service/dev
```

Expected response contains a `propertySources` array — verify your new property appears with the correct value.

### Triggering a live refresh on a client (no restart needed)

```bash
# After pushing a change to this repo, call the client's refresh endpoint
POST http://localhost:<client-port>/actuator/refresh
```

This only reloads beans annotated with `@RefreshScope` on the client. Beans without it keep the startup value until the next restart.

## Current Services

| Folder          | Spring application name | Profiles defined     |
|-----------------|-------------------------|----------------------|
| `global/`       | `application` (wildcard)| default, dev, qa, prod |
| `orderService/` | `order-service`         | default, dev, qa, prod |

## Common Pitfalls

| Problem | Cause | Fix |
|---------|-------|-----|
| Client gets global value instead of service-specific | `spring.application.name` on the client does not match the file prefix | Align the client's app name with the properties filename prefix |
| Property not refreshed after commit | Client has not called `/actuator/refresh`, or actuator endpoint not exposed | Call `POST /actuator/refresh`; ensure `management.endpoints.web.exposure.include=refresh` |
| `/actuator/refresh` succeeds but bean still has old value | The bean is not annotated with `@RefreshScope` | Add `@RefreshScope` to the bean (demonstrated in commits `ec38660` → `9e87130`) |
| Profile fallback unexpected | Profile-specific file missing | Add the missing `<service>-<profile>.properties` |
| Config Server returns 404 for a service | Service folder not in `search-paths` config | Add the folder path (or a `*` wildcard) to `spring.cloud.config.server.git.search-paths` |

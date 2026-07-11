# AGENT.md — Config Repository Guide

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
# Start Config Server pointing at this repo (adjust port/path as needed)
curl http://localhost:8888/order-service/dev
```

Expected response contains a `propertySources` array — verify your new property appears with the correct value.

## Current Services

| Folder          | Spring application name | Profiles defined     |
|-----------------|-------------------------|----------------------|
| `global/`       | `application` (wildcard)| default, dev, qa, prod |
| `orderService/` | `order-service`         | default, dev, qa, prod |

## Common Pitfalls

| Problem | Cause | Fix |
|---------|-------|-----|
| Client gets global value instead of service-specific | Service name in client does not match file prefix | Check `spring.application.name` on the client |
| Property not refreshed after commit | Client has not called `/actuator/refresh` | Call the refresh endpoint or restart the client |
| Profile fallback unexpected | Profile-specific file missing | Add the missing `<service>-<profile>.properties` |
